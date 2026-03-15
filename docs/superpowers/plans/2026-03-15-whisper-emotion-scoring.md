> **SUPERSEDED** — This plan has been replaced by `2026-03-15-transcript-analyzer.md`. The inline Emotion dimension approach was replaced with a dedicated transcript-analyzer agent. Only Task 1 (Whisper base→small) is preserved in the new plan.

---

# Whisper Small Model + Emotion-Based Excitement Scoring

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Switch Whisper to the `small` model and integrate transcript-based Emotion analysis into the excitement scoring formula alongside the tuned weights from `scoring-weights.md`.

**Architecture:** The audio-analyzer agent gains a post-Whisper hallucination filter and a new Emotion dimension computed from transcript punctuation/caps. The Full Tier scoring formula is replaced with the 6-dimension weighted formula from `scoring-weights.md` (Volume, HighFreq, Visual, DynRange, Breadth, Emotion). The Minimal Tier (no Whisper) keeps the existing 5-dimension formula from `scoring-weights.md` without Emotion.

**Tech Stack:** Whisper (small model), ffmpeg, markdown agent definitions

---

## Chunk 1: Whisper Model Upgrade + Hallucination Filtering

### Task 1: Switch Whisper model from `base` to `small`

**Files:**
- Modify: `gameplay-editor/agents/audio-analyzer.md:60` (the whisper command line)

- [ ] **Step 1: Edit the Whisper command in Stage 2**

In `agents/audio-analyzer.md`, line 60, change:
```bash
PYTHONIOENCODING=utf-8 whisper "<tmp_dir>/voice.wav" --model base --language <language> --output_format srt --output_dir "<tmp_dir>/"
```
to:
```bash
PYTHONIOENCODING=utf-8 whisper "<tmp_dir>/voice.wav" --model small --language <language> --output_format srt --output_dir "<tmp_dir>/"
```

- [ ] **Step 2: Commit**

```bash
git add gameplay-editor/agents/audio-analyzer.md
git commit -m "feat(audio-analyzer): switch Whisper from base to small model

The small model produces more accurate transcripts on long (>1h) gameplay
recordings with game audio bleed, reducing hallucinations."
```

### Task 2: Add Whisper hallucination filtering

**Files:**
- Modify: `gameplay-editor/agents/audio-analyzer.md` (add new subsection after step 4 in Stage 2)

- [ ] **Step 1: Add hallucination filter after SRT parsing**

In `agents/audio-analyzer.md`, after step 4 ("Parse `voice.srt` for segment-level timestamps and text."), insert a new step 5:

```markdown
5. **Hallucination filter:** Before using the transcript for scoring, check for signs of Whisper hallucination:
   - **Repeated bigrams:** Extract all bigrams (consecutive word pairs) from the full transcript. If any single bigram appears more than 3 times, flag the transcript as potentially hallucinated.
   - **Dominant word frequency:** Count word frequencies across the full transcript. If the most common word accounts for more than 50% of all words, flag the transcript.
   - If either check triggers, discard the transcript for ALL scoring purposes — do NOT use it for Emotion scoring OR Drama Detection's `transcript_emotion_bonus` (set `transcript_emotion_bonus = 1.0` for all windows). Still save the `.srt` file for manual review. Report: `"Whisper transcript flagged as hallucinated (repeated bigrams: X, dominant word: Y at Z%). Transcript-based scoring disabled for this session."`
   - If neither check triggers, proceed normally with the transcript for Emotion analysis.
```

- [ ] **Step 2: Commit**

```bash
git add gameplay-editor/agents/audio-analyzer.md
git commit -m "feat(audio-analyzer): add Whisper hallucination filtering

Checks for repeated bigrams (>3 occurrences) and dominant word frequency
(>50%) before using transcript for Emotion scoring. Discards hallucinated
transcripts from scoring while preserving the SRT for manual review."
```

---

## Chunk 2: Emotion Dimension + Scoring Formula Alignment

### Task 3: Add Emotion dimension computation

**Files:**
- Modify: `gameplay-editor/agents/audio-analyzer.md` (add new subsection in Stage 4, before Composite Scoring)

- [ ] **Step 1: Add Emotion Analysis subsection in Stage 4**

In `agents/audio-analyzer.md`, in Stage 4, after the "Drama Detection" subsection and before "Crosstalk Detection (Multi-track only)", insert:

Insert this markdown block into `audio-analyzer.md` (not inside a code fence — this is raw markdown content to be inserted directly):

---

**Content to insert:**

