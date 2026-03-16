---
description: Edit gameplay videos into highlight reels or short-form clips
argument-hint: <video_path> [--platform youtube|tiktok|both] [--language hu] [--score-threshold 70] [--duration 5m] [--mode analyze|auto] [--output path]
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(python:*), Bash(python3:*), Bash(ls:*), Bash(mkdir:*), Bash(rm:*), Bash(cat:*), Bash(head:*), Bash(date:*), Bash(stat:*), Bash(mktemp:*), Bash(df:*)
---

# Gameplay Video Editor

You are editing a gameplay video into a highlight reel or short-form clips.

## Parse Arguments

From the user's input, extract:
- **video_path** (required): path to the source video file
- **platform** (default: `youtube`): one of `youtube`, `tiktok`, `both`
- **language** (default: `hu`): Whisper language code
- **score-threshold** (default: `70`): minimum score to include a moment (0-100). Used in both auto and analyze modes as the default filter.
- **duration** (optional): target duration like `5m`, `10m`, `60s`. If provided, overrides score-threshold — selects the highest-scoring moments that fit within the target duration.
- **mode** (default: `analyze`): one of `analyze`, `auto`
- **output** (default: `Outs/<source_basename>_edit.mp4` relative to source video's parent directory). For TikTok clips, each file includes the moment's score: `<basename>_s<SCORE>_tk_<NN>.mp4`

## Windows Compatibility

**PYTHONUTF8=1**: All `python3`/`python` invocations MUST be prefixed with `PYTHONUTF8=1`. Windows defaults to cp1250 encoding which crashes on non-ASCII characters in stdout. This applies to every Python one-liner and heredoc script in the pipeline.

**ASCII-only output**: All printed output from Python scripts must use ASCII characters only. No `★`, `→`, `×`, `─` — use `*`, `->`, `x`, `-` instead.

## Validate

1. Check the video file exists: `ls -la "<video_path>"`
2. Check ffmpeg is available: `ffmpeg -version`
3. Check video has audio tracks: `ffprobe -v quiet -print_format json -show_streams "<video_path>"` — if no stream has `codec_type == "audio"`, report "Error: No audio tracks found in source video" and stop
4. Read the style file from this plugin's `styles/clean.md` — parse the YAML frontmatter for parameters
5. Probe source video for resolution and fps:
   ```bash
   ffprobe -v quiet -print_format json -show_streams "<video_path>"
   ```
   Extract: width, height, fps (from `avg_frame_rate`).
6. Create the output directory if it doesn't exist
7. Estimate required disk space (source_size * 1.5) and warn if <2GB free
8. Record pipeline start time:
   ```bash
   PIPELINE_START=$(date +%s%N)
   ```

## Check faster-whisper Availability

Run `PYTHONUTF8=1 python3 -c "from faster_whisper import WhisperModel; print('ok')"` quietly. If it works, report: `"Running with faster-whisper (Full tier) — transcription + content analysis"`
If not: `"faster-whisper not available — using volume-based analysis only (Minimal tier). Run /gameplay-setup for better results."`

## Single Prompt (analyze mode only)

If mode is `analyze` AND not all parameters were provided via CLI flags, present one prompt:

```
Quick setup:
- Language? (default: hu)
- Platform? youtube / tiktok / both (default: youtube)
- Score threshold? (default: 70) — include all moments above this score
```

Parse the user's response flexibly:
- They can answer all, some, or none (enter = all defaults)
- Match answers to fields by keyword (e.g., "tiktok" → platform, "en" → language, "50" → score_threshold)
- If ambiguous, ask for clarification

If mode is `auto` OR all parameters were provided via CLI flags, skip the prompt. Use defaults for any missing values.

## Execute

### Create temp directory

Use Python's `tempfile` to create the temp directory — this ensures the path is a native Windows path that both bash and Python can resolve consistently. Do NOT use `mktemp -d "/tmp/..."` on Windows, as MSYS2/Git Bash maps `/tmp/` differently than Python.

```bash
TMP=$(PYTHONUTF8=1 python3 -c "import tempfile; print(tempfile.mkdtemp(prefix='gameplay-editor-'))")
```

Verify the directory exists before proceeding: `ls -d "$TMP"`

### Run Audio Analysis

Dispatch the **audio-analyzer** agent with:
- video_path
- language
- tmp_dir: $TMP

The agent returns a JSON result with all detected moments (preliminary 3-dim scores), per-window dimension arrays (`window_dimensions`), timing data, noise floor, and `has_transcript` flag.

### Run Transcript Analysis (Full Tier only)

**Gate:** Only dispatch the transcript-analyzer if ALL of the following are true:
1. The audio-analyzer's output has `has_transcript: true`
2. The active style defines `transcript_signals` (parsed from frontmatter)

If either condition is false, skip this step. Use the audio-analyzer's 3-dim scores as final scores and its original timestamps. The pipeline step count adjusts accordingly (3 steps in analyze mode, 2 in auto mode).

**Partial transcript:** If `has_transcript` is `true` but the `.srt` only covers part of the video (Whisper partial failure), dispatch proceeds normally. The transcript-analyzer handles uncovered windows by scoring them 0.

If both conditions are true, dispatch the **transcript-analyzer** agent with:
- transcript_srt: the `.srt` path from audio-analyzer output
- moments: the audio-analyzer's full moment list
- transcript_signals: from the active style's YAML frontmatter
- language
- source_duration: from audio-analyzer output
- window_size: 5.0

The agent returns per-window Transcript scores, discovered moments, clip summaries, and refined cut boundaries.

### Recompute Composite Scores (Full Tier only)

After receiving the transcript-analyzer's output, recompute scores using the Full Tier 5-dimension formula from `docs/scoring-weights.md`:

    score = 0.20 x Volume + 0.25 x HighFreq + 0.15 x DynRange + 0.15 x Breadth + 0.25 x Transcript

For each 5-second window:
1. Read the 3 audio dimensions from `window_dimensions` (audio-analyzer output)
2. Read the Transcript score from `transcript_scores` (transcript-analyzer output)
3. Compute Breadth: count how many of the 4 dimensions (Volume, HighFreq, DynRange, Transcript) have normalized value > 0.08. Breadth = `active_count / 4`
4. Compute weighted sum

**Merge discovered moments:**
- For each discovered moment from transcript-analyzer, check if it overlaps with any existing audio-analyzer moment
- If overlap: discard the discovered moment (the audio-analyzer already captured that region; the Transcript scores for those windows still contribute via recomputation)
- If no overlap: add it to the moment list. Its 3 audio dimension scores come from `window_dimensions` for the corresponding windows. Its Transcript score comes from `transcript_scores`.

**Apply merging and padding:**
- Adjacent windows (within 10s) that both score above the session mean merge into one continuous segment
- Segment score = peak window score within the segment
- Add 2s lead-in and 1s lead-out padding (no overlap with neighbors)

**Apply refined timestamps:**
- For each moment, use the transcript-analyzer's `refined_start` and `refined_end` instead of the originals

**Apply summaries:**
- For each moment, replace `transcript_excerpt` with the transcript-analyzer's `summary`

**Re-rank** all moments by composite score (descending).

The moment list with refined timestamps and summaries is what gets passed to the **edit-assembler** in the Assembly step and presented to the user in analyze mode.

### Present Plan (analyze mode)

Present the plan to the user. Include ALL detected moments (not just selected ones), with moments above threshold clearly marked:

```
=== Analysis Results ===
Source: recording.mkv (3h 24m)
Score threshold: 70 | Platform: youtube

All detected moments (37):
  * #1  [Score: 95] 00:12:28 → 00:13:07 (39s)
        Audio: Mass laughter, 3 voices overlapping, volume spike +12dB
        Summary: "Mindenki egyszerre kiabál amikor a csapattárs véletlenül felrobbantja az egész bázist"

  * #2  [Score: 88] 00:34:12 → 00:34:55 (43s)
        Audio: 4s silence → sudden shouting, dramatic tension-release
        Summary: "4 másodperc csend után hirtelen ordítás — NEEEEM! Várj... WHAT?!"

  * #3  [Score: 75] 00:48:01 → 00:48:28 (27s)
        Audio: Sustained high-freq energy (laughter), moderate volume
        Summary: "Folyamatos nevetés — ez nem lehet igaz"

    #4  [Score: 62] 01:05:18 → 01:05:42 (24s)
        Audio: Brief volume spike, single speaker
        Summary: "na jó, ez szar volt"

    #5  [Score: 55] 01:22:10 → 01:22:35 (25s)
        Audio: Moderate energy, no standout signals
        Summary: "figyelj ide..."
...

* = above threshold (70) — will be included
Total above threshold: 14 moments (6m 12s)
Total below threshold: 23 moments
```

Each moment MUST include two description lines:
- **Audio**: What's happening in the audio (volume spikes, laughter, silence, crosstalk, etc.)
- **Summary**: Clip summary from transcript-analyzer (if available), or transcript excerpt as fallback

Record user review start time. Wait for user response. They may:
- Approve: "looks good", "go", "just do it" → proceed to assembly with moments above threshold
- Set different threshold: "use score 60" or "lower to 50" → re-filter, re-present
- Set target duration instead: "make it 3 min" → select top-scoring moments to fill duration
- Include/exclude specific moments: "add #4", "remove #3" → update selection
- Change platform: "also do tiktok" → note for assembly

Record user review end time: `user_review_ms`.

### Auto Mode

Skip the plan presentation. Select moments automatically based on:
- If `--duration` was provided: select top-scoring moments that fit within the target duration
- Otherwise: include all moments with score ≥ `score-threshold` (default 70)

Proceed directly to assembly.

### Assembly

Dispatch the **edit-assembler** agent with:
- source_video_path
- The approved/selected moment list
- The style preset parameters (from YAML frontmatter)
- platform: `youtube` or `tiktok`
- output_path
- noise_floor_db (from analyzer output)
- source_width, source_height, source_fps

**If platform is `both`:** dispatch the edit-assembler TWICE:
1. First with platform=`youtube`, output=`<basename>_edit_yt.mp4`
2. Then with platform=`tiktok`, output_dir for `<basename>_edit_tk_NN.mp4` files

Collect timing data from the assembler's response.

## Pipeline Summary

After assembly completes, calculate total time and render the summary:

```bash
PIPELINE_END=$(date +%s%N)
TOTAL_MS=$(( (PIPELINE_END - PIPELINE_START) / 1000000 ))
```

Present the summary:

```
=== Pipeline Summary ===
Source: <filename> (<source_duration>)
Output: <output_path> (<output_duration>, <output_size>)
Moments: <included> included, <excluded> excluded
Score threshold: <threshold>

Timing:
  Audio analysis:      <audio_analysis time> (probe + transcription + scoring)
  Transcript analysis: <transcript_analysis time>  (only in Full Tier)
  User review:         <user_review time>  (only in analyze mode)
  Segment processing:  <segment_processing time>
  Export:              <concat_export time>
  ─────────────────────────
  Total:               <total time>

Platform: <platform> | Tier: <tier> | Language: <language>
```

For `platform=tiktok`, also list all produced clips:
```
Output clips (TikTok):
  <filename>_s95_tk_01.mp4 (<duration>, <size>) — [Score: 95] "<description>"
  <filename>_s88_tk_02.mp4 (<duration>, <size>) — [Score: 88] "<description>"
  ...
```

For `platform=both`, show both summaries.

## Error Handling

- Source video not found → report error, stop
- ffmpeg not available → report install instructions, stop
- No audio tracks → report error, stop
- faster-whisper not available → warn, fall back to Minimal tier
- No moments detected above threshold → report, suggest lowering threshold
- Segment processing fails → skip segment, continue, report at end
- Disk space < 2GB → warn before starting

## Progress Reporting

Report at each stage. Step counts depend on whether the transcript-analyzer runs.

**When transcript-analyzer runs — analyze mode (5 steps):**

    [1/5] Analyzing audio tracks and transcribing (faster-whisper small)...
    [2/5] Analyzing transcript for content signals...
    [3/5] Computing final scores and merging moments...
    [4/5] Presenting analysis for review... (N moments found, M above threshold)
    [5/5] Assembling edit... segment N/M complete

**When transcript-analyzer runs — auto mode (4 steps):**

    [1/4] Analyzing audio tracks and transcribing (faster-whisper small)...
    [2/4] Analyzing transcript for content signals...
    [3/4] Computing final scores and merging moments...
    [4/4] Assembling edit... segment N/M complete

**When transcript-analyzer is skipped — analyze mode (3 steps):**

    [1/3] Analyzing audio tracks and scoring moments...
    [2/3] Presenting analysis for review... (N moments found, M above threshold)
    [3/3] Assembling edit... segment N/M complete

**When transcript-analyzer is skipped — auto mode (2 steps):**

    [1/2] Analyzing audio tracks and scoring moments...
    [2/2] Assembling edit... segment N/M complete
