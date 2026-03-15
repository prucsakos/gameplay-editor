---
name: transcript-analyzer
description: |
  Use this agent to analyze a Whisper transcript and score moments for excitement.
  Produces per-window scores, discovers missed moments, writes clip summaries, and refines cut boundaries.
model: sonnet
---

# Transcript Analyzer Agent

You analyze Whisper `.srt` transcripts to score excitement, discover moments missed by audio analysis, write clip summaries, and refine cut boundaries to sentence edges.

## Input

You receive:
- **transcript_srt**: path to the `.srt` file produced by Whisper
- **moments**: audio-analyzer's full moment list (JSON array with timestamps, scores, signals, descriptions)
- **transcript_signals**: list of signals from the active style (e.g., `["humor", "dramatic reactions", "banter"]`)
- **language**: Whisper language code (e.g., `hu`)
- **source_duration**: total video length in seconds
- **window_size**: window size in seconds (default: 5.0)

Your job is to process the transcript and return structured scoring, discoveries, summaries, and refined timestamps.

## Step 1: Parse the Transcript

Read the `.srt` file with explicit `encoding='utf-8'` (Windows may not default to UTF-8).

Parse each subtitle segment into:
- **index**: subtitle sequence number
- **start**: start timestamp as float seconds
- **end**: end timestamp as float seconds
- **text**: subtitle text content

Map each segment to 5-second window(s) by timestamp overlap. A segment spanning `00:01:03.500` to `00:01:07.200` overlaps windows `[60.0, 65.0)` and `[65.0, 70.0)`.

## Step 2: Window Scoring

For each 5-second window from `0` to `source_duration`:

1. Collect all transcript text from `.srt` segments that overlap this window
2. Score the window 0-10 based on the `transcript_signals` criteria provided by the style

**Scoring scale:**
| Range | Meaning |
|-------|---------|
| 0 | Silence or completely uninteresting |
| 1-3 | Normal conversation, no signal match |
| 4-6 | Mild signal presence (slight humor, minor reaction) |
| 7-8 | Strong signal match (clear humor, dramatic moment) |
| 9-10 | Peak excitement (multiple strong signals converging) |

- Windows with no transcript coverage score **0**
- **Score based on the `transcript_signals` list provided by the style.** Do not hardcode what to look for — the style controls the scoring criteria.

**Normalization:**

Normalize all raw scores to 0-1 range:

```
normalized = raw_score / max(all_raw_scores)
```

If all raw scores are zero, all normalized scores remain zero.

## Step 3: Moment Discovery

Find high-scoring transcript windows that fall **outside** any audio-analyzer moment's `[start, end]` range. These are moments where the dialogue is interesting but audio energy was low (quiet humor, whispered reactions, calm dramatic reveals).

1. Identify windows outside all existing moment ranges with normalized score > 0.5
2. Cluster adjacent qualifying windows into continuous discovered moments
3. Align start/end to sentence boundaries (using Step 5 rules: pull start back to `.srt` segment start if mid-sentence, max 3s; extend end to `.srt` segment end, max 3s)
4. Discard any discovered moment that overlaps with an existing audio-analyzer moment

For each discovered moment, record:
- Start/end timestamps (HH:MM:SS format)
- Transcript score (normalized)
- Which `transcript_signals` triggered (e.g., `["humor", "banter"]`)
- A summary (see Step 4)

## Step 4: Clip Summaries

For **every** moment — both audio-analyzer's existing moments (identified by `original_index`) and newly discovered ones — write a 1-2 sentence summary:

- Describe what's happening in the conversation
- Reference which `transcript_signals` are present
- For existing audio-analyzer moments, use their `audio_description` and `screen_description` fields as additional context
- Written in the **language of the transcript** (e.g., if the gameplay is in Hungarian, summaries are in Hungarian)
- Be concise and descriptive, not generic. Avoid filler phrases like "an exciting moment occurs" — say what actually happens.

## Step 5: Cut Boundary Refinement

For each moment (existing + discovered), check whether the moment boundaries fall mid-sentence in the `.srt` file:

- **Start:** If the moment starts mid-sentence, pull start back to the beginning of that `.srt` segment. Maximum adjustment: **3 seconds backward**.
- **End:** If the moment ends mid-sentence, extend end to the end of that `.srt` segment. Maximum adjustment: **3 seconds forward**.
- **Overlap protection:** If refinement would cause overlap with an adjacent moment, keep the original boundary. Neither moment wins — the safe choice is to not extend.

Report both original and refined timestamps, plus a brief reason for each adjustment.

## Output

Return a structured result. Print it as a fenced JSON code block so the calling command can parse it:

```json
{
  "transcript_scores": [
    { "window_start": 0.0, "window_end": 5.0, "score": 0.12 },
    { "window_start": 5.0, "window_end": 10.0, "score": 0.05 },
    { "window_start": 10.0, "window_end": 15.0, "score": 0.00 },
    { "window_start": 745.0, "window_end": 750.0, "score": 0.91 }
  ],
  "discovered_moments": [
    {
      "start": "00:45:12",
      "end": "00:45:38",
      "transcript_score": 0.82,
      "signals": ["humor", "banter"],
      "summary": "Péter halkan megjegyzi hogy a másik csapat taktikája 'nagyon profi' miközben azok egymásnak futnak a falon — tipikus száraz humor"
    }
  ],
  "moment_refinements": [
    {
      "original_index": 1,
      "summary": "Mindenki egyszerre kiabál amikor a csapattárs véletlenül felrobbantja az egész bázist — kaotikus nevetés és pánik",
      "original_start": "00:12:28",
      "original_end": "00:13:07",
      "refined_start": "00:12:26",
      "refined_end": "00:13:08",
      "refinement_reason": "Extended start 2s to include sentence beginning; extended end 1s to complete sentence"
    },
    {
      "original_index": 2,
      "summary": "4 másodperc csend után hirtelen ordítás — drámai feszültség-feloldás a menü képernyő után",
      "original_start": "00:34:12",
      "original_end": "00:34:55",
      "refined_start": "00:34:12",
      "refined_end": "00:34:55",
      "refinement_reason": "No adjustment needed — boundaries align with sentence edges"
    }
  ]
}
```

### Output Rules

- **`transcript_scores`** must cover **every** 5-second window from `0` to `source_duration`. Use float seconds for window boundaries.
- **`moment_refinements`** must have exactly **one entry for every input moment** (by `original_index`). Use HH:MM:SS strings for timestamps.
- **`discovered_moments`** may be empty if no qualifying windows are found outside existing moments. Use HH:MM:SS strings for timestamps.
- **All summaries** must be written in the transcript's language.
