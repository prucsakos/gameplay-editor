# V1 Mechanics Restoration — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore v1 audio detection mechanics (crosstalk, tension-release, exclamation bonus), remove visual analysis, add sound quality preprocessing, and update the scoring formula across all files.

**Architecture:** This is a markdown-only change set across 3 agent/command definition files and 1 documentation file. The audio-analyzer gets the biggest rewrite (remove visual, add preprocessing, restore v1 signals). The scoring-weights doc and gameplay-edit command update to match. No code files, no tests — these are LLM agent prompt definitions.

**Tech Stack:** Markdown, YAML frontmatter, ffmpeg CLI commands (documented in prompts)

**Spec:** `docs/superpowers/specs/2026-03-15-v1-mechanics-restoration-design.md`

---

## Chunk 1: Scoring Weights and Audio-Analyzer

### Task 1: Update scoring-weights.md

**Files:**
- Modify: `docs/scoring-weights.md`

- [ ] **Step 1: Replace the formula section**

Replace the current formulas (lines 7-13) with:

```markdown
## Formula

\```
# With valid Whisper transcript (Full Tier):
score = 0.25 × Volume + 0.20 × HF + 0.15 × Crosstalk + 0.15 × TensionRelease + 0.15 × Breadth + 0.10 × Transcript
+ exclamation bonus (+0.1 per exclamation, capped at +0.3)

# Without transcript (Minimal Tier):
score = 0.30 × Volume + 0.25 × HF + 0.20 × Crosstalk + 0.15 × TensionRelease + 0.10 × Breadth
+ exclamation bonus (if any transcript available)
\```
```

- [ ] **Step 2: Replace the dimensions table**

Replace the current dimensions table (lines 18-27) with:

```markdown
## Dimensions

| Dimension | Weight (Full) | Weight (Minimal) | Source | Description |
|-----------|---------------|-------------------|--------|-------------|
| Volume | **0.25** | **0.30** | ffmpeg `ebur128` | Per-window RMS loudness relative to session mean. Primary excitement signal. |
| High Freq | **0.20** | **0.25** | ffmpeg spectral analysis | Ratio of high-frequency energy (2-4kHz). Correlates with laughter and screaming. |
| Crosstalk | **0.15** | **0.20** | ffmpeg multi-track / Whisper gaps | Simultaneous speakers detected per window. Multi-track: both tracks above session_mean - 6dB. Single-track: Whisper segments with gaps < 0.3s + elevated volume. |
| Tension-Release | **0.15** | **0.15** | ffmpeg `silencedetect` | Silence ≥3s followed by loudness spike. Score = min(silence_duration / 3.0, 1.5) × loudness_delta. Raw — no safeguards for session starts. |
| Breadth | **0.15** | **0.10** | Computed | Signal diversity bonus: `active_dimensions / 5` (Full) or `/4` (Minimal). Active if normalized > 0.08. Breadth itself is excluded from the count. |
| Transcript | **0.10** | — | transcript-analyzer agent | LLM-scored per-window transcript excitement based on style's `transcript_signals`. Only active with valid Whisper output + style defines `transcript_signals`. |
```

- [ ] **Step 3: Update the Breadth Bonus section**

Replace the Breadth Bonus section (lines 29-56) with:

```markdown
## Breadth Bonus

The breadth bonus rewards moments where multiple signals fire simultaneously and penalizes one-dimensional moments.

**Full Tier (with Transcript):**

| Active dims | Breadth value | Bonus points (×0.15) |
|:-----------:|:-------------:|:--------------------:|
| 5/5 | 1.00 | ~15 pts |
| 4/5 | 0.80 | ~12 pts |
| 3/5 | 0.60 | ~9 pts |
| 2/5 | 0.40 | ~6 pts |
| 1/5 | 0.20 | ~3 pts |

The five dimensions counted for breadth (Full Tier): Volume, High Freq, Crosstalk, Tension-Release, Transcript. Breadth itself is excluded.

**Minimal Tier (without Transcript):**

| Active dims | Breadth value | Bonus points (×0.10) |
|:-----------:|:-------------:|:--------------------:|
| 4/4 | 1.00 | ~10 pts |
| 3/4 | 0.75 | ~7.5 pts |
| 2/4 | 0.50 | ~5 pts |
| 1/4 | 0.25 | ~2.5 pts |

The four dimensions counted for breadth (Minimal Tier): Volume, High Freq, Crosstalk, Tension-Release.

Activation threshold: normalized value > 0.08.
```

