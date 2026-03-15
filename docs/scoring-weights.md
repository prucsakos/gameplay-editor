# Scoring Weights

Tuned scoring weights for gameplay highlight detection. Derived from user feedback on `burglingnomes.mkv` (2026-03-15).

## Formula

```
# With valid Whisper transcript (Full Tier):
score = 0.25 × Volume + 0.20 × HF + 0.15 × Crosstalk + 0.15 × TensionRelease + 0.15 × Breadth + 0.10 × Transcript
+ exclamation bonus (+0.1 per exclamation, capped at +0.3)

# Without transcript (Minimal Tier):
score = 0.30 × Volume + 0.25 × HF + 0.20 × Crosstalk + 0.15 × TensionRelease + 0.10 × Breadth
+ exclamation bonus (if any transcript available)
```

Normalized to 0–100 where the highest-scoring window = 100.

## Dimensions

| Dimension | Weight (Full) | Weight (Minimal) | Source | Description |
|-----------|:-------------:|:----------------:|--------|-------------|
| Volume | **0.25** | **0.30** | ffmpeg `ebur128` | Per-window RMS loudness relative to session mean. Primary excitement signal. |
| High Freq | **0.20** | **0.25** | ffmpeg spectral analysis | Ratio of high-frequency energy (2–4kHz). Correlates with laughter and screaming. |
| Crosstalk | **0.15** | **0.20** | ffmpeg multi-track / Whisper gaps | Simultaneous speakers detected per window. Multi-track: both tracks above session_mean - 6dB. Single-track: Whisper segments with gaps < 0.3s + elevated volume. |
| Tension-Release | **0.15** | **0.15** | ffmpeg `silencedetect` | Silence ≥3s followed by loudness spike. Score = min(silence_duration / 3.0, 1.5) × loudness_delta. Raw — no safeguards for session starts. |
| Breadth | **0.15** | **0.10** | Computed | Signal diversity bonus: `active_dimensions / 5` (Full) or `/4` (Minimal). Active if normalized > 0.08. Breadth itself is excluded from the count. |
| Transcript | **0.10** | — | transcript-analyzer agent | LLM-scored per-window transcript excitement based on style's `transcript_signals`. Only active with valid Whisper output + style defines `transcript_signals`. |

## Breadth Bonus

The breadth bonus rewards moments where multiple signals fire simultaneously and penalizes one-dimensional moments.

**Full Tier (with Transcript):**

Five dimensions counted: Volume, High Freq, Crosstalk, Tension-Release, Transcript. Breadth itself is excluded from the count. Weight is ×0.15.

| Active dims | Breadth value | Bonus points (×0.15) |
|:-----------:|:-------------:|:--------------------:|
| 5/5 | 1.00 | ~15 pts |
| 4/5 | 0.80 | ~12 pts |
| 3/5 | 0.60 | ~9 pts |
| 2/5 | 0.40 | ~6 pts |
| 1/5 | 0.20 | ~3 pts |

**Minimal Tier (without Transcript):**

Four dimensions counted: Volume, High Freq, Crosstalk, Tension-Release. Weight is ×0.10.

| Active dims | Breadth value | Bonus points (×0.10) |
|:-----------:|:-------------:|:--------------------:|
| 4/4 | 1.00 | ~10 pts |
| 3/4 | 0.75 | ~7.5 pts |
| 2/4 | 0.50 | ~5 pts |
| 1/4 | 0.25 | ~2.5 pts |

Activation threshold: normalized value > 0.08.

## Exclamation Bonus

Applied whenever a Whisper transcript exists (full or partial), regardless of tier. Scanned from the `.srt` output:

- `!` punctuation
- ALL CAPS words (3+ characters)
- Repeated letters ("NOOO", "WHAAAAT", "hahaha")

Each occurrence in a 5-second window adds +0.1 to the window's score, capped at +0.3 per window. Applied after the weighted sum, before normalization to 0–100.

## Why These Weights

### Problem with v2 weights (Visual + Dynamic Range)

