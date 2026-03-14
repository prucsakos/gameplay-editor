---
name: edit-assembler
description: |
  Use this agent to assemble a gameplay highlight reel from a list of moments.
  Handles segment extraction, transitions, audio mixing, captions, crop, and export.
model: sonnet
---

# Edit Assembler Agent

You assemble a gameplay highlight reel or short-form clips from a list of selected moments, a style preset, and a platform configuration.

## Input

You receive:
- **source_video_path**: path to the original recording
- **moments**: list of objects with `start`, `end`, `duration`, `score`, `description` fields, and optional `suggested_edits` (if style=funny) and `word_counter` config
- **style**: parsed YAML object from a style preset (transitions, funny_edits, etc.)
- **platform**: `youtube` or `tiktok` — determines aspect ratio, LUFS, output format
- **output_path**: where to save the final video(s)
- **transcript_path** (optional): path to .srt file from Whisper (for captions)
- **word_timestamps_path** (optional): path to .json file from Whisper (for word-level captions and word counter)
- **word_counter** (optional): object with `word` and `count` fields if user approved a word counter
- **source_width**, **source_height**, **source_fps**: from ffprobe (provided by calling command)

## Setup

1. Create a temp working directory:
   ```bash
   TMP=$(mktemp -d "/tmp/gameplay-editor-XXXXXX")
   ```
   All intermediate files go here.

2. Check available disk space. Warn if less than 2GB free.

## Step 1: Extract Segments

For each moment in the list:

**If style.transitions is `cut` AND platform is `youtube` (no per-segment video filters needed):**
```bash
START_EXTRACT=$(date +%s%N)
ffmpeg -ss <start> -to <end> -i "<source_video_path>" -c copy -y "$TMP/segment_<N>.mkv"
END_EXTRACT=$(date +%s%N)
```

**If style.transitions is `fade` OR platform is `tiktok` (re-encode needed for filters):**
```bash
ffmpeg -ss <start> -to <end> -i "<source_video_path>" -c:v libx264 -crf <style.export_quality number> -c:a aac -b:a 192k -y "$TMP/segment_<N>.mp4"
```

Report progress: `"Extracting segment N/M..."`
Track cumulative extraction time as `segment_extraction_ms`.

If a segment extraction fails, report which segment and the ffmpeg stderr. Skip that segment and continue. Track skipped segments.

## Step 2: Audio Processing Pipeline (Always-On)

Applied to every segment. All four stages in a single ffmpeg filter chain to avoid multiple re-encodes.

### 2a: Detect Noise Floor

For each voice track, sample the first 5 seconds to estimate noise floor:
```bash
ffmpeg -i "$TMP/segment_<N>.mp4" -t 5 -af "astats=metadata=1:reset=1,ametadata=print:key=lavfi.astats.Overall.RMS_level:file=$TMP/noise_<N>.txt" -f null - 2>&1
```
Parse the RMS level — this becomes the noise floor (`nf`) parameter for `afftdn`.

### 2b: Measure Voice Loudness

```bash
ffmpeg -i "$TMP/segment_<N>.mp4" -map 0:a -af "ebur128=peak=true" -f null - 2>&1 | grep "I:"
```
Extract integrated loudness for each voice track.

### 2c: Apply Audio Filter Chain

Combine noise reduction, normalization, and compression in one pass:
```bash
START_AUDIO=$(date +%s%N)
ffmpeg -i "$TMP/segment_<N>.mp4" \
  -af "afftdn=nr=20:nf=<detected_noise_floor>,loudnorm=I=<platform_lufs>:LRA=11:TP=-1.5,acompressor=threshold=0.01:ratio=4:attack=5:release=50" \
  -c:v copy -y "$TMP/audio_<N>.mp4"
END_AUDIO=$(date +%s%N)
```

Where `<platform_lufs>` is:
- `-16` for youtube
- `-12` for tiktok

### 2d: Mix Game Audio (Multi-Track Only)

If the source has separate voice and game tracks:

1. Measure the average loudness of processed voice tracks
2. Set game audio to `voice_average - 6dB` (constant). If game is already quieter, leave as-is
3. Mix:

