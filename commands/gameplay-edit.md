---
description: Edit gameplay videos into highlight reels or short-form clips
argument-hint: <video_path> [--style clean|funny] [--platform youtube|tiktok|both] [--language hu] [--duration 5m] [--mode analyze|auto] [--output path]
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(whisper:*), Bash(python:*), Bash(python3:*), Bash(ls:*), Bash(mkdir:*), Bash(rm:*), Bash(cat:*), Bash(head:*), Bash(date:*), Bash(stat:*), Bash(mktemp:*), Bash(df:*)
---

# Gameplay Video Editor

You are editing a gameplay video into a highlight reel or short-form clips.

## Parse Arguments

From the user's input, extract:
- **video_path** (required): path to the source video file
- **style** (default: `clean`): one of `clean`, `funny`
- **platform** (default: `youtube`): one of `youtube`, `tiktok`, `both`
- **language** (default: `hu`): Whisper language code
- **duration** (default: `5m` for youtube, individual clips for tiktok): target duration like `5m`, `10m`, `60s`, `3m`
- **mode** (default: `analyze`): one of `analyze`, `auto`
- **output** (default: `Outs/<source_basename>_edit.mp4` relative to source video's parent directory)

## Validate

1. Check the video file exists: `ls -la "<video_path>"`
2. Check ffmpeg is available: `ffmpeg -version`
3. Check video has audio tracks: `ffprobe -v quiet -print_format json -show_streams "<video_path>"` — if no stream has `codec_type == "audio"`, report "Error: No audio tracks found in source video" and stop
4. Read the style file from this plugin's `styles/<style>.md` — parse the YAML frontmatter for parameters
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

Run `whisper --help` quietly. If it works, report: `"Running with Whisper (Full tier) — transcription + visual analysis + laughter detection + captions"`
If not: `"Whisper not available — using volume-based analysis only (Minimal tier). Run /gameplay-setup for better results."`

## Single Prompt (analyze mode only)

If mode is `analyze` AND not all parameters were provided via CLI flags, present one prompt:

```
Quick setup:
- Language? (default: hu)
- Style? clean / funny (default: clean)
- Platform? youtube / tiktok / both (default: youtube)
- Duration? (default: 5m for youtube, individual clips for tiktok)
```

Parse the user's response flexibly:
- They can answer all, some, or none (enter = all defaults)
- Match answers to fields by keyword (e.g., "funny" → style, "tiktok" → platform, "en" → language, "10m" → duration)
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
- style
- target_duration (converted to seconds)
- tmp_dir: $TMP

The agent will return a JSON result with moments, timing data, word frequency alerts, and (if style=funny) suggested edits.

### Word Counter Prompt

If the analyzer returned `word_frequency_alerts` (any word appearing 10+ times):

For each alert, ask:
```
Word frequency alert:
  "<word>" appears <count> times. Add a running counter overlay? [y/n]
```

In `auto` mode, skip this prompt — do not add word counters automatically.

### Present Plan (analyze mode)

Present the plan to the user. Include:

**For clean style:**
```
=== Highlight Plan ===
Source: recording.mkv (3h 24m)
Target: 5m highlight reel | Style: clean | Platform: youtube

Selected moments (14):
#1  [Score: 95] 00:12:28 → 00:13:07 (39s) — "Mass laughter + 3 voices overlapping"
#2  [Score: 88] 00:34:12 → 00:34:55 (43s) — "Dramatic comeback after silence"
...

Excluded moments (23):
#15 [Score: 58] 01:05:18 → 01:05:42 (24s) — Below threshold (61.0)
...
```

**For funny style (includes edit suggestions):**
```
=== Highlight Plan ===
Source: recording.mkv (3h 24m)
Target: 5m highlight reel | Style: funny | Platform: youtube

Selected moments (14):
#1  [Score: 95] 00:12:28 → 00:13:07 (39s) — "Mass laughter + 3 voices overlapping"
    Suggested edits:
    ✎ Text pop-up: "NO WAY!" at 00:12:35
    ✎ SFX: dramatic_boom at 00:12:33
    ✎ Slow-mo: 00:12:33 → 00:12:36
    ✎ Replay: 00:12:33 → 00:12:36
    [ ] Keep all  [ ] Remove: ___

#2  [Score: 88] 00:34:12 → 00:34:55 (43s) — "Dramatic comeback after silence"
    Suggested edits:
    ✎ SFX: sad_violin at 00:34:15
    [ ] Keep all  [ ] Remove: ___
...
```

Record user review start time. Wait for user response. They may:
- Approve: "looks good", "go", "just do it" → proceed to assembly
- Modify: "remove #2", "add the part around 01:30:00", "make it 3 min" → update plan, re-present
- Change style/platform: "swap to funny", "also do tiktok" → re-read preset, re-present
- Modify edits: "remove slow-mo from #1", "no SFX on #3" → update suggestions, re-present

Record user review end time: `user_review_ms`.

### Auto Mode

Skip the plan presentation and word counter prompt. Select moments automatically and proceed to assembly.

### Assembly

Dispatch the **edit-assembler** agent with:
- source_video_path
- The approved/selected moment list (with suggested_edits if style=funny)
- The style preset parameters (from YAML frontmatter)
- platform: `youtube` or `tiktok`
- output_path
- transcript_path (if available)
- word_timestamps_path (if available)
- word_counter config (if user approved)
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
Word counter: "<word>" x<count>  (only if word counter was active)

Timing:
  Audio probe:         <audio_probe time>
  Whisper:             <whisper time>
  Visual analysis:     <visual_analysis time>
  Scoring:             <scoring time>
  User review:         <user_review time>  (only in analyze mode)
  Segment extraction:  <extraction time>
  Audio processing:    <audio_processing time>
  Transitions/FX:      <transitions_fx time>
  Captions:            <captions time>  (only for tiktok)
  Crop & export:       <crop_export time>
  ─────────────────────────
  Total:               <total time>

Style: <style> | Platform: <platform> | Tier: <tier> | Language: <language>
```

For `platform=tiktok`, also list all produced clips:
```
Output clips (TikTok):
  <filename>_edit_tk_01.mp4 (<duration>, <size>) — "<description>"
  <filename>_edit_tk_02.mp4 (<duration>, <size>) — "<description>"
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
- SFX file missing → skip that SFX, warn in summary
- Visual analysis fails → skip visual signals, reweight remaining, warn
- Disk space < 2GB → warn before starting

## Progress Reporting

Report at each stage:
```
[1/6] Probing audio tracks...
[2/6] Running Whisper transcription... (estimated ~X min for Y hour video)
[3/6] Analyzing visual motion...
[4/6] Scoring moments... found N candidates
[5/6] Presenting plan for approval...
[6/6] Assembling edit... segment N/M complete
```