> ### Emotion Analysis (Full Tier only)
>
> If Whisper transcript is available AND passed hallucination filtering:
>
> For each 5-second window, scan the corresponding transcript segments for emotional markers:
> - Count of exclamation marks (`!`)
> - Count of interrobangs (`?!`)
> - Count of ALL CAPS words (≥2 characters, excluding common abbreviations like "OK", "GG", "AFK")
> - Count of words with repeated letters (≥3 consecutive same letter, e.g., "NEEEEM", "WHAAAAT", "nooo")
>
> Compute the per-window Emotion score:
>
>     emotion_raw = (exclamation_count × 1.0) + (interrobang_count × 1.5) + (caps_word_count × 2.0) + (repeated_letter_count × 1.5)
>
> Normalize `emotion_raw` across all windows to 0-1 range (min-max normalization, same as other dimensions).
>
> If the transcript was flagged as hallucinated or Whisper was unavailable, set all Emotion values to 0 and exclude Emotion from the scoring formula.

---

- [ ] **Step 2: Commit**

```bash
git add gameplay-editor/agents/audio-analyzer.md
git commit -m "feat(audio-analyzer): add Emotion dimension from Whisper transcript

Computes per-window emotion score from exclamation marks, interrobangs,
ALL CAPS words, and repeated-letter words in the transcript."
```

### Task 4: Replace Full Tier scoring with scoring-weights.md formula

**Files:**
- Modify: `gameplay-editor/agents/audio-analyzer.md` (replace Composite Scoring section)

- [ ] **Step 1: Replace the Full Tier scoring table and formula**

In `agents/audio-analyzer.md`, replace the entire "Composite Scoring" subsection (from `### Composite Scoring` through the exclamation bonus paragraph). Insert the following raw markdown content directly (not inside a code fence):

---

**Content to replace the Composite Scoring subsection with:**

> ### Composite Scoring
>
> **Full Tier (with Whisper + valid transcript):**
>
> Uses the tuned 6-dimension formula from `docs/scoring-weights.md`:
>
>     score = 0.175 × Volume + 0.225 × HighFreq + 0.25 × Visual + 0.125 × DynRange + 0.15 × Breadth + 0.075 × Emotion
>
> | Dimension | Weight | Source |
> |-----------|--------|--------|
> | Volume | 0.175 | Per-window RMS loudness relative to session mean (from Loudness Analysis) |
> | High Freq | 0.225 | Ratio of high-frequency energy >2kHz (from High-Frequency Energy) |
> | Visual | 0.25 | Scene change intensity + frame diff per window (from Visual Motion Analysis) |
> | Dynamic Range | 0.125 | Per-second amplitude std within each window (from Loudness Analysis) |
> | Breadth | 0.15 | Signal diversity bonus: `active_dimensions / 5`. A dimension is "active" if its normalized value > 0.08. The five dimensions counted: Volume, High Freq, Visual, Dynamic Range, Emotion. |
> | Emotion | 0.075 | Transcript punctuation/caps analysis (from Emotion Analysis) |
>
> All weights sum to 1.0. Each dimension is normalized to 0-1 range (min-max across all windows in the session). The weighted sum is the raw score per window.
>
> **Full Tier (with Whisper, transcript hallucinated or empty):**
>
> Falls back to the 5-dimension formula without Emotion:
>
>     score = 0.20 × Volume + 0.25 × HighFreq + 0.25 × Visual + 0.15 × DynRange + 0.15 × Breadth
>
> Breadth counts 4 dimensions (Volume, High Freq, Visual, Dynamic Range). Drama Detection's `transcript_emotion_bonus` is also set to 1.0 (neutral) when transcript is hallucinated.
>
> **Minimal Tier (no Whisper):**
>
>     score = 0.20 × Volume + 0.25 × HighFreq + 0.25 × Visual + 0.15 × DynRange + 0.15 × Breadth
>
> | Dimension | Weight | Source |
> |-----------|--------|--------|
> | Volume | 0.20 | Per-window RMS loudness relative to session mean |
> | High Freq | 0.25 | High-frequency energy ratio (>2kHz) |
> | Visual | 0.25 | Scene change intensity + frame diff per window |
> | Dynamic Range | 0.15 | Per-second amplitude std within each window |
> | Breadth | 0.15 | Signal diversity: `active_dimensions / 4`. Counts: Volume, High Freq, Visual, Dynamic Range. Active if normalized > 0.08. |

---

- [ ] **Step 2: Commit**

```bash
git add gameplay-editor/agents/audio-analyzer.md
git commit -m "feat(audio-analyzer): align scoring with scoring-weights.md formula

Replace old Full/Minimal tier weights with the tuned 6-dimension formula
(Volume, HighFreq, Visual, DynRange, Breadth, Emotion). When Emotion is
active, Volume, HighFreq, and DynRange each reduce by 0.025 to maintain
sum = 1.0. Breadth now counts Emotion as an active dimension."
```

### Task 5: Update scoring-weights.md to reflect Emotion is now active

**Files:**
- Modify: `gameplay-editor/docs/scoring-weights.md`

- [ ] **Step 1: Update the main formula**

