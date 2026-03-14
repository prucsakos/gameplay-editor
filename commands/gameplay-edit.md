---
description: Edit gameplay videos into highlight reels or short-form clips
argument-hint: <video_path> [--style clean|polished|shortform] [--duration 5m] [--mode analyze|auto|manual] [--timestamps "HH:MM:SS-HH:MM:SS,..."] [--output path]
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(whisper:*), Bash(python:*), Bash(python3:*), Bash(ls:*), Bash(mkdir:*), Bash(rm:*), Bash(cat:*), Bash(head:*)
---

# Gameplay Video Editor

You are editing a gameplay video into a highlight reel or short-form clip.

## Parse Arguments

From the user's input, extract:
- **video_path** (required): path to the source video file
- **style** (default: `clean`): one of `clean`, `polished`, `shortform`
- **duration** (default: `5m`, shortform default: `60s`): target duration like `5m`, `10m`, `60s`, `3m`
- **mode** (default: `analyze`): one of `analyze`, `auto`, `manual`
- **timestamps** (manual mode only): comma-separated `start-end` pairs like `"01:23:00-01:24:30,02:10:00-02:11:15"`
- **output** (default: `Outs/<source_basename>_edit.mp4` relative to source video's parent directory)

## Validate

1. Check the video file exists: `ls -la "<video_path>"`
2. Check ffmpeg is available: `ffmpeg -version`
3. Check video has audio tracks: `ffprobe -v quiet -print_format json -show_streams "<video_path>"` — if no stream has `codec_type == "audio"`, report "Error: No audio tracks found in source video" and stop
4. If mode is `manual`, check that `--timestamps` was provided
4. Read the style file from this plugin's `styles/<style>.md` — parse the YAML frontmatter for parameters
5. Create the output directory if it doesn't exist

## Check Whisper Availability

Run `whisper --help` quietly. If it works, report: "Running with Whisper (Full tier) — transcription + laughter detection + captions"
If not: "Whisper not available — using volume-based analysis only (Minimal tier). Run /gameplay-setup for better results."

## Execute Based on Mode

### If mode is `manual`:
Skip analysis. Parse the `--timestamps` argument into a moment list. Each pair becomes a moment with score 100 and description "Manual selection". Proceed directly to the edit-assembler agent.

### If mode is `analyze` or `auto`:
1. Dispatch the **audio-analyzer** agent with the video path. It will:
   - Probe audio tracks
   - Run Whisper transcription (if available)
   - Score all 5-second windows
   - Return a ranked list of moments with timestamps, scores, descriptions, and transcript excerpts

2. **If mode is `analyze`** (default):
   Present the plan document to the user (see spec for format). Include:
   - Selected moments (enough to fill target duration) with scores and descriptions
   - Excluded moments (below threshold)
   - Assembly notes (speed ramps, text overlays, audio levels)

   Wait for user response. They may:
   - Approve: "looks good", "go", "just do it" → proceed to assembly
   - Modify: "remove #2", "add the part around 01:30:00", "make it 3 min" → update plan, re-present
   - Change style: "swap to shortform" → re-read style preset, re-present

3. **If mode is `auto`**:
   Skip the approval step. Select moments automatically and proceed to assembly.

### Assembly:
Dispatch the **edit-assembler** agent with:
- The approved/selected moment list (timestamps, durations)
- The style preset parameters (from YAML frontmatter)
- The source video path
- The output path
- The Whisper transcript path (if available, for captions)

The edit-assembler will handle all ffmpeg operations and return the final output path.

## Report Result

When complete, report:
- Output file path and size
- Number of moments included
- Total duration
- Tier used (Full or Minimal)

## Error Handling

- If the source video is not found, report the error and stop
- If ffmpeg is not available, report install instructions and stop
- If no moments are detected above threshold, report this and suggest lowering duration or using manual mode
- If a segment fails to encode, skip it and continue, report which ones failed
- Estimate required disk space (source_size * 1.5) and warn if <2GB free

## Progress Reporting

Report at each stage:
```
[1/5] Probing audio tracks...
[2/5] Running Whisper transcription... (estimated ~X min)
[3/5] Scoring moments... found N candidates
[4/5] Presenting plan for approval...
[5/5] Assembling edit... segment N/M complete
```
