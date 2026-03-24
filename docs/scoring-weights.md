# Scoring Weights (v4)

Scoring weights for gameplay highlight detection. v3 weights derived from user feedback on `burglingnomes.mkv` (2026-03-15). v4 rebalances toward transcript (voice is the content).

## Formula

```
# With valid transcript (Full Tier):
score = 0.15 x Volume + 0.20 x HighFreq + 0.10 x DynRange + 0.15 x Breadth + 0.40 x Transcript

# Without transcript (Minimal Tier):
score = 0.25 x Volume + 0.35 x HighFreq + 0.20 x DynRange + 0.20 x Breadth
```

Normalized to 0-100 where the highest-scoring window = 100.

## Dimensions

| Dimension | Full Weight | Minimal Weight | Source | Description |
|-----------|:-----------:|:--------------:|--------|-------------|
| Volume | **0.15** | **0.25** | ffmpeg `ebur128` | Per-window RMS loudness relative to session mean |
| High Freq | **0.20** | **0.35** | Spectral analysis (2-4kHz) | Laughter/screaming detection |
| Dynamic Range | **0.10** | **0.20** | Per-second amplitude std | Amplitude variation within each window |
| Breadth | **0.15** | **0.20** | Computed | Signal diversity bonus (see below) |
| Transcript | **0.40** | -- | LLM scoring of transcript | Per-window excitement based on style's `transcript_signals` |

## v4 Weight Shift Rationale

Transcript weight increased from 0.25 (v3) to 0.40 (v4) because the user's content is voice/conversation-driven ("it is us that matter usually and our voice"). Audio dimensions are supporting signals, not primary.

**This is a hypothesis.** Should be validated against real recordings and tuned if needed. The v3 weights were derived from empirical testing; v4 weights are based on stated user priorities.

| Dimension | v3 Full | v4 Full | Change |
|-----------|:-------:|:-------:|:------:|
| Volume | 0.20 | 0.15 | -0.05 |
| High Freq | 0.25 | 0.20 | -0.05 |
| Dynamic Range | 0.15 | 0.10 | -0.05 |
| Breadth | 0.15 | 0.15 | 0 |
| Transcript | 0.25 | 0.40 | +0.15 |

## Breadth Bonus

Rewards moments where multiple signals fire simultaneously.

**Full Tier (with Transcript):**

Breadth = `active_of(Volume, HighFreq, DynRange, Transcript) / 4`. Breadth itself excluded from its own count.

| Active dims | Breadth value | Bonus points (x0.15) |
|:-----------:|:-------------:|:--------------------:|
| 4/4 | 1.00 | ~15 pts |
| 3/4 | 0.75 | ~11 pts |
| 2/4 | 0.50 | ~7.5 pts |
| 1/4 | 0.25 | ~3.75 pts |

**Minimal Tier (without Transcript):**

Breadth = `active_of(Volume, HighFreq, DynRange) / 3`.

| Active dims | Breadth value | Bonus points (x0.20) |
|:-----------:|:-------------:|:--------------------:|
| 3/3 | 1.00 | ~20 pts |
| 2/3 | 0.67 | ~13 pts |
| 1/3 | 0.33 | ~6.5 pts |

Activation threshold: normalized value > 0.08.

## Confidence Tiers

- **Strong:** score > 70, at least 2 dimensions above 0.08
- **Maybe:** score 30-70, or only 1 active dimension

Over-detection threshold: 30 (configurable in style as `detection_threshold`).

## Transcription

**Library:** faster-whisper (CTranslate2 backend)
**Model:** `small` (always)
**VAD:** Enabled (`vad_filter=True`)
**Chunking:** 2-minute chunks with 15-second overlap at boundaries

## Processing Pipeline

1. Audio denoised and normalized (Phase 1 — audio-preparer)
2. Analysis mix divided into **5-second windows**
3. Each audio dimension computed per window via ffmpeg
4. Transcript scored per window via LLM (Phase 2 — moment-detector)
5. Each dimension **normalized to 0-1**
6. Weighted sum computed per window
7. Composite scores normalized to **0-100** (max window = 100)
8. Adjacent above-threshold windows **merged** (gap <= 10s)
9. Moment score = peak window score
10. **Lead-in (2s)** and **lead-out (1s)** padding
11. Context zones: **10s** each side (for dashboard "hear more")