In `scoring-weights.md`, replace the formula block:
```
score = 0.20 × Volume + 0.25 × HighFreq + 0.25 × Visual + 0.15 × DynRange + 0.15 × Breadth
```
with:
```
# With valid Whisper transcript:
score = 0.175 × Volume + 0.225 × HighFreq + 0.25 × Visual + 0.125 × DynRange + 0.15 × Breadth + 0.075 × Emotion

# Without Whisper or with hallucinated transcript:
score = 0.20 × Volume + 0.25 × HighFreq + 0.25 × Visual + 0.15 × DynRange + 0.15 × Breadth
```

- [ ] **Step 2: Update the Dimensions table**

Change the Emotion row from weight `—` to `0.075` and update its description. Also update the note about it not being in the base formula:

| Dimension | Weight | Source | Description |
|-----------|--------|--------|-------------|
| Volume | **0.175** | ffmpeg `astats` | Per-window RMS loudness relative to session mean. (0.20 without Emotion) |
| High Freq | **0.225** | ffmpeg spectral analysis | Ratio of high-frequency energy (>2kHz). (0.25 without Emotion) |
| Visual | **0.25** | ffmpeg `select='gt(scene,0.3)'` | Scene change intensity per window. |
| Dynamic Range | **0.125** | ffmpeg `astats` (per-second std) | Amplitude variation within each 5s window. (0.15 without Emotion) |
| Breadth | **0.15** | Computed | Signal diversity bonus: `active_dimensions / 5` (or `/4` without Emotion). Active if normalized > 0.08. |
| Drama | **0.00** | — | Disabled: produces false positives. |
| Emotion | **0.075** | Whisper transcript (small model) | Exclamation marks, interrobangs, ALL CAPS words, repeated-letter words. Only active with valid (non-hallucinated) Whisper output. When inactive, Volume/HighFreq/DynRange revert to base weights. |

- [ ] **Step 3: Update the Breadth Bonus section**

Update the breadth section to reflect 5 dimensions when Emotion is active:

| Active dims (with Emotion) | Breadth value | Bonus points (x0.15) |
|:--------------------------:|:-------------:|:--------------------:|
| 5/5 | 1.00 | ~15 pts |
| 4/5 | 0.80 | ~12 pts |
| 3/5 | 0.60 | ~9 pts |
| 2/5 | 0.40 | ~6 pts |
| 1/5 | 0.20 | ~3 pts |

Note: When Emotion is inactive, breadth reverts to 4 dimensions (same as current table).

- [ ] **Step 4: Replace the "Whisper Integration (Optional)" section**

Replace the entire section at the bottom. Insert the following raw markdown content directly (not inside a code fence):

---

**Content to replace the "Whisper Integration (Optional)" section with:**

> ## Whisper Integration
>
> Whisper (`small` model) provides the Emotion dimension when available.
>
> **Model:** `small` (chosen over `base` for better accuracy on long recordings with game audio bleed).
>
> **Hallucination filtering:** Before using the transcript, check for:
> - Repeated bigrams: any bigram appearing >3 times flags the transcript
> - Dominant word frequency: most common word >50% of total words flags the transcript
>
> If flagged, Emotion is disabled and scoring reverts to the 5-dimension base formula. Drama Detection's `transcript_emotion_bonus` also reverts to 1.0 (neutral).
>
> **When Emotion is active**, weights shift:
> - Volume: 0.20 → 0.175 (-0.025)
> - High Freq: 0.25 → 0.225 (-0.025)
> - Dynamic Range: 0.15 → 0.125 (-0.025)
> - Emotion: 0.00 → 0.075 (+0.075)
> - Visual and Breadth: unchanged

---

- [ ] **Step 5: Commit**

```bash
git add gameplay-editor/docs/scoring-weights.md
git commit -m "docs(scoring-weights): activate Emotion dimension, document small model

Emotion (0.075 weight) is now active when Whisper produces a valid
transcript. Volume, HighFreq, and DynRange each reduce by 0.025 to
compensate. Documents hallucination filtering and small model choice."
```

### Task 6: Update output JSON to include Emotion signal

**Files:**
- Modify: `gameplay-editor/agents/audio-analyzer.md` (output section)

- [ ] **Step 1: Add `emotion` to the signals array in the output example**

In the output JSON example in `audio-analyzer.md`, update the first moment's signals to include `"emotion"`:
```json
"signals": ["volume_spike", "high_freq", "crosstalk", "visual_motion", "emotion"],
```

- [ ] **Step 2: Add `has_emotion` flag to the top-level output**

In the output JSON structure, add after `"transcript_srt"`:
```json
"has_emotion": true,
```

This tells the calling command whether Emotion scoring was active (i.e., Whisper ran and transcript passed hallucination filter). When `false`, the caller knows the 5-dimension base formula was used.

- [ ] **Step 3: Commit**

```bash
git add gameplay-editor/agents/audio-analyzer.md
git commit -m "feat(audio-analyzer): add emotion signal and has_emotion flag to output

The has_emotion flag indicates whether Emotion scoring was active for the
session, letting the calling command know which formula was used."
```
