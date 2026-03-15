# V1 Mechanics Restoration — Design Spec

> **Goal:** Restore the original audio-analyzer detection mechanics (crosstalk, tension-release, exclamation bonus) that reliably found exciting moments, remove visual analysis, add sound quality preprocessing, and update the scoring formula to integrate with the transcript-analyzer.

## Problem

The current audio-analyzer replaced v1's crosstalk detection (20%), tension-release detection (20%), and exclamation bonus with Visual Motion (25%) and Dynamic Range (15%). The result: the system finds *loud* moments but not *exciting* ones. Visual scores are consistently low (0-40 range) for gameplay content, diluting the formula. Crosstalk and exclamation detection — the key discriminators for genuinely exciting group gameplay — are gone.

## Decision Summary

| Decision | Choice |
|----------|--------|
| Visual dimension | **Remove entirely** — pure audio + Transcript |
| Tension-release | **Restore raw from v1** — no safeguards for session starts/loading screens |
| Breadth bonus | **Keep** — penalizes one-dimensional "just loud" moments |
| Exclamation detection | **Keep in audio-analyzer** as simple bonus (complements LLM-based Transcript) |
| Sound quality preprocessing | **Add** — noise suppression + voice enhancement before Whisper and scoring |

## Scoring Formula

### Full Tier (with Transcript)

```
score = 0.25×Volume + 0.20×HF + 0.15×Crosstalk + 0.15×TensionRelease + 0.15×Breadth + 0.10×Transcript
+ exclamation bonus (+0.1 per exclamation, capped at +0.3)
```

Breadth counts 5 scored dimensions (excluding Breadth itself): Volume, HF, Crosstalk, Tension-release, Transcript. Active if normalized > 0.08.

Exclamation bonus applies whenever a Whisper transcript exists (full or partial), regardless of tier.

### Minimal Tier (no Transcript)

```
score = 0.30×Volume + 0.25×HF + 0.20×Crosstalk + 0.15×TensionRelease + 0.10×Breadth
+ exclamation bonus (if any transcript available)
```

Breadth counts 4 scored dimensions (excluding Breadth itself): Volume, HF, Crosstalk, Tension-release.

Normalized to 0-100, max window = 100.

## Audio-Analyzer Pipeline

### Stage 1: Track Detection (unchanged)

Probe audio streams, classify voice vs game tracks, measure average loudness.

### Stage 2: Sound Quality Preprocessing (new)

After track detection, before any analysis:

1. Extract each voice track to WAV (`-ac 1 -ar 16000`) — produces `voice.wav` (and `voice_2.wav` etc. if multi-track)
2. Detect longest quiet section using silence detection (`silencedetect=noise=-35dB:d=1`) on the primary voice track
3. Measure noise profile from the longest quiet section — this is the session-wide `noise_floor_db` (replaces the old Stage 4 noise floor measurement)
4. Apply noise suppression + voice enhancement to each voice track:
   ```
   afftdn=nf=<measured_floor> → highpass=f=80 → lowpass=f=12000 → acompressor=threshold=0.05:ratio=3:attack=5:release=50
   ```
   - `afftdn` — FFT-based noise reduction tuned to measured floor
   - `highpass=80Hz` — removes rumble/hum
   - `lowpass=12kHz` — removes hiss above speech range
   - `acompressor` — evens out quiet/loud speech
5. Output: `voice_clean.wav` (and `voice_2_clean.wav` etc.) — used by all downstream stages. Raw `voice.wav` is kept but not used further.
6. Record `noise_floor_db` for the edit-assembler
7. Whisper SRT output filename stays `voice.srt` regardless of cleaned input.

### Stage 3: Whisper Transcription (uses cleaned audio)

Same as current — `whisper` with `small` model, specified language, `.srt` output. Runs on `voice_clean.wav` instead of raw extract.

### Stage 4: Energy Scoring (uses cleaned audio)

All scoring runs on `voice_clean.wav`:

**Loudness Analysis** — per-second momentary loudness, grouped into 5-second windows. Session mean + stddev.

**High-Frequency Energy** — 2-4kHz band energy per 5-second window. Correlates with laughter and screaming.

**Silence Detection** — `silencedetect=noise=-35dB:d=3`. Feeds tension-release scoring.

**Crosstalk Detection (restored from v1):**
- Multi-track: both cleaned voice tracks (`voice_clean.wav`, `voice_2_clean.wav`) above session_mean - 6dB simultaneously per 5-second window.
- Single-track with Whisper: Whisper segments with gaps < 0.3s AND elevated volume in that window. Heuristic for overlapping speakers.

**Tension-Release Detection (restored from v1, raw — no safeguards):**
- For each silence ≥3s, analyze the 5-second window immediately after silence ends.
- Loudness delta vs session mean.
- Score = min(silence_duration / 3.0, 1.5) × loudness_delta.

**Exclamation Detection (restored from v1, requires any Whisper transcript):**
- Scan Whisper transcript for:
  - `!` punctuation
  - ALL CAPS words (3+ characters)
  - Repeated letters ("NOOO", "WHAAAAT", "hahaha")