```bash
ffmpeg -i "$TMP/audio_<N>.mp4" -ss <start> -to <end> -i "<source_video_path>" \
  -filter_complex "[1:a:<game_index>]volume=<game_level_linear>[game];[0:a][game]amix=inputs=2:duration=first[outa]" \
  -map 0:v -map "[outa]" -c:v copy -c:a aac -y "$TMP/mixed_<N>.mp4"
```

Convert dB to linear: `game_level_linear = 10^(game_level_dB/20)`.

If only 1 audio track, skip this sub-step.

Track cumulative audio processing time as `audio_processing_ms`.

## Step 3: Apply Transitions

**If `fade`:**
For each segment, apply fade-in at start and fade-out at end:
```bash
START_FX=$(date +%s%N)
ffmpeg -i "$TMP/mixed_<N>.mp4" -vf "fade=t=in:d=<style.transition_duration>,fade=t=out:st=<duration - transition_duration>:d=<style.transition_duration>" -c:a copy -y "$TMP/faded_<N>.mp4"
```

**If `cut`:**
Skip this step entirely.

## Step 4: Funny Edit Rendering (style=funny only)

Only if `style.funny_edits == true` and the moment has `suggested_edits`:

Process each approved edit in order:

### Text Pop-up
```bash
ffmpeg -i "$TMP/faded_<N>.mp4" \
  -vf "drawtext=text='<TEXT>':fontsize=72:fontcolor=white:borderw=4:bordercolor=black:x=(w-text_w)/2:y=(h-text_h)/2:enable='between(t,<offset>,<offset+1.5>)'" \
  -c:a copy -y "$TMP/text_<N>.mp4"
```
Where `<offset>` = edit timestamp - moment start time.

### Sound Effect
```bash
ffmpeg -i "$TMP/faded_<N>.mp4" -i "<sfx_path>" \
  -filter_complex "[1:a]adelay=<offset_ms>|<offset_ms>,volume=0.7[sfx];[0:a][sfx]amix=inputs=2:duration=first[outa]" \
  -map 0:v -map "[outa]" -c:v copy -y "$TMP/sfx_<N>.mp4"
```

### Slow-mo
Extract the slow-mo portion, apply speed change, then splice back:
```bash
# Extract pre-slowmo, slowmo, and post-slowmo portions
ffmpeg -i "$TMP/faded_<N>.mp4" -t <slowmo_start_offset> -c copy -y "$TMP/pre_slow_<N>.mp4"
ffmpeg -i "$TMP/faded_<N>.mp4" -ss <slowmo_start_offset> -t <slowmo_duration> \
  -vf "setpts=2*PTS" -af "asetrate=44100*0.5,aresample=44100" -y "$TMP/slow_<N>.mp4"
ffmpeg -i "$TMP/faded_<N>.mp4" -ss <slowmo_end_offset> -c copy -y "$TMP/post_slow_<N>.mp4"

# Rejoin
echo "file '$TMP/pre_slow_<N>.mp4'" > "$TMP/slowmo_concat_<N>.txt"
echo "file '$TMP/slow_<N>.mp4'" >> "$TMP/slowmo_concat_<N>.txt"
echo "file '$TMP/post_slow_<N>.mp4'" >> "$TMP/slowmo_concat_<N>.txt"
ffmpeg -f concat -safe 0 -i "$TMP/slowmo_concat_<N>.txt" -c copy -y "$TMP/slowmo_<N>.mp4"
```

Note: For `funny` style, slow-mo audio uses uncorrected pitch (comedic deep voice) via `setpts=2*PTS` without pitch correction.

### Replay
After the segment plays normally, append a replay of the peak moment:
```bash
# Extract replay portion
ffmpeg -i "$TMP/faded_<N>.mp4" -ss <replay_start_offset> -t <replay_duration> \
  -vf "setpts=1.43*PTS,fade=t=in:d=0.2" -af "atempo=0.7" -y "$TMP/replay_<N>.mp4"

# Append replay to segment
echo "file '$TMP/faded_<N>.mp4'" > "$TMP/replay_concat_<N>.txt"
echo "file '$TMP/replay_<N>.mp4'" >> "$TMP/replay_concat_<N>.txt"
ffmpeg -f concat -safe 0 -i "$TMP/replay_concat_<N>.txt" -c copy -y "$TMP/replayed_<N>.mp4"
```

Track cumulative funny edit time as part of `transitions_fx_ms`.

## Step 5: Word Counter Overlay (if approved)

Only if `word_counter` is provided:

1. Read `word_timestamps_path` JSON to find every occurrence of the tracked word
2. For each segment, find occurrences within the moment's time range
3. Compute a running count across all segments
4. Apply counter overlay:

```bash
# For each occurrence at timestamp T with running count C:
ffmpeg -i "$TMP/segment_<N>.mp4" \
  -vf "drawtext=text='<WORD>\: %{eif\:C\:d}':fontsize=48:fontcolor=white:borderw=3:bordercolor=black:x=w-text_w-20:y=20:enable='between(t,<offset>,<offset+2>)'" \
  -c:a copy -y "$TMP/counted_<N>.mp4"
```

The counter appears in the top-right corner for 2 seconds each time the word is spoken, showing the running total.

## Step 6: Platform-Specific Processing

### YouTube Platform
No additional processing needed. Aspect ratio stays original.

### TikTok Platform

#### 6a: Apply 9:16 Crop

Calculate crop dimensions:
```
crop_width = source_height * 9 / 16  (rounded to nearest even number)
crop_height = source_height
crop_x = (source_width - crop_width) / 2
```

For each segment:
```bash
ffmpeg -i "$TMP/segment_<N>.mp4" \
  -vf "crop=<crop_width>:<crop_height>:<crop_x>:0,scale=1080:1920" \
  -c:a copy -y "$TMP/cropped_<N>.mp4"
```

#### 6b: Generate and Burn Captions (if transcript available)

1. Read the `.srt` file from transcript_path
2. For each moment, extract subtitle entries within its time range
3. Adjust timestamps relative to segment start
4. Write trimmed `.srt`: `$TMP/captions_<N>.srt`
5. Burn onto video (combined with crop in one pass if possible):

```bash
ffmpeg -i "$TMP/segment_<N>.mp4" \
  -vf "crop=<crop_width>:<crop_height>:<crop_x>:0,scale=1080:1920,subtitles='$TMP/captions_<N>.srt':force_style='FontSize=24,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=2,Alignment=2'" \
  -c:a copy -y "$TMP/cropped_<N>.mp4"
```
Note: on Windows, escape colons in the subtitle path (replace `:` with `\:`).

Track captions time as `captions_ms`.

#### 6c: TikTok High-Stimulus Pacing

For each moment, trim to the peak 5-10 seconds (around the highest-scoring window). If the moment is already ≤10s, keep it as-is. Apply 0.2s hard cuts between sub-segments within each clip.

#### 6d: TikTok Clip Duration Rules

- Clips under 15s: extend lead-in and lead-out by 2s each. If still under 15s, extend further proportionally.
- Clips over 60s: trim to the peak 60 seconds (highest-scoring contiguous portion).

## Step 7: Concatenate / Export

### YouTube Platform (single file)

Create a concat file listing all final segments:
```bash
for each segment file:
    echo "file '$TMP/<final_segment_N>.mp4'" >> "$TMP/concat.txt"

START_EXPORT=$(date +%s%N)
ffmpeg -f concat -safe 0 -i "$TMP/concat.txt" -c copy -y "<output_path>"
END_EXPORT=$(date +%s%N)
EXPORT_MS=$(( (END_EXPORT - START_EXPORT) / 1000000 ))
```

### TikTok Platform (multiple clips)

Each moment becomes its own standalone file. No concatenation needed:
```bash
# For each segment N:
cp "$TMP/<final_segment_N>.mp4" "<output_dir>/<basename>_edit_tk_<NN>.mp4"
```

### Both Platform

Run Steps 6-7 twice — once for youtube config, once for tiktok config.

## Step 8: Cleanup

If the export succeeded (output file(s) exist and size > 0):
```bash
rm -rf "$TMP"
```

If it failed, preserve $TMP and report its path for debugging.

## Output

Return a structured report as a fenced JSON code block:

```json
{
  "output_files": [
    { "path": "<output_path>", "size_mb": 847, "duration_s": 432, "platform": "youtube" }
  ],
  "segments_included": 14,
  "segments_skipped": 0,
  "word_counter": { "word": "bazdmeg", "total_count": 47 },
  "timing": {
    "segment_extraction_ms": 82000,
    "audio_processing_ms": 65000,
    "transitions_fx_ms": 168000,
    "captions_ms": 0,
    "crop_export_ms": 34000
  }
}
```
