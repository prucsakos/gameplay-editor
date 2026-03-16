# Scoring Weights

Tuned scoring weights for gameplay highlight detection. Derived from user feedback on `burglingnomes.mkv` (2026-03-15), refined to audio-only analysis (2026-03-16).

## Formula

```
# With valid transcript (Full Tier):
score = 0.20 x Volume + 0.25 x HighFreq + 0.15 x DynRange + 0.15 x Breadth + 0.25 x Transcript

# Without transcript (Minimal Tier):
score = 0.25 x Volume + 0.35 x HighFreq + 0.20 x DynRange + 0.20 x Breadth
```

Normalized to 0-100 where the highest-scoring window = 100.

## Dimensions

| Dimension | Full Weight | Minimal Weight | Source | Description |
|-----------|:-----------:|:--------------:|--------|-------------|
| Volume | **0.20** | **0.25** | ffmpeg `ebur128` | Per-window RMS loudness relative to session mean |
| High Freq | **0.25** | **0.35** | Spectral analysis (2-4kHz) | Laughter/screaming detection |
| Dynamic Range | **0.15** | **0.20** | Per-second amplitude std | Amplitude variation within each window |
| Breadth | **0.15** | **0.20** | Computed | Signal diversity bonus (see below) |
| Transcript | **0.25** | -- | transcript-analyzer agent | LLM-scored per-window excitement based on style's `transcript_signals` |
| Drama | **0.00** | **0.00** | -- | Disabled: produces false positives on session starts, loading screens, AFK returns |

## Breadth Bonus

The breadth bonus rewards moments where multiple signals fire simultaneously and penalizes one-dimensional moments.

**Full Tier (with Transcript):**

| Active dims | Breadth value | Bonus points (x0.15) |
|:-----------:|:-------------:|:--------------------:|
| 4/4 | 1.00 | ~15 pts |
| 3/4 | 0.75 | ~11 pts |
| 2/4 | 0.50 | ~7.5 pts |
| 1/4 | 0.25 | ~3.75 pts |

The four dimensions counted for breadth (Full Tier): Volume, High Freq, Dynamic Range, Transcript.

**Minimal Tier (without Transcript):**

| Active dims | Breadth value | Bonus points (x0.20) |
|:-----------:|:-------------:|:--------------------:|
| 3/3 | 1.00 | ~20 pts |
| 2/3 | 0.67 | ~13 pts |
| 1/3 | 0.33 | ~6.5 pts |

The three dimensions counted for breadth (Minimal Tier): Volume, High Freq, Dynamic Range.

Activation threshold: normalized value > 0.08.

## Why These Weights

### Audio-only analysis

Visual analysis was removed from the pipeline:
- Frame-by-frame analysis was the slowest stage (~3-5 min for a 3h video)
- Scene detection produced noisy signals (menu transitions, loading screens scored high)
- The transcript-analyzer provides better moment discrimination for gameplay content

Without visual, the remaining audio dimensions combined with transcript analysis are stronger discriminators of exciting moments.

### What user feedback revealed

**Good moments** (confirmed exciting by the user) shared these traits:
- High-frequency energy present (laughter/excitement)
- 3+ dimensions active simultaneously
- Moderate-to-high volume (but not necessarily the loudest)
- Interesting dialogue content (humor, reactions, banter)

**Bad moments** (false positives flagged by the user) shared these traits:
- Extremely high volume + dynamic range only
- Zero high-frequency energy
- Only 1-2 dimensions active
- Game audio spikes or loading screen transitions

### Weight derivation

1. **Drama -> 0.00**: Session-start silence->burst was ranked too high. Not real gameplay content.

2. **HighFreq -> 0.25 / 0.35**: The strongest audio discriminator. Good moments avg HF = 5.6, bad moments avg HF = 0.7. Gets higher weight in Minimal Tier where transcript isn't available to compensate.

3. **Volume -> 0.20 / 0.25**: Important but not sufficient alone. Reduced to prevent pure-volume false positives.

4. **Dynamic Range -> 0.15 / 0.20**: Present in both good and bad moments at similar levels. Useful but not discriminating on its own.

5. **Breadth -> 0.15 / 0.20**: Penalizes one-dimensional moments. A "just loud" moment with 1/3 or 1/4 active dimensions gets heavily penalized.

6. **Transcript -> 0.25**: Equal to HighFreq in Full Tier. The transcript-analyzer captures humor, banter, dramatic reactions, and other content signals that audio energy alone cannot detect. This is the primary quality differentiator when available.

## Transcription

**Library:** faster-whisper (CTranslate2 backend)
**Model:** `small` (always). Chosen over `base` for better accuracy on long recordings with game audio bleed. Not `medium` or `large` to keep processing time reasonable.
**VAD:** Enabled (`vad_filter=True`). Skips non-speech sections for faster processing.

## Processing Pipeline

1. Video divided into **5-second windows**
2. Each audio dimension computed per window via ffmpeg
3. Each dimension **normalized to 0-1** (min-max across all windows in the session)
4. Weighted sum computed per window
5. Adjacent above-mean windows **merged** (gap <= 10s) into segments
6. Segment score = peak window score within the segment
7. **Lead-in (2s)** and **lead-out (1s)** padding added for edit breathing room

## Transcript Analysis Integration

faster-whisper (`small` model) produces the `.srt` transcript. The **transcript-analyzer** agent reads it to produce the Transcript dimension.

**What the transcript-analyzer scores:** Defined by the active style's `transcript_signals` field. For the `clean` style: humor, dramatic reactions, surprised exclamations, banter, storytelling peaks, emotional outbursts, comedic timing.

**When Transcript is active**, weights shift from Minimal Tier:
- Volume: 0.25 -> 0.20 (-0.05)
- High Freq: 0.35 -> 0.25 (-0.10)
- Dynamic Range: 0.20 -> 0.15 (-0.05)
- Breadth: 0.20 -> 0.15 (-0.05)
- Transcript: 0.00 -> 0.25 (+0.25)

**When Transcript is inactive** (no faster-whisper, empty transcript, or style lacks `transcript_signals`):
Scoring uses the Minimal Tier formula. Breadth counts 3 dimensions.
