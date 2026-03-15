# Transcript Analyzer Agent — Design Spec

> **Note:** The scoring formulas in this spec have been **superseded** by `docs/superpowers/specs/2026-03-15-v1-mechanics-restoration-design.md`. The transcript-analyzer agent behavior (window scoring, moment discovery, summaries, cut boundary refinement) is unchanged. Only the composite scoring formula used by gameplay-edit has changed.

**Date:** 2026-03-15
**Status:** Reviewed
**Supersedes:** Inline Emotion dimension from original plan (`2026-03-15-whisper-emotion-scoring.md` Tasks 2-3)

## Goal

Add a dedicated transcript analyzer agent to the gameplay-editor pipeline that uses LLM-powered analysis of Whisper transcripts to score moments, discover moments the audio-analyzer missed, write clip summaries, and refine cut boundaries to sentence edges. The agent's output becomes a new **Transcript** dimension in the scoring formula.

## Context

The audio-analyzer currently handles all excitement detection using ffmpeg signal processing (volume, high-frequency energy, visual motion, dynamic range). Whisper transcription exists but is only used for basic `transcript_emotion_bonus` multipliers in Drama Detection and `transcript_excerpt` fields.

The original plan proposed inline punctuation/caps counting (Emotion dimension) with hallucination filtering. This design replaces that with a proper agent that can understand language, humor, and conversational dynamics — things signal processing cannot capture.

## Architecture

```
Whisper (small) ──► .srt file
                        │
                        ├──► audio-analyzer (ffmpeg signals) ──► preliminary moments (5-dim scores)
                        │
                        └──► transcript-analyzer (LLM) ◄── preliminary moments + style signals
                                    │
                                    ├── per-window Transcript scores (0-1)
                                    ├── newly discovered moments
                                    ├── clip summaries
                                    └── refined cut boundaries
                                            │
                                            ▼
                              gameplay-edit recomputes 6-dim scores, re-ranks, presents
```

The transcript-analyzer runs **after** the audio-analyzer (sequential, not parallel). It receives the audio-analyzer's moment list as context so it can reinforce findings, spot gaps, refine boundaries, and write informed summaries.

## Transcript Analyzer Agent

**File:** `gameplay-editor/agents/transcript-analyzer.md`
**Model:** Sonnet

### Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `transcript_srt` | path | Path to the `.srt` file produced by Whisper |
| `moments` | JSON array | Audio-analyzer's full moment list (timestamps, scores, signals, descriptions) |
| `transcript_signals` | string list | From the active style's `transcript_signals` field |
| `language` | string | Whisper language code (e.g., `hu`) |
| `source_duration` | float | Total video length in seconds |
| `window_size` | float | Window size in seconds (default: 5.0, matches audio-analyzer) |

### Processing

#### 1. Window Scoring

Read the full `.srt` transcript. For each 5-second window across the entire video duration:

- Map `.srt` segments to windows by timestamp overlap
- Score how well the dialogue in that window matches the `transcript_signals` criteria
- Output a raw score per window
- Normalize all window scores to 0-1 range: `normalized = raw_score / max(all_raw_scores)`. If all scores are 0, all normalized scores are 0.

This normalized per-window array becomes the **Transcript** dimension, consumed by the gameplay-edit command's 6-dimension scoring formula.

The agent scores based on whatever `transcript_signals` the style defines. It does not hardcode what to look for — the style controls this.

#### 2. Moment Discovery

Scan the transcript for exciting passages that fall **outside** the audio-analyzer's detected moment windows. These are moments where:
- The dialogue is clearly interesting/exciting per the style's `transcript_signals`
- But the audio energy was low (quiet humor, whispered reactions, calm dramatic reveals)

For each discovered moment, output:
- Start/end timestamps (aligned to sentence boundaries)
- A transcript-based score (raw, same scale as window scores)
- Which `transcript_signals` triggered (e.g., `["humor", "banter"]`)

#### 3. Clip Summaries