- [ ] **Step 4: Add exclamation bonus section**

Add after the Breadth Bonus section:

```markdown
## Exclamation Bonus

Applied whenever a Whisper transcript exists (full or partial), regardless of tier. Scanned from the `.srt` output:

- `!` punctuation
- ALL CAPS words (3+ characters)
- Repeated letters ("NOOO", "WHAAAAT", "hahaha")

Each occurrence in a 5-second window adds +0.1 to the window's score, capped at +0.3 per window. Applied after the weighted sum, before normalization to 0-100.
```

- [ ] **Step 5: Rewrite the "Why These Weights" section**

Replace the entire "Why These Weights" section (lines 58-95) with:

```markdown
## Why These Weights

### Problem with v2 weights (Visual + Dynamic Range)

Visual Motion Analysis scored consistently low (0-40 range) for gameplay content where the camera is relatively static. This diluted the formula — 25% of the score came from a near-zero signal. Dynamic Range was present in both good and bad moments at similar levels. Meanwhile, the v1 mechanics that actually discriminated exciting moments (crosstalk, tension-release, exclamation bonus) were removed.

### What worked in v1

The original v1 formula (Volume 35%, HF 25%, Crosstalk 20%, Tension-Release 20%) reliably found:
- **Group laughter and reactions** — high HF + crosstalk firing together
- **Dramatic reveals** — silence → explosion via tension-release
- **Excited exclamations** — exclamation bonus from transcript punctuation and caps

### Current weight rationale

1. **Volume → 0.25 (Full) / 0.30 (Minimal)**: Primary excitement signal. Reduced from v1's 0.35 to make room for Breadth and Transcript, but still the largest single weight.

2. **HF → 0.20 / 0.25**: Laughter and screaming detector. Strong discriminator between exciting and merely loud moments.

3. **Crosstalk → 0.15 / 0.20**: Restored from v1. Multiple people talking/shouting simultaneously is a reliable excitement signal for group gameplay.

4. **Tension-Release → 0.15 / 0.15**: Restored from v1 (raw, no safeguards). Captures dramatic silence→explosion patterns. May false-positive on session starts but this is accepted.

5. **Breadth → 0.15 / 0.10**: Penalizes one-dimensional "just loud" moments. A moment scoring high on only 1-2 dimensions gets penalized vs one scoring moderately across 3-4 dimensions.

6. **Transcript → 0.10**: LLM-scored excitement from what's actually being said. Complements the audio signals with semantic understanding.

7. **Exclamation bonus (+0.1, cap +0.3)**: Simple pattern-match bonus from v1. Fast, cheap, and effective at boosting genuinely excited moments where people type/say "!!!" or "NOOO".
```

- [ ] **Step 6: Update the Processing Pipeline and Transcript Analysis sections**

Replace the Processing Pipeline section (lines 97-106) with:

```markdown
## Processing Pipeline

1. Video's voice track(s) extracted and **preprocessed** (noise suppression, highpass, lowpass, compression)
2. Whisper transcription on cleaned audio (`small` model)
3. Audio divided into **5-second windows**
4. Each dimension computed per window via ffmpeg (on cleaned audio)
5. Exclamation bonus computed from Whisper transcript
6. Weighted sum computed per window (Minimal Tier in audio-analyzer)
7. Adjacent above-mean windows **merged** (gap ≤ 10s) into segments
8. Segment score = peak window score within the segment
9. **Lead-in (2s)** and **lead-out (1s)** padding added
10. Transcript-analyzer runs (if Full Tier): adds Transcript dimension, summaries, cut refinement
11. Composite scores **recomputed** with Full Tier 6-dimension formula by gameplay-edit command
```

Replace the Transcript Analysis Integration section (lines 107-123) with:

```markdown
## Sound Quality Preprocessing

Before Whisper and scoring, the voice track is cleaned:

1. Extract voice track(s) to WAV
2. Detect longest quiet section (`silencedetect`)
3. Measure noise profile → `noise_floor_db`
4. Apply: `arnndn` (RNNoise noise reduction) → `highpass=80Hz` → `lowpass=12kHz` → `acompressor`
5. Output: `voice_clean.wav` for all downstream stages

## Transcript Analysis Integration

Whisper (`small` model) produces the `.srt` transcript from cleaned audio. The **transcript-analyzer** agent reads it to produce the Transcript dimension.

**What the transcript-analyzer scores:** Defined by the active style's `transcript_signals` field. For the `clean` style: humor, dramatic reactions, surprised exclamations, banter, storytelling peaks, emotional outbursts, comedic timing.

**When Transcript is active**, weights shift from Minimal Tier:
- Volume: 0.30 → 0.25 (-0.05)
- High Freq: 0.25 → 0.20 (-0.05)
- Crosstalk: 0.20 → 0.15 (-0.05)
- Breadth: 0.10 → 0.15 (+0.05)
- Transcript: 0.00 → 0.10 (+0.10)
- Tension-Release: unchanged at 0.15

**When Transcript is inactive** (no Whisper, empty transcript, or style lacks `transcript_signals`):
Scoring uses the Minimal Tier formula. Breadth counts 4 dimensions.
```

- [ ] **Step 7: Verify and commit**

Read through the complete file to verify consistency. Check that:
- Full Tier weights sum to 1.00 (0.25+0.20+0.15+0.15+0.15+0.10 = 1.00)
- Minimal Tier weights sum to 1.00 (0.30+0.25+0.20+0.15+0.10 = 1.00)
- No remaining references to Visual, Dynamic Range, or Drama

```bash
git add docs/scoring-weights.md
git commit -m "docs(scoring-weights): restore v1 mechanics — crosstalk, tension-release, exclamation bonus"
```

---

### Task 2: Rewrite audio-analyzer.md — Remove Visual, Add Preprocessing

**Files:**
- Modify: `agents/audio-analyzer.md`

This is the largest change. The audio-analyzer needs: visual analysis removed, sound quality preprocessing added, crosstalk/tension-release/exclamation restored, composite scoring formula updated, output format updated.

- [ ] **Step 1: Update the agent description header**

Replace the description line (line 10) and input section. The agent description should say:

```markdown
You analyze gameplay video audio to find the most exciting moments — laughter, screaming, crosstalk, exclamations, and tension-release patterns.
```

- [ ] **Step 2: Remove Visual Motion Analysis (Stage 3)**

Delete the entire "Stage 3: Visual Motion Analysis" section (lines 96-130) — everything from `## Stage 3: Visual Motion Analysis` through the `Report timing` line. This removes scene detection, frame extraction, and frame differencing.

- [ ] **Step 3: Add Sound Quality Preprocessing as new Stage 2**

Insert after Stage 1 (Track Detection), before the current Stage 2 (Whisper Transcription). The current Stage 2 becomes Stage 3.

```markdown
## Stage 2: Sound Quality Preprocessing (new)

After track detection, preprocess all voice tracks to improve analysis quality.

### Extract Voice Tracks

For each voice track:
```bash
ffmpeg -i "<video_path>" -map 0:a:<voice_index> -ac 1 -ar 16000 -y "<tmp_dir>/voice.wav"
```
If multiple voice tracks: `voice.wav`, `voice_2.wav`, etc.

### Measure Noise Floor

Detect all silent sections:
```bash
ffmpeg -i "<tmp_dir>/voice.wav" -af "silencedetect=noise=-35dB:d=1" -f null - 2>&1
```

Parse all `silence_start` / `silence_end` pairs. Find the **longest** quiet section. Measure its RMS level:
```bash
ffmpeg -i "<tmp_dir>/voice.wav" -ss <longest_silence_start> -t <longest_silence_duration> \
  -af "astats=metadata=1:reset=1,ametadata=print:key=lavfi.astats.Overall.RMS_level:file=<tmp_dir>/noise_floor.txt" -f null - 2>&1
