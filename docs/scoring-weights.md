# Scoring Weights

Tuned scoring weights for gameplay highlight detection. Derived from user feedback on `burglingnomes.mkv` (2026-03-15).

## Formula

```
# With valid Whisper transcript (Full Tier):
score = 0.175 × Volume + 0.225 × HighFreq + 0.25 × Visual + 0.125 × DynRange + 0.15 × Breadth + 0.075 × Transcript

# Without transcript (Minimal Tier):
score = 0.20 × Volume + 0.25 × HighFreq + 0.25 × Visual + 0.15 × DynRange + 0.15 × Breadth
```

Normalized to 0–100 where the highest-scoring window = 100.

## Dimensions

| Dimension | Weight | Source | Description |
|-----------|--------|--------|-------------|
| Volume | **0.175** | ffmpeg `astats` | Per-window RMS loudness relative to session mean. (0.20 in Minimal Tier) |
| High Freq | **0.225** | ffmpeg spectral analysis | Ratio of high-frequency energy (>2kHz) to total energy. (0.25 in Minimal Tier) |
| Visual | **0.25** | ffmpeg `select='gt(scene,0.3)'` | Scene change intensity per window. Same in both tiers. |
| Dynamic Range | **0.125** | ffmpeg `astats` (per-second std) | Amplitude variation within each 5-second window. (0.15 in Minimal Tier) |
| Breadth | **0.15** | Computed | Signal diversity bonus: `active_dimensions / 5` (Full) or `/4` (Minimal). Active if normalized > 0.08. |
| Drama | **0.00** | — | Disabled: produces false positives on session starts, loading screens, AFK returns. |
| Transcript | **0.075** | transcript-analyzer agent | LLM-scored per-window transcript excitement based on style's `transcript_signals`. Only active with valid Whisper output + style defines `transcript_signals`. |

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

The five dimensions counted for breadth (Full Tier): Volume, High Freq, Visual, Dynamic Range, Transcript.

**Minimal Tier (without Transcript):**

| Active dims | Breadth value | Bonus points (×0.15) |
|:-----------:|:-------------:|:--------------------:|
| 4/4 | 1.00 | ~15 pts |
| 3/4 | 0.75 | ~11 pts |
| 2/4 | 0.50 | ~7.5 pts |
| 1/4 | 0.25 | ~3.75 pts |

The four dimensions counted for breadth (Minimal Tier): Volume, High Freq, Visual, Dynamic Range.

Activation threshold: normalized value > 0.08.

## Why These Weights

### Problem with equal/original weights

Pure volume spikes ranked too high. A game audio spike at 01:54 with +16 dB above mean scored #3 overall despite having:
- Zero high-frequency energy (no laughter or excited voices)
- Zero visual activity (nothing happening on screen)
- Only volume + dynamic range active

These are false positives — loud ≠ exciting.

### What user feedback revealed

**Good moments** (confirmed exciting by the user) shared these traits:
- At least some high-frequency energy (laughter/excitement present)
- OR at least some visual activity (something happening on screen)
- 3–4 dimensions active simultaneously
- Moderate-to-high volume (but not necessarily the loudest)

**Bad moments** (false positives flagged by the user) shared these traits:
- Extremely high volume + dynamic range
- Zero high-frequency energy AND zero visual activity
- Only 2/4 dimensions active
- Game audio spikes, loading screen transitions, or single loud sounds

### Weight derivation

1. **Drama → 0.00**: Session-start silence→burst was #1 in both original and equal-weight rankings. Not real gameplay content — just the stream starting up.

2. **HF and Visual → 0.25 each**: These are the strongest discriminators between good and bad moments:
   - Good moments: avg HF = 5.6, avg Visual = 6.8
   - Bad moments: avg HF = 0.7, avg Visual = 0.0

3. **Volume → 0.20**: Still important (good moments are loud) but not sufficient alone. Reduced from original 0.30 to prevent pure-volume false positives.

4. **Dynamic Range → 0.15**: Present in both good and bad moments at similar levels. Useful but not discriminating on its own.

5. **Breadth → 0.15**: The key innovation. A moment scoring high on only 2 dimensions gets penalized vs one scoring moderately across 3–4 dimensions. This directly targets the "just loud" false positive pattern.

## Processing Pipeline

1. Video divided into **5-second windows**
2. Each dimension computed per window via ffmpeg
3. Each dimension **normalized to 0–1** (min-max across all windows in the session)
4. Weighted sum computed per window
5. Adjacent above-mean windows **merged** (gap ≤ 10s) into segments
6. Segment score = peak window score within the segment
7. **Lead-in (2s)** and **lead-out (1s)** padding added for edit breathing room

## Transcript Analysis Integration

Whisper (`small` model) produces the `.srt` transcript. The **transcript-analyzer** agent reads it to produce the Transcript dimension.

**Model:** `small` (chosen over `base` for better accuracy on long recordings with game audio bleed).

**What the transcript-analyzer scores:** Defined by the active style's `transcript_signals` field. For the `clean` style: humor, dramatic reactions, surprised exclamations, banter, storytelling peaks, emotional outbursts, comedic timing.

**When Transcript is active**, weights shift from Minimal Tier:
- Volume: 0.20 → 0.175 (-0.025)
- High Freq: 0.25 → 0.225 (-0.025)
- Dynamic Range: 0.15 → 0.125 (-0.025)
- Transcript: 0.00 → 0.075 (+0.075)
- Visual and Breadth: unchanged

**When Transcript is inactive** (no Whisper, empty transcript, or style lacks `transcript_signals`):
Scoring uses the Minimal Tier formula. Breadth counts 4 dimensions.