- Each occurrence in a 5-second window: +0.1, capped at +0.3 per window.
- Applied as bonus after weighted sum.

**Composite Scoring** — Minimal Tier formula (audio-analyzer always uses Minimal). The calling command recomputes with Full Tier when transcript-analyzer provides Transcript scores.

### Merging and Padding (unchanged)

- Adjacent windows within 10s merge
- 2s lead-in, 1s lead-out
- Return ALL moments sorted by score descending

### Output

```json
{
  "tier": "full",
  "noise_floor_db": -62.3,
  "has_transcript": true,
  "transcript_srt": "<tmp_dir>/voice.srt",
  "window_dimensions": [
    { "window_start": 0.0, "window_end": 5.0, "volume": 0.32, "high_freq": 0.15, "crosstalk": 0.0, "tension_release": 0.0 }
  ],
  "moments": [
    {
      "index": 1,
      "start": "00:12:28",
      "end": "00:13:07",
      "score": 95,
      "signals": ["volume_spike", "high_freq", "crosstalk"],
      "audio_description": "Mass laughter, 3 voices overlapping",
      "transcript_excerpt": "DID HE JUST— NO WAY! [laughing]"
    }
  ]
}
```

`window_dimensions` fields: `volume`, `high_freq`, `crosstalk`, `tension_release` (visual and dyn_range removed).

## What Gets Removed

- **Visual Motion Analysis** — entire stage: scene detection (`select='gt(scene,0.3)'`), frame extraction (`fps=1`), frame differencing (`blend=all_mode=difference`), all visual-related ffmpeg commands.
- **Dynamic Range dimension** — per-second amplitude std within windows.
- **`visual` and `dyn_range`** from `window_dimensions` output.
- **`screen_description`** from moment output.
- **Drama Detection section** in audio-analyzer — renamed and restored as Tension-Release Detection (they are the same concept; "Drama" was the v2 name).
- **Noise Floor Measurement in Stage 4** — moved to Stage 2 (Sound Quality Preprocessing). Single measurement point.

## Files Changed

| File | Change |
|------|--------|
| `agents/audio-analyzer.md` | **Full rewrite of composite scoring section** with new Minimal Tier formula (`0.30×Vol + 0.25×HF + 0.20×Crosstalk + 0.15×TensionRelease + 0.10×Breadth`). Remove Visual Motion Analysis stage entirely. Remove Dynamic Range dimension. Remove Drama Detection (replaced by Tension-Release). Remove Stage 4 noise floor measurement (moved to Stage 2). Add Sound Quality Preprocessing as new Stage 2. Restore Crosstalk, Tension-release, Exclamation detection from v1. Update `window_dimensions` to: volume, high_freq, crosstalk, tension_release. Remove `screen_description` from moment output. |
| `docs/scoring-weights.md` | Replace both formulas with new weights. Remove Visual, Dynamic Range, and Drama dimensions. Add Crosstalk and Tension-release dimensions. Update Breadth tables (5-dim Full, 4-dim Minimal). Document exclamation bonus. Document Sound Quality Preprocessing. **Rewrite "Why These Weights" rationale** — remove references to Visual being a discriminator, explain why Crosstalk and Tension-release are restored. |
| `commands/gameplay-edit.md` | Update Full Tier recomputation to: `0.25×Vol + 0.20×HF + 0.15×Crosstalk + 0.15×TensionRelease + 0.15×Breadth + 0.10×Transcript + exclamation bonus`. Remove "Screen:" lines from moment presentation format. Remove `screen_description` references. Update progress reporting labels (remove "visual" references). Remove "Visual analysis: &lt;time&gt;" from timing summary. Remove visual-failure error handling case. |
| `agents/transcript-analyzer.md` | No changes to agent behavior. |
| `styles/clean.md` | No changes. |

## Supersession Note

This spec **supersedes the scoring formulas** in the transcript-analyzer design spec (`2026-03-15-transcript-analyzer-design.md`). That spec's Full Tier formula (`0.175×Vol + 0.225×HF + 0.25×Visual + 0.125×DynRange + 0.15×Breadth + 0.075×Transcript`) is replaced by this spec's formula. The transcript-analyzer agent's behavior (window scoring, moment discovery, summaries, cut boundary refinement) is unchanged — only the formula that gameplay-edit uses to recompute composite scores changes.

## Relationship to Transcript-Analyzer

The transcript-analyzer (built in the previous spec) is unaffected in behavior. It still:
- Runs after the audio-analyzer
- Receives moments + `.srt`
- Scores per-window Transcript excitement based on style's `transcript_signals`
- Writes clip summaries
- Refines cut boundaries to sentence edges

The gameplay-edit command recomputes composite scores using the Full Tier 6-dimension formula after the transcript-analyzer returns:
```
score = 0.25×Volume + 0.20×HF + 0.15×Crosstalk + 0.15×TensionRelease + 0.15×Breadth + 0.10×Transcript
+ exclamation bonus
```