```

Parse the average RMS level — this is the session-wide `noise_floor_db`.

### Apply Noise Suppression and Voice Enhancement

For each voice track:
```bash
ffmpeg -i "<tmp_dir>/voice.wav" \
  -af "arnndn=m=<rnnoise_model>,highpass=f=80,lowpass=f=12000,acompressor=threshold=0.05:ratio=3:attack=5:release=50" \
  -y "<tmp_dir>/voice_clean.wav"
```

- `arnndn=m=<rnnoise_model>` — RNNoise neural network noise reduction (model at `$HOME/.cache/gameplay-editor/sh.rnnn`)
- `highpass=f=80` — removes rumble/hum below 80Hz
- `lowpass=f=12000` — removes hiss above speech range
- `acompressor` — evens out quiet/loud speech

Output: `voice_clean.wav` (and `voice_2_clean.wav` etc.). Raw `voice.wav` is kept but not used further. Whisper SRT output filename stays `voice.srt`.

Report: `"Noise floor: <noise_floor_db> dB, preprocessing complete"`
Report timing: `"preprocessing_ms": <PREPROCESSING_MS>`
```

- [ ] **Step 4: Update Whisper Transcription stage**

This becomes Stage 3. Rename the heading from `## Stage 2: Whisper Transcription (Full Tier)` to `## Stage 3: Whisper Transcription (uses cleaned audio)`. Update the Whisper command to use `voice_clean.wav`:

```bash
PYTHONIOENCODING=utf-8 whisper "<tmp_dir>/voice_clean.wav" --model small --language <language> --output_format srt --output_dir "<tmp_dir>/"
```

The SRT output will be named `voice_clean.srt`. Rename it to `voice.srt` for consistency:
```bash
mv "<tmp_dir>/voice_clean.srt" "<tmp_dir>/voice.srt"
```

- [ ] **Step 5: Rewrite Energy Scoring stage**

This stays as Stage 4 (numbering unchanged since Visual Stage 3 was deleted and Preprocessing was inserted as new Stage 2). Rename the heading from `## Stage 4: Energy Scoring` to `## Stage 4: Energy Scoring (uses cleaned audio)`. Remove Dynamic Range. Remove the old noise floor measurement (moved to Stage 2). Restore Crosstalk, Tension-Release, and Exclamation Detection. All scoring uses `voice_clean.wav`.

The Loudness Analysis and High-Frequency Energy subsections stay the same but reference `voice_clean.wav`.

The Silence Detection subsection stays but remove the "Session Noise Floor Measurement" subsection (moved to Stage 2).

Replace the "Drama Detection" subsection with:

```markdown
### Tension-Release Detection

For each silence period ≥3s (from Silence Detection), analyze the 5-second window immediately after silence ends:
- Measure loudness delta vs session mean
- Score = min(silence_duration / 3.0, 1.5) × loudness_delta
- Normalize to 0-1 range across all windows (min-max)
```

Rename the Crosstalk Detection heading from `### Crosstalk Detection (Multi-track only)` to `### Crosstalk Detection` (remove the "(Multi-track only)" qualifier — the body already describes both multi-track and single-track-with-Whisper cases).

Add after Crosstalk Detection:

```markdown
### Exclamation Detection (requires Whisper transcript)

If a Whisper transcript (`.srt`) was produced (full or partial), scan it for exclamation markers:
- `!` punctuation
- ALL CAPS words (3+ characters)
- Repeated letters ("NOOO", "WHAAAAT", "hahaha")

For each 5-second window, count occurrences that fall within that window's timestamp range. Each occurrence adds +0.1 to the window's final score, capped at +0.3 per window. Applied after the weighted sum.
```

- [ ] **Step 6: Update Composite Scoring subsection**

Replace the composite scoring formula and table with:

```markdown
### Composite Scoring

Score each 5-second window using the **Minimal Tier** formula:

    score = 0.30 × Volume + 0.25 × HighFreq + 0.20 × Crosstalk + 0.15 × TensionRelease + 0.10 × Breadth
    + exclamation bonus

| Dimension | Weight | Source |
|-----------|--------|--------|
| Volume | 0.30 | Per-window RMS loudness relative to session mean (from Loudness Analysis) |
| High Freq | 0.25 | Ratio of high-frequency energy >2kHz (from High-Frequency Energy) |
| Crosstalk | 0.20 | Simultaneous speakers per window (from Crosstalk Detection) |
| Tension-Release | 0.15 | Silence→loudness spike score (from Tension-Release Detection) |
| Breadth | 0.10 | Signal diversity bonus: `active_dimensions / 4`. A dimension is "active" if its normalized value > 0.08. Counts: Volume, High Freq, Crosstalk, Tension-Release. |

Each dimension is normalized to 0-1 range (min-max across all windows in the session). The weighted sum is the raw score per window. Then add exclamation bonus (+0.1 per occurrence, capped +0.3).

**Note:** This is always the Minimal Tier formula. When a valid transcript exists, the calling command (gameplay-edit) recomputes composite scores using the Full Tier 6-dimension formula with the transcript-analyzer's Transcript scores.
```

- [ ] **Step 7: Update Output section**

Replace the output JSON example and field descriptions. Remove `screen_description` from moments. Update `window_dimensions` to contain: `volume`, `high_freq`, `crosstalk`, `tension_release`.

```json
{
  "tier": "full",
  "language": "hu",
  "source_duration": 12255.0,
  "session_mean": 42.3,
  "session_stddev": 18.7,
  "noise_floor_db": -62.3,
  "has_transcript": true,
  "transcript_srt": "<tmp_dir>/voice.srt",
  "timing": {
    "audio_probe_ms": 4200,
    "preprocessing_ms": 12000,
    "whisper_ms": 758000,
    "scoring_ms": 800
  },
  "window_dimensions": [
    { "window_start": 0.0, "window_end": 5.0, "volume": 0.32, "high_freq": 0.15, "crosstalk": 0.0, "tension_release": 0.0 },
    { "window_start": 5.0, "window_end": 10.0, "volume": 0.41, "high_freq": 0.67, "crosstalk": 0.85, "tension_release": 0.0 }
  ],
  "moments": [
    {
      "index": 1,
      "start": "00:12:28",
      "end": "00:13:07",
      "duration": 39.0,
      "score": 95,
      "signals": ["volume_spike", "high_freq", "crosstalk"],
      "audio_description": "Mass laughter, 3 voices overlapping, volume spike +12dB above mean",
      "transcript_excerpt": "DID HE JUST— NO WAY! [laughing]"
    }
  ]
}
```

Update the field descriptions:
- Remove `screen_description` from the moment format
- Remove `visual_analysis_ms` from timing
- Add `preprocessing_ms` to timing
- `window_dimensions` fields: `volume`, `high_freq`, `crosstalk`, `tension_release`
- Note: `visual` and `dyn_range` removed

Update the "Important" note at the bottom to remove the `screen_description` requirement. Keep `audio_description` and `transcript_excerpt`.

- [ ] **Step 8: Verify and commit**

Read through the complete file. Check:
- No remaining references to Visual Motion Analysis, scene detection, frame extraction, frame differencing
- No remaining references to Dynamic Range
- Drama Detection replaced by Tension-Release Detection
- Noise floor measurement only in Stage 2 (not duplicated in Stage 4)
- All ffmpeg commands reference `voice_clean.wav` (not `voice.wav`) for analysis
- Stages are numbered 1-4 (not 1-5)

```bash
git add agents/audio-analyzer.md
git commit -m "feat(audio-analyzer): restore v1 mechanics, add preprocessing, remove visual analysis"
```

---

## Chunk 2: gameplay-edit Command Updates

### Task 3: Update gameplay-edit.md

**Files:**
- Modify: `commands/gameplay-edit.md`

- [ ] **Step 1: Update the Full Tier recomputation formula**

Replace line 104:
```
    score = 0.175 × Volume + 0.225 × HighFreq + 0.25 × Visual + 0.125 × DynRange + 0.15 × Breadth + 0.075 × Transcript
```

With:
```
    score = 0.25 × Volume + 0.20 × HF + 0.15 × Crosstalk + 0.15 × TensionRelease + 0.15 × Breadth + 0.10 × Transcript
    + exclamation bonus (+0.1 per exclamation, capped at +0.3)
```

- [ ] **Step 2: Update the Breadth computation**

