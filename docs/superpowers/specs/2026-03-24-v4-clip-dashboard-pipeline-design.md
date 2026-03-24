# V4 Clip Dashboard Pipeline — Design Spec

**Date:** 2026-03-24
**Version:** 4.0.0
**Status:** Approved

## Problem Statement

The current gameplay-editor (v3) has fundamental workflow issues:

1. **Static noise breaks analysis** — denoising happens at export, but scoring runs on noisy audio, producing unreliable results
2. **Mid-quality clip selection** — threshold-based filtering at score 70 produces too many false positives and misses quiet but memorable moments
3. **Sluggish human-in-the-loop** — reviewing a text list of scores in the terminal is insufficient; you can't *feel* a clip from a description
4. **Export is coupled to assembly** — rendering happens before the user has full control, forcing restarts on bad results
5. **No shorts generation** — TikTok/Reels output requires manual platform flags and separate runs
6. **No voice normalization** — multi-speaker recordings have unbalanced volume levels

The user's gold standard is: auto-editor → kdenlive → manual curation → export. The goal is to replace this entire chain with a single skill invocation that produces a highlight reel + shorts with minimal interaction.

## Core Principle

**Voice is the content.** The recordings are about friends talking — their laughter, stories, reactions, and banter. Game audio is context, not signal. The entire pipeline is designed around this reality.

## Architecture Overview

Four distinct phases with a hard boundary between analysis and rendering:

```
Phase 1: PREPARE        Phase 2: DETECT         Phase 3: CURATE           Phase 4: EXPORT
─────────────────       ───────────────         ─────────────────         ─────────────────
Raw video               Clean audio              HTML dashboard            Highlight reel
  → Track detection       → Energy scoring         → Audio previews          → Shorts
  → Noise profiling       → Transcription          → Keep/Remove/Comment     → Voice-normalized
  → Denoise audio         → Over-detection         → User curates            → Platform-formatted
  → Speaker detection     → Moment ranking         → Agent reads feedback
  → Voice normalization   → Extended context
```

**No rendering happens until Phase 4.** Phases 1-3 are cheap (analysis + lightweight audio extraction for previews). Phase 4 runs once on the final approved clip set.

### Agent Structure

Four agents aligned to phases (replacing the current 3):

- **audio-preparer** — Phase 1: noise profiling, denoising, track detection, speaker normalization
- **moment-detector** — Phase 2: energy scoring, transcription, moment discovery, over-detection
- **dashboard-builder** — Phase 3: generates HTML with extracted audio previews
- **export-assembler** — Phase 4: final rendering of highlight reel + shorts

### Runtime Dependencies

- **ffmpeg/ffprobe** — all audio/video processing (required)
- **Python 3 + faster-whisper** — transcription (optional, degrades gracefully)

No new dependencies from v3.

---

## Phase 1: PREPARE

**Goal:** Take raw video → produce clean, normalized audio ready for analysis.

### Step 1: Track Detection & Classification

- ffprobe identifies all audio streams
- Auto-classify each track as voice or game audio:
  - Heuristic: mono + lower loudness = voice, stereo + higher loudness = game
  - Multiple mono tracks = multiple voice tracks (friends on separate channels)
  - Single mixed track = combined, no separation attempted

### Step 2: Noise Profiling

- For each voice track: find the longest silence segment (ffmpeg `silencedetect`, -35dB, 3s minimum)
- Measure noise floor RMS from that silence using `astats` (metadata: `lavfi.astats.Overall.RMS_level`)
- This RMS value is used to: (a) set `afftdn` noise reduction level, (b) set gate threshold in Phase 4
- Fallback: if no silence segment ≥3s found, use the first 30 seconds and take the minimum RMS value, then warn user

### Step 3: Denoising

- Apply `afftdn` with `nr` (noise reduction) level calibrated from measured RMS: `nr=clamp(abs(noise_floor_rms) - 20, 10, 40)`
  - Noisier recordings get stronger reduction; clean recordings get light touch
  - `nt=w` (Wiener method) for best quality
