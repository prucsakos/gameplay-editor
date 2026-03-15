---
description: Edit gameplay videos into highlight reels or short-form clips
argument-hint: <video_path> [--platform youtube|tiktok|both] [--language hu] [--score-threshold 70] [--duration 5m] [--mode analyze|auto] [--output path]
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(whisper:*), Bash(python:*), Bash(python3:*), Bash(ls:*), Bash(mkdir:*), Bash(rm:*), Bash(cat:*), Bash(head:*), Bash(date:*), Bash(stat:*), Bash(mktemp:*), Bash(df:*)
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

## Check Whisper Availability

Run `whisper --help` quietly. If it works, report: `"Running with Whisper (Full tier) — transcription + laughter detection"`
If not: `"Whisper not available — using volume-based analysis only (Minimal tier). Run /gameplay-setup for better results."`

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

```bash
TMP=$(mktemp -d "/tmp/gameplay-editor-XXXXXX")
```

### Run Analysis

Dispatch the **audio-analyzer** agent with:
- video_path
- language
- tmp_dir: $TMP

The agent will return a JSON result with all detected moments, timing data, and noise floor.

### Present Plan (analyze mode)

Present the plan to the user. Include ALL detected moments (not just selected ones), with moments above threshold clearly marked:

```
=== Analysis Results ===
Source: recording.mkv (3h 24m)
Score threshold: 70 | Platform: youtube

All detected moments (37):
  ★ #1  [Score: 95] 00:12:28 → 00:13:07 (39s)
        Audio: Mass laughter, 3 voices overlapping, volume spike +12dB
        Screen: Explosion effect, rapid scene changes, high motion
        Transcript: "DID HE JUST— NO WAY! [laughing]"

  ★ #2  [Score: 88] 00:34:12 → 00:34:55 (43s)
        Audio: 4s silence → sudden shouting, dramatic tension-release
        Screen: Static menu → intense action sequence
        Transcript: "NEEEEM! Várj... WHAT?!"

  ★ #3  [Score: 75] 00:48:01 → 00:48:28 (27s)
        Audio: Sustained high-freq energy (laughter), moderate volume
        Screen: Normal gameplay, low motion
        Transcript: "[laughing] ez nem lehet igaz"

    #4  [Score: 62] 01:05:18 → 01:05:42 (24s)
        Audio: Brief volume spike, single speaker
        Screen: Menu navigation, minimal motion
        Transcript: "na jó, ez szar volt"

    #5  [Score: 55] 01:22:10 → 01:22:35 (25s)
        Audio: Moderate energy, no standout signals
        Screen: Standard gameplay
        Transcript: "figyelj ide..."
...

★ = above threshold (70) — will be included
Total above threshold: 14 moments (6m 12s)
Total below threshold: 23 moments
```

Each moment MUST include three description lines:
- **Audio**: What's happening in the audio (volume spikes, laughter, silence, crosstalk, etc.)
- **Screen**: What's happening visually (explosions, scene changes, motion level, static, menus)
- **Transcript**: Relevant excerpt from Whisper transcript (if available)

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
  Audio probe:         <audio_probe time>
  Whisper:             <whisper time>
  Visual analysis:     <visual_analysis time>
  Scoring:             <scoring time>
  User review:         <user_review time>  (only in analyze mode)
  Segment extraction:  <extraction time>
  Audio processing:    <audio_processing time>
  Transitions:         <transitions time>
  Crop & export:       <crop_export time>
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
- Whisper not available → warn, fall back to Minimal tier
- No moments detected above threshold → report, suggest lowering duration
- Segment extraction fails → skip segment, continue, report at end
- Visual analysis fails → skip visual signals, reweight remaining, warn
- Disk space < 2GB → warn before starting

## Progress Reporting

Report at each stage. In `analyze` mode (5 steps):
```
[1/5] Probing audio tracks...
[2/5] Running Whisper transcription... (estimated ~X min for Y hour video)
[3/5] Analyzing visual motion and scoring moments...
[4/5] Presenting analysis for review... (N moments found, M above threshold)
[5/5] Assembling edit... segment N/M complete
```

In `auto` mode, skip step 4 (4 steps total):
```
[1/4] Probing audio tracks...
[2/4] Running Whisper transcription...
[3/4] Analyzing visual motion and scoring moments... (N moments found, M above threshold)
[4/4] Assembling edit... segment N/M complete
```