Replace line 109:
```
3. Compute Breadth: count how many of the 5 dimensions (Volume, HighFreq, Visual, DynRange, Transcript) have normalized value > 0.08. Breadth = `active_count / 5`
```

With:
```
3. Compute Breadth: count how many of the 5 scored dimensions (Volume, HF, Crosstalk, TensionRelease, Transcript) have normalized value > 0.08. Breadth = `active_count / 5`. Breadth itself is excluded from the count.
```

- [ ] **Step 3: Update the dimension references in recomputation**

Replace line 107-108:
```
1. Read the 4 audio dimensions from `window_dimensions` (audio-analyzer output)
2. Read the Transcript score from `transcript_scores` (transcript-analyzer output)
```

With:
```
1. Read the 4 audio dimensions from `window_dimensions`: `volume`, `high_freq`, `crosstalk`, `tension_release` (audio-analyzer output)
2. Read the Transcript score from `transcript_scores` (transcript-analyzer output)
3. Read the exclamation bonus from the audio-analyzer's per-window exclamation counts (applied after weighted sum, +0.1 each, capped +0.3)
```

- [ ] **Step 4: Remove "Screen:" from the plan presentation format**

In the "Present Plan" section (lines 132-171), remove all "Screen:" lines from the example output. Change the three-line format to two lines:

```
  ★ #1  [Score: 95] 00:12:28 → 00:13:07 (39s)
        Audio: Mass laughter, 3 voices overlapping, volume spike +12dB
        Summary: "DID HE JUST— NO WAY! [laughing]"
```

Update line 173-176 — change the description requirement from three lines to two:

```markdown
Each moment MUST include two description lines:
- **Audio**: What's happening in the audio (volume spikes, laughter, crosstalk, tension-release, etc.)
- **Summary**: Clip summary from transcript-analyzer (if available), or Whisper transcript excerpt as fallback
```

- [ ] **Step 5: Remove "Visual analysis" from the Pipeline Summary timing**

Delete line 233:
```
  Visual analysis:     <visual_analysis time>
```

Add after "Audio probe" line:
```
  Preprocessing:       <preprocessing time>
```

- [ ] **Step 6: Remove visual-failure error handling**

Delete line 265:
```
- Visual analysis fails → skip visual signals, reweight remaining, warn
```

- [ ] **Step 7: Update progress reporting labels**

In all four progress reporting variants (lines 272-302), replace references to "visual":

- `"Analyzing audio and visual signals..."` → `"Scoring audio signals..."`
- `"Analyzing visual motion and scoring moments..."` → `"Scoring audio signals..."`

- [ ] **Step 8: Update audio-analyzer output description**

Replace line 78:
```
The agent returns a JSON result with all detected moments (preliminary 5-dim scores), per-window dimension arrays (`window_dimensions`), timing data, noise floor, and `has_transcript` flag.
```

With:
```
The agent returns a JSON result with all detected moments (preliminary Minimal Tier scores), per-window dimension arrays (`window_dimensions` with volume, high_freq, crosstalk, tension_release), per-window exclamation bonus counts, timing data, noise floor, and `has_transcript` flag.
```

- [ ] **Step 9: Verify and commit**

Read through the complete file. Check:
- No remaining references to Visual, visual, screen, DynRange, Dynamic Range
- Formula matches spec exactly
- All moment presentation examples show 2 lines (Audio + Summary), not 3
- Progress labels say "audio signals" not "visual"

```bash
git add commands/gameplay-edit.md
git commit -m "feat(gameplay-edit): update to v1 formula, remove visual references"
```

---

### Task 4: Update transcript-analyzer-design.md supersession note

**Files:**
- Modify: `docs/superpowers/specs/2026-03-15-transcript-analyzer-design.md`

- [ ] **Step 1: Add supersession note at the top**

After the title line, add:

```markdown
> **Note:** The scoring formulas in this spec have been **superseded** by `docs/superpowers/specs/2026-03-15-v1-mechanics-restoration-design.md`. The transcript-analyzer agent behavior (window scoring, moment discovery, summaries, cut boundary refinement) is unchanged. Only the composite scoring formula used by gameplay-edit has changed.
```

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/specs/2026-03-15-transcript-analyzer-design.md
git commit -m "docs: add supersession note to transcript-analyzer spec"
```