- Output: clean voice track(s) as temp WAV files — these are the **analysis copies** used in Phase 2
- **Critical fix:** all downstream analysis runs on denoised audio, so static noise no longer poisons scoring
- The **original raw tracks** are preserved for Phase 4 export (denoising is applied fresh there with the same parameters)

### Step 4: Speaker Detection & Voice Normalization

- Measure each voice track's integrated loudness (`ebur128`)
- Record per-track LUFS offset from -16 LUFS target (used in Phase 4 for final normalization)
- For analysis: normalize all voice tracks to -16 LUFS so scoring isn't biased by volume differences
- Single mixed track: normalize whole track, skip per-speaker normalization
- **Note:** Phase 1 normalization is for analysis only. Phase 4 applies final platform-target normalization to raw tracks

### Step 5: Prepare Analysis Audio

- Mix normalized voice tracks + game audio (attenuated -6dB relative to voices) into single analysis mix
- Voices are front and center; game audio provides context but doesn't dominate scoring

### Phase 1 Output

- Clean, normalized analysis mix (WAV) — for Phase 2 energy scoring
- Individual clean voice tracks (WAV) — for Phase 2 transcription
- Original raw track paths — preserved for Phase 4 mastering (avoids double-denoising)
- Track map (which track is which speaker/game)
- Noise floor measurements per track (RMS values + calibrated `afftdn nr` parameter)
- Per-speaker LUFS offsets (for Phase 4 normalization)
- Source video metadata (resolution, fps, duration)

---

## Phase 2: DETECT

**Goal:** Find every potentially interesting moment. Cast a wide net — false positives are cheap to remove in the dashboard, false negatives are gone forever.

### Strategy

Over-detect intentionally. Low threshold (score 30 vs current 70). The dashboard lets users quickly filter out noise. This directly addresses both false positives (easy to remove) and false negatives (wider net catches more).

### Step 1: Energy Scoring

On clean analysis mix, 5-second windows across entire recording:

- **Volume** — ebur128 loudness relative to session mean
- **HighFreq** — 2-4kHz energy (laughter, screaming, excitement)
- **DynRange** — amplitude variance within window (sudden reactions)
- Normalization: 0-1 per dimension relative to session max

### Step 2: Transcription

- faster-whisper with VAD filter on each voice track separately
- Merge transcripts with speaker labels ("Speaker 1", "Speaker 2", or track names)
- Output: timestamped, speaker-attributed transcript

### Step 3: Transcript Scoring (LLM Pass)

LLM reads transcript in 2-minute chunks with 15-second overlap at boundaries. Each chunk scores its constituent 5-second windows 0-1 for:

- Laughter / humor
- Emotional reactions (surprise, excitement, anger)
- Extended conversation (long engaged talks — "memories")
- Storytelling moments
- Banter between friends

Overlap windows take the higher score from either chunk. For a 2-hour recording (~60 chunks), this stays well within context limits per call.

**Failure handling:** If LLM transcript scoring fails (timeout, malformed output), retry once. If still failing, fall back to Minimal Tier weights (audio-only) for affected chunks and warn user. The pipeline continues — degraded scoring is better than no output.

This is where the LLM understands *meaning*, not just volume.

### Step 4: Moment Assembly

Combine audio + transcript scores per window.

**Full Tier formula:**
```
score = 0.15 × Volume + 0.20 × HighFreq + 0.10 × DynRange + 0.15 × Breadth + 0.40 × Transcript
```

Transcript dominates at 0.40 because the content is voice/conversation-driven. Audio dimensions are supporting signals. This weight shift is a hypothesis based on the user's stated priority ("it is us that matter usually and our voice"). It should be validated against real recordings and tuned if needed.

**Breadth calculation:** Breadth = count of active dimensions / total scored dimensions. A dimension is "active" if its normalized value > 0.08. For Full Tier: `active_of(Volume, HighFreq, DynRange, Transcript) / 4` (Breadth itself is excluded from its own count). For Minimal Tier: `active_of(Volume, HighFreq, DynRange) / 3`.