The v2 formula included Visual (scene change detection) and Dynamic Range (amplitude variation). Visual scored 0–40 even for static scenes due to video compression artifacts, which diluted the formula with noise. Dynamic Range didn't discriminate between exciting and boring moments — both had similar amplitude variation. Together they absorbed ~37.5% of the Full Tier weight while contributing minimal signal quality.

### What worked in v1

The v1 formula used: Volume 35%, HF 25%, Crosstalk 20%, Tension-Release 20%. This combination reliably found:
- Group laughter (HF spike + Crosstalk)
- Dramatic reveals (Tension-Release pattern)
- Exclamations and reactive moments (Volume + HF)

Removing Visual and DynRange and restoring these four audio-only dimensions recovers that signal quality.

### Current weight rationale

1. **Volume 0.25 / 0.30**: Primary excitement signal. Louder = more active. Slightly higher in Minimal Tier to compensate for the absence of Transcript.
2. **High Freq 0.20 / 0.25**: Laughter and screaming concentrate energy in the 2–4kHz band. Strong discriminator between game audio spikes (low HF) and genuine reactions (high HF).
3. **Crosstalk 0.15 / 0.20**: Group gameplay is more exciting when multiple people are talking. Detects overlapping speech on multi-track recordings and rapid back-and-forth on single-track.
4. **Tension-Release 0.15 / 0.15**: Silence followed by a burst is a universal comedy/drama pattern. Same weight in both tiers because it's transcript-independent.
5. **Breadth 0.15 / 0.10**: Higher in Full Tier because five dimensions are available — simultaneous firing across five signals is a stronger quality signal. Lower in Minimal Tier (four dimensions) to avoid over-rewarding incomplete data.
6. **Transcript 0.10**: Tie-breaker and style-aware signal. LLM can catch moments that audio alone misses (e.g., a quietly delivered punchline). Capped at 0.10 to prevent transcript quality variance from dominating the score.
7. **Exclamation bonus +0.1 / cap +0.3**: Applied post-sum, pre-normalization. Rewards the most obvious excitement markers without restructuring the weighted formula.

## Processing Pipeline

1. Voice track(s) extracted and preprocessed
2. Whisper transcription on cleaned audio (small model)
3. Audio divided into 5-second windows
4. Each dimension computed per window via ffmpeg
5. Exclamation bonus computed from transcript
6. Weighted sum (Minimal Tier in audio-analyzer)
7. Adjacent above-mean windows merged (gap ≤ 10s)
8. Segment score = peak window score
9. Lead-in (2s) and lead-out (1s) padding
10. Transcript-analyzer runs (if Full Tier)
11. Composite scores recomputed with Full Tier formula

## Sound Quality Preprocessing

1. Extract WAV from voice track(s)
2. Detect longest quiet section (used as noise profile source)
3. Measure noise profile from that section
4. Apply ffmpeg filter chain: `afftdn` (noise reduction) + `highpass=f=80` (remove rumble below 80Hz) + `lowpass=f=12000` (remove hiss above 12kHz) + `acompressor` (normalize dynamics)
5. Output `voice_clean.wav` for Whisper and ffmpeg dimension analysis

## Transcript Analysis Integration

Whisper (`small` model) produces the `.srt` transcript. The **transcript-analyzer** agent reads it to produce the Transcript dimension.

**Model:** `small` (chosen over `base` for better accuracy on long recordings with game audio bleed).

**What the transcript-analyzer scores:** Defined by the active style's `transcript_signals` field. For the `clean` style: humor, dramatic reactions, surprised exclamations, banter, storytelling peaks, emotional outbursts, comedic timing.

**When Transcript is active**, weights shift from Minimal Tier to Full Tier:
- Volume: 0.30 → 0.25 (-0.05)
- High Freq: 0.25 → 0.20 (-0.05)
- Crosstalk: 0.20 → 0.15 (-0.05)
- Breadth: 0.10 → 0.15 (+0.05)
- Transcript: 0.00 → 0.10 (+0.10)
- Tension-Release: unchanged at 0.15

**When Transcript is inactive** (no Whisper output, empty transcript, or style lacks `transcript_signals`):
Scoring uses the Minimal Tier formula. Breadth counts 4 dimensions.