For every moment — both audio-analyzer's existing moments and newly discovered ones — write a 1-2 sentence summary:
- Describe what's happening in the conversation
- Reference why it's interesting (which signals are present)
- Written in the language of the transcript (e.g., if the gameplay is in Hungarian, summaries are in Hungarian; if mixed, use the dominant language)

These summaries replace the current `transcript_excerpt` field in the final moment list.

#### 4. Cut Boundary Refinement

For each moment (existing + discovered):
- Check whether the moment's start timestamp falls mid-sentence in the `.srt`
  - If so, pull start back to the beginning of that sentence
- Check whether the moment's end timestamp falls mid-sentence
  - If so, extend end to the end of that sentence
- Report both original and refined timestamps
- Maximum adjustment: 3 seconds in either direction (don't over-extend into unrelated content)
- If refinement would cause overlap with an adjacent moment, keep the original boundary

### Output

```json
{
  "transcript_scores": [
    { "window_start": 0.0, "window_end": 5.0, "score": 0.12 },
    { "window_start": 5.0, "window_end": 10.0, "score": 0.05 },
    ...
  ],
  "discovered_moments": [
    {
      "start": "00:45:12",
      "end": "00:45:38",
      "transcript_score": 0.82,
      "signals": ["humor", "banter"],
      "summary": "..."
    }
  ],
  "moment_refinements": [
    {
      "original_index": 1,
      "summary": "...",
      "original_start": "00:12:28",
      "original_end": "00:13:07",
      "refined_start": "00:12:26",
      "refined_end": "00:13:08",
      "refinement_reason": "Extended start by 2s to include sentence beginning; extended end by 1s to complete sentence"
    },
    ...
  ]
}
```

## Style Guide Extension

**File:** `gameplay-editor/styles/clean.md`

Add `transcript_signals` to the YAML frontmatter:

```yaml
transcript_signals:
  - humor
  - dramatic reactions
  - surprised exclamations
  - banter
  - storytelling peaks
  - emotional outbursts
  - comedic timing
```

The transcript analyzer reads this list and uses it as its scoring criteria. Different styles can define different signal lists (e.g., a "competitive" style might look for callouts, strategy discussion, clutch moments).

**Fallback when `transcript_signals` is absent:** If the active style does not define `transcript_signals`, the transcript-analyzer is skipped entirely and scoring falls back to the 5-dimension Minimal Tier formula. The gameplay-edit command checks for this field before dispatching the transcript-analyzer.

## Scoring Formula Update

**File:** `gameplay-editor/docs/scoring-weights.md`

The dimension formerly proposed as "Emotion" is now **Transcript**. Weight: **0.075**.

### With valid transcript (Full Tier):

```
score = 0.175 × Volume + 0.225 × HighFreq + 0.25 × Visual + 0.125 × DynRange + 0.15 × Breadth + 0.075 × Transcript
```

| Dimension | Weight | Source |
|-----------|--------|--------|
| Volume | 0.175 | ffmpeg `astats` — per-window RMS loudness relative to session mean |
| High Freq | 0.225 | ffmpeg spectral analysis — high-frequency energy ratio (>2kHz) |
| Visual | 0.25 | ffmpeg scene detection + frame diff |
| Dynamic Range | 0.125 | ffmpeg `astats` — per-second amplitude std within each window |
| Breadth | 0.15 | `active_dimensions / 5`. Active if normalized > 0.08. Counts: Volume, HighFreq, Visual, DynRange, Transcript |
| Transcript | 0.075 | transcript-analyzer agent — LLM-scored per-window transcript excitement |

Sum = 1.0. Weight shifts from base: Volume -0.025, HighFreq -0.025, DynRange -0.025, Transcript +0.075.

### Without transcript (Minimal Tier):

```
score = 0.20 × Volume + 0.25 × HighFreq + 0.25 × Visual + 0.15 × DynRange + 0.15 × Breadth
```

Breadth counts 4 dimensions (Volume, HighFreq, Visual, DynRange). Breadth = `active_dimensions / 4`. Sum = 1.0.

### Breadth Bonus Table

**With Transcript (Full Tier):**

| Active dims | Breadth value | Bonus points (x0.15) |
|:-----------:|:-------------:|:--------------------:|
| 5/5 | 1.00 | ~15 pts |
| 4/5 | 0.80 | ~12 pts |
| 3/5 | 0.60 | ~9 pts |
| 2/5 | 0.40 | ~6 pts |
| 1/5 | 0.20 | ~3 pts |

**Without Transcript (Minimal Tier):**

| Active dims | Breadth value | Bonus points (x0.15) |
|:-----------:|:-------------:|:--------------------:|
| 4/4 | 1.00 | ~15 pts |
| 3/4 | 0.75 | ~11 pts |
| 2/4 | 0.50 | ~7.5 pts |
| 1/4 | 0.25 | ~3.75 pts |

A dimension is "active" if its normalized value > 0.08.

## Pipeline Integration

**File:** `gameplay-editor/commands/gameplay-edit.md`

Updated pipeline (analyze mode):

Analyze mode (6 steps):
```
[1/6] Probing audio tracks...
[2/6] Running Whisper transcription (small model)...
[3/6] Analyzing audio and visual signals...           ← audio-analyzer
[4/6] Analyzing transcript...                         ← transcript-analyzer (NEW)
[5/6] Presenting analysis for review...
[6/6] Assembling edit...
```

Auto mode (5 steps — skip user review):
```
[1/5] Probing audio tracks...
[2/5] Running Whisper transcription (small model)...
[3/5] Analyzing audio and visual signals...           ← audio-analyzer
[4/5] Analyzing transcript...                         ← transcript-analyzer (NEW)
[5/5] Assembling edit...
```

If Whisper is unavailable or `has_transcript` is `false`, steps 4 (transcript-analyzer) is skipped and step numbers adjust accordingly (5 steps in analyze, 4 in auto — same as current pipeline).

Step 4 details:
- Dispatch transcript-analyzer with: `.srt` path, audio-analyzer's moments, style's `transcript_signals`, language, source duration
- Receive: per-window Transcript scores, discovered moments, summaries, refined timestamps

Step 5 (between transcript-analyzer and presentation):
- Merge discovered moments into the audio-analyzer's moment list
- For each window, compute the 6-dimension composite score using audio-analyzer's Volume/HighFreq/Visual/DynRange + transcript-analyzer's Transcript scores + Breadth
- Apply merging and padding rules (adjacent windows within 10s, 2s lead-in / 1s lead-out)
- Use refined timestamps from transcript-analyzer instead of originals
- Replace `transcript_excerpt` with transcript-analyzer's summaries
- Re-rank all moments by composite score
- Present to user with summaries

The edit-assembler receives the final refined timestamps.

## Edge Cases

**Empty transcript (`.srt` with zero speech segments):** `has_transcript` is `false`. Transcript-analyzer is not dispatched. Scoring uses the 5-dimension Minimal Tier formula.

**Partial Whisper failure (`.srt` covers only part of the video):** `has_transcript` is `true` (there is usable content). The transcript-analyzer processes whatever segments exist. Windows with no transcript coverage get a Transcript score of 0. Breadth for uncovered windows counts only 4 dimensions (Transcript is inactive for those windows since its value is 0, below the 0.08 activation threshold).

**Whisper unavailable:** `has_transcript` is `false`. Transcript-analyzer is skipped. Pipeline falls back to current behavior (5-dim Minimal Tier).

**Style missing `transcript_signals`:** Transcript-analyzer is skipped. 5-dimension Minimal Tier formula used.

**Cut boundary refinement overlap:** When two adjacent moments both want to extend toward each other and would overlap, both keep their original boundaries. Neither wins — the safe choice is to not extend.

**Discovered moment overlaps with existing moment:** If a transcript-analyzer discovered moment overlaps in time with an existing audio-analyzer moment, the discovered moment is discarded — the audio-analyzer already captured that region. The transcript-analyzer's summary and Transcript score for those windows still contribute to the existing moment's composite score.

## Timestamp Format Convention

Two timestamp formats are used in the pipeline:
- **Float seconds** (e.g., `0.0`, `5.0`): Used in per-window arrays (`transcript_scores`, `window_dimensions`) for precise computation
- **HH:MM:SS strings** (e.g., `"00:12:28"`): Used in moment-level data (`discovered_moments`, `moment_refinements`) for human readability

The gameplay-edit command handles both formats. Per-window arrays use float seconds because they're consumed programmatically. Moment timestamps use HH:MM:SS because they appear in user-facing output.

## Changes to audio-analyzer.md

- Switch Whisper from `base` to `small` model
- Keep all ffmpeg-based signal processing unchanged
- **Remove both** transcript-dependent scoring mechanisms: (a) the Drama Detection `transcript_emotion_bonus` multiplier (values 1.0/1.3/1.5 — set to 1.0 always) and (b) the Composite Scoring exclamation bonus (`+0.1 per exclamation, capped at +0.3`). These are separate code paths in audio-analyzer.md that both need removal.
- Output adds `has_transcript: true/false` flag — `true` if Whisper produced a `.srt` with at least one speech segment, `false` if Whisper failed, was unavailable, or the `.srt` contains zero segments
- **New output field:** `window_dimensions` — per-window raw values for each ffmpeg dimension (Volume, HighFreq, Visual, DynRange), normalized to 0-1. This data is needed by gameplay-edit to compute 6-dimension composite scores for transcript-analyzer's discovered moments. Format:

```json
"window_dimensions": [
  { "window_start": 0.0, "window_end": 5.0, "volume": 0.32, "high_freq": 0.15, "visual": 0.08, "dyn_range": 0.22 },
  { "window_start": 5.0, "window_end": 10.0, "volume": 0.41, "high_freq": 0.67, "visual": 0.55, "dyn_range": 0.38 },
  ...
]
```

This replaces the current behavior where the audio-analyzer only outputs moment-level data. The per-window arrays are the raw normalized dimension values computed during Stage 4 scoring, before moment merging. They are always included in the output regardless of tier.

## Changes to gameplay-edit.md

- Update analyze mode pipeline from 5 steps to 6 steps; auto mode from 4 to 5 steps
- After audio-analyzer dispatch, add: check `has_transcript` flag AND style has `transcript_signals` — if both true, dispatch transcript-analyzer
- If transcript-analyzer is dispatched, add score recomputation step: read `window_dimensions` from audio-analyzer + `transcript_scores` from transcript-analyzer, compute 6-dim composite scores per window, merge discovered moments (discarding overlaps with existing), apply merging/padding rules, re-rank
- Replace `transcript_excerpt` in moment output with transcript-analyzer's summaries
- Use refined timestamps from transcript-analyzer for assembly dispatch
- When transcript-analyzer is skipped (no Whisper, empty transcript, or missing `transcript_signals`): use audio-analyzer's 5-dim scores and original timestamps as-is (current behavior)
- Pass style's `transcript_signals` to the transcript-analyzer

## Changes to scoring-weights.md

- Rename Emotion → Transcript throughout
- Update formula to 6-dimension version
- Update Breadth table to show 5 active dimensions
- Replace "Whisper Integration (Optional)" section with documentation of the transcript-analyzer agent and the Transcript dimension
- Document the `small` model choice

## What This Design Removes from the Original Plan

| Original Plan Item | Disposition |
|---|---|
| Task 2: Whisper hallucination filtering (bigram/frequency heuristics) | **Removed.** The transcript analyzer is an LLM — it can judge transcript quality naturally without heuristic pre-filtering. |
| Task 3: Inline Emotion computation (punctuation/caps counting) | **Replaced** by transcript-analyzer agent's per-window scoring. |
| Task 4: Composite Scoring replacement in audio-analyzer | **Moved.** Score recomputation happens in gameplay-edit command, not audio-analyzer. Audio-analyzer uses the 5-dim base formula. |
| Task 6: `has_emotion` flag | **Renamed** to `has_transcript`. |
| Task 1: Whisper `base` → `small` | **Kept.** |
| Task 5: scoring-weights.md updates | **Kept** with Emotion → Transcript rename. |