**Minimal Tier formula (no whisper):**
```
score = 0.25 × Volume + 0.35 × HighFreq + 0.20 × DynRange + 0.20 × Breadth
```

### Step 5: Over-Detection with Moment Merging

- Include everything scoring above 30
- Adjacent high-scoring windows merge into single moments (2s lead-in, 1s lead-out padding — asymmetric to capture buildup)
- Overlapping moments merge
- Context zones (10s each side) are clamped to video boundaries and truncated at neighboring moments' core boundaries (e.g., if moment A ends at 120s and moment B starts at 125s, A's context zone extends to 124s, not 130s)
- Each moment gets:
  - Start/end timestamps
  - Extended context zone: 10s before, 10s after (for "hear more" in dashboard)
  - Score + per-dimension breakdown
  - Transcript excerpt
  - 1-2 sentence LLM description ("Speaker 1 tells a story about X, Speaker 2 starts laughing")
  - Confidence tier: **strong** (score >70 on 0-100 scale, at least 2 active dimensions) / **maybe** (score 30-70, or single active dimension)
  - Note: composite scores are normalized to 0-100 scale (max window = 100) after weighted sum, matching v3 convention

### Phase 2 Output

- Full moment list with scores, transcripts, descriptions, confidence tiers
- Extended context boundaries
- Path to clean voice tracks (for audio preview extraction)
- Session statistics

---

## Phase 3: CURATE (The Dashboard)

**Goal:** A single HTML file opened in the browser. User hears each clip, decides in seconds, closes the tab when done. Agent reads decisions and proceeds to export.

### Dashboard Generation

- Extract audio preview per moment as .mp3 from clean voice tracks (the detected range)
- Extract extended audio preview (+10s each side) as separate .mp3 for "hear more"
- Generate self-contained HTML file with inline CSS/JS
- Audio files stored in temp folder, referenced by relative paths
- Open with `start` command (Windows)

### Dashboard Layout

Per clip card:
- Clip number, confidence tier (STRONG / MAYBE), score, timestamps, duration
- LLM-generated description of what's happening
- Audio player — plays the detected moment
- "Hear More" buttons — extend playback 10s before/after using context zone audio
- Transcript excerpt with speaker labels
- Keep / Remove toggle (one click)
  - Strong clips (>70): default to Keep
  - Maybe clips (30-70): default to Keep but visually dimmed
- Boundary adjustment buttons: `-5s start`, `+5s start`, `-5s end`, `+5s end`
- Comment field — optional free text for instructions to the agent

Global controls:
- Moment count and tier breakdown at top
- "Approve All" button
- Running summary at bottom: X kept, Y removed, Z commented
- "Save & Close" button — writes `decisions.json` and shows "Return to terminal" message

### Comment Examples

- "make this longer, the funny part continues"
- "this is a visual moment, keep despite quiet audio"
- "use this for a short"
- "trim the first 3 seconds, it's just coughing"

### State Persistence

All decisions stored in `decisions.json` next to the HTML. Schema:

```json
{
  "version": 1,
  "source_video": "path/to/recording.mkv",
  "moments": [
    {
      "id": 1,
      "action": "keep",
      "start": 252.0,
      "end": 278.0,
      "original_start": 252.0,
      "original_end": 278.0,
      "comment": ""
    },
    {
      "id": 2,
      "action": "remove",
      "start": 723.0,
      "end": 739.0,
      "original_start": 723.0,
      "original_end": 739.0,
      "comment": ""
    },
    {
      "id": 5,
      "action": "keep",
      "start": 1400.0,
      "end": 1430.0,
      "original_start": 1405.0,
      "original_end": 1425.0,
      "comment": "use this for a short"
    }
  ]
}
```

- `action`: `"keep"` or `"remove"`
- `start`/`end`: adjusted timestamps (after user boundary changes)
- `original_start`/`original_end`: as detected (for reference)
- `comment`: free text, empty string if none

### Polling Mechanism

- Agent checks for `decisions.json` every 5 seconds using Python: `os.path.exists()` (cross-platform, consistent with Windows compatibility approach)
- Timeout after 30 minutes with a message: "Dashboard timed out. Run `/gameplay-edit --resume` to reopen."
- If user closes browser without saving, the JSON won't exist and the timeout catches it
- Stale detection: the JSON includes a `session_id` field matching one generated at pipeline start; mismatched IDs are ignored

---

## Phase 4: EXPORT

**Goal:** One pass, all outputs. Highlight reel + shorts + properly mastered audio.

### Step 1: Process Dashboard Feedback

- Read `decisions.json` — kept clips, adjusted boundaries, comments
- LLM interprets comment instructions:
  - "make this longer" → extend boundaries using context zone
  - "this is a visual moment" → keep as-is, skip audio gate for this clip
  - "trim the first 3 seconds" → adjust start timestamp
  - "use this for a short" → tag for shorts generation
- Sort kept clips chronologically

### Step 2: Audio Mastering (single pass per track, from raw sources)

**Important:** Phase 4 works from the **original raw tracks**, not the Phase 1 denoised copies. This avoids double-denoising artifacts. Phase 1's noise measurements are reused to parameterize the filters.

Full filter chain on each voice track:
1. `highpass=f=80` — rumble removal
2. `afftdn=nr=<Phase1_calibrated_value>:nt=w` — denoise (same parameters as Phase 1, applied once to raw audio)
3. `agate` — noise gate (threshold: Phase 1 noise floor + 6dB, range=0.06, attack=5ms, release=150ms, hold=25ms)
4. `acompressor` — even out dynamics (ratio 3:1, attack 10ms, release 100ms, knee 2.83)
5. `loudnorm=I=<platform_target>` — final normalization (applied once, to platform target)

Game audio: normalize separately, attenuate -6dB relative to voice.
Mix all tracks to final stereo.

Platform targets:
- YouTube: -16 LUFS
- TikTok: -12 LUFS

### Step 3: Highlight Reel Assembly

- Extract each kept clip as segment (video + mastered audio)
- Apply transitions between segments (per style config: fade or cut)
- Concatenate into single `.mp4`
- Output: `<basename>_highlight.mp4`

### Step 4: Shorts Generation

- Auto-select top `shorts_count` (default 3) clips by score for shorts
- User-tagged clips ("use this for a short") are always included and do NOT count toward the `shorts_count` cap
- Minimum score for auto-selection: 60 (only strong-ish clips become shorts automatically)
- If fewer clips qualify than `shorts_count`, produce only what qualifies — no padding
- Per short:
  - Crop to 9:16 (center crop from source)
  - Enforce 15-60s duration (trim to peak moment if longer)
  - Louder master at -12 LUFS
  - Hard cuts, no fades
- Output: `<basename>_short_01.mp4`, `<basename>_short_02.mp4`, etc.

### Step 5: Quick-Fix Loop (optional, one round)

- Agent reports what was produced: file paths, sizes, durations
- User can watch results and give corrections in terminal:
  - "remove the clip at 1:45 in the highlight reel"
  - "short #2 audio is too quiet"
  - "add 5 more seconds to the ending"
- Agent re-exports with corrections
- One round only — keeps it fast

### Output Structure

```
Outs/
  <basename>_highlight.mp4       # Full highlight reel (YouTube-ready)
  <basename>_short_01.mp4        # Short #1 (TikTok/Reels-ready)
  <basename>_short_02.mp4        # Short #2
  <basename>_short_03.mp4        # Short #3
```

---

## Graceful Degradation

| Feature | Full Tier (ffmpeg + whisper) | Minimal Tier (ffmpeg only) |
|---------|---------------------------|--------------------------|
| Noise removal | ✓ | ✓ |
| Voice normalization | ✓ | ✓ |
| Energy scoring | ✓ | ✓ |
| Transcription | ✓ Speaker-labeled | ✗ |
| Transcript scoring | ✓ 0.40 weight | ✗ Falls back to audio-only weights |
| Dashboard transcripts | ✓ Full text | ✗ Audio descriptions only |
| Dashboard audio preview | ✓ | ✓ |
| Shorts generation | ✓ | ✓ |

---

## Scoring Formulas

### Full Tier
```
score = 0.15 × Volume + 0.20 × HighFreq + 0.10 × DynRange + 0.15 × Breadth + 0.40 × Transcript
```

### Minimal Tier
```
score = 0.25 × Volume + 0.35 × HighFreq + 0.20 × DynRange + 0.20 × Breadth
```

### Dimensions

| Dimension | Source | Full | Minimal |
|-----------|--------|:----:|:-------:|
| Volume | ffmpeg ebur128 per-window | 0.15 | 0.25 |
| HighFreq | 2-4kHz band energy | 0.20 | 0.35 |
| DynRange | Per-second amplitude std | 0.10 | 0.20 |
| Breadth | Active dimensions count / total | 0.15 | 0.20 |
| Transcript | LLM analysis of transcript | 0.40 | — |

### Confidence Tiers

- **Strong:** score > 70, at least 2 dimensions above 0.08 activation threshold
- **Maybe:** score 30-70, or single signal only

---

## Performance Estimates

For a 2-hour recording:

| Phase | Time | Bottleneck |
|-------|------|-----------|
| Phase 1: Prepare | ~1-2 min | Denoising |
| Phase 2: Detect | ~3-5 min | Transcription |
| Phase 3: Curate | User's pace | — |
| Phase 4: Export | ~2-5 min | Segment encoding |
| **Total automated** | **~6-12 min** | |

---

## Windows Compatibility

- PYTHONUTF8=1 on all Python invocations
- ASCII-only output from Python scripts
- Python `tempfile.mkdtemp()` for native Windows path resolution
- Dashboard opened with `start` command
- Audio previews as .mp3 with relative paths (no server needed)
- Path escaping via `filter_complex_script` for ffmpeg

---

## Style Configuration

Extends current `styles/clean.md` with new fields:

```yaml
name: clean
transitions: fade
transition_duration: 0.3
audio_normalization: standard
aspect_ratio: per-platform
export_codec: libx264
export_quality: 20
transcript_signals:
  - humor
  - dramatic reactions
  - surprised exclamations
  - banter
  - storytelling peaks
  - emotional outbursts
  - comedic timing
# New in v4:
shorts_count: 3                    # Max number of auto-generated shorts
shorts_duration_range: [15, 60]    # Min/max seconds per short
detection_threshold: 30            # Low threshold for over-detection
```

---

## Migration from v3

- Replaces all 3 current agents with 4 new agents
- `gameplay-edit.md` command rewritten for new 4-phase flow
- `gameplay-setup.md` unchanged (same dependencies)
- Scoring weights doc updated
- Pipeline flow doc updated
- Style config extended (backwards compatible — new fields have defaults)

---

## Comparison: v3 vs v4

| Aspect | v3 (Current) | v4 (New) |
|--------|-------------|----------|
| Noise handling | At export only — poisons analysis | Phase 1 denoise — clean analysis |
| Detection strategy | Threshold at 70, miss quiet moments | Over-detect at 30, curate in dashboard |
| Transcript weight | 0.25 (supporting) | 0.40 (primary — voice is the content) |
| Review UX | Text list in terminal | HTML dashboard with audio previews |
| Boundary adjustment | Type IDs and numbers | Click buttons, leave comments |
| Export timing | Per-segment during assembly | Single pass at the very end |
| Shorts | Manual platform flag | Auto-generated from top clips |
| Voice normalization | None | Per-speaker LUFS normalization |
| Post-export fix | Start over | One round of quick corrections |
