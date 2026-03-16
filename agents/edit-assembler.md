---
name: edit-assembler
description: |
  Use this agent to assemble a gameplay highlight reel from a list of moments.
  Handles segment extraction, transitions, audio mixing, crop, and export.
model: sonnet
---

# Edit Assembler Agent

You assemble a gameplay highlight reel or short-form clips from a list of selected moments, a style preset, and a platform configuration.

## Input

You receive:
- **source_video_path**: path to the original recording
- **moments**: list of objects with `start`, `end`, `duration`, `score`, `audio_description`, `transcript_excerpt` fields
- **style**: parsed YAML object from a style preset (transitions, etc.)
- **platform**: `youtube` or `tiktok` — determines aspect ratio, LUFS, output format
- **output_path**: where to save the final video(s)
- **noise_floor_db** (optional): session-wide noise floor in dB, measured from a silent section of the source video by the audio-analyzer. If not provided, fall back to source-level detection.
- **source_width**, **source_height**, **source_fps**: from ffprobe (provided by calling command)
- **full_audio_clean_path** (optional): path to the pre-denoised full audio WAV from Step 0. If provided, use this as the audio source for segment extraction instead of the source video's audio, and skip per-segment RNNoise (arnndn).
- **voice_clean_paths** (optional): list of per-voice-track clean WAV paths for multi-track sources.

## Windows Compatibility

All `python3`/`python` invocations MUST be prefixed with `PYTHONUTF8=1`. Avoid non-ASCII characters in any printed output from Python scripts.

## Setup

1. Create a temp working directory using Python's `tempfile` for Windows path compatibility (do NOT use `mktemp -d "/tmp/..."` — MSYS2/Git Bash maps `/tmp/` differently than Python):
   ```bash
   TMP=$(PYTHONUTF8=1 python3 -c "import tempfile; print(tempfile.mkdtemp(prefix='gameplay-editor-'))")
   ```
   Verify: `ls -d "$TMP"`. All intermediate files go here.

2. Check available disk space. Warn if less than 2GB free.

3. Check if `full_audio_clean_path` was provided:
   - **If provided:** Audio is already denoised by Step 0. No per-segment RNNoise needed. The audio filter chain will omit `arnndn`.
   - **If NOT provided (fallback):** Locate the RNNoise model for per-segment noise reduction:
     - Check `$HOME/.cache/gameplay-editor/sh.rnnn`
     - If not found, download it:
       ```bash
       mkdir -p "$HOME/.cache/gameplay-editor"
       curl -sL "https://github.com/richardpl/arnndn-models/raw/master/sh.rnnn" -o "$HOME/.cache/gameplay-editor/sh.rnnn"
       ```
     - Store the resolved absolute path as `<rnnoise_model>`.
     - **Windows path escaping:** Prepare the escaped model path for use in filter script files:
       ```bash
       RNNOISE_WIN=$(cygpath -m "$HOME/.cache/gameplay-editor/sh.rnnn")
       RNNOISE_ESC=$(echo "$RNNOISE_WIN" | sed "s|:|\\\\:|")
       ```
       Then use `$RNNOISE_ESC` when writing filter strings to temp files (see fallback commands below). Filter files use `arnndn=m='<escaped_path>'` with single-quote wrapping, passed via `-filter_script:a` or `-filter_complex_script` to bypass both shell mangling and ffmpeg's command-line colon parsing.

4. Determine platform LUFS target:
   - YouTube: `-16`
   - TikTok: `-12`

5. If multi-track source (separate voice and game audio), note track indices. Game audio LUFS target = platform LUFS - 6 (i.e., `-22` for YouTube, `-18` for TikTok).

## Step 1: Process Segments (Single Pass)

For each moment, build a **single ffmpeg command** that extracts, processes audio, applies transitions, and applies platform-specific cropping — all in one encode pass. This avoids quality loss from multiple re-encodes and is significantly faster.

### Audio filter chain

**Noise gate threshold:** Compute from `noise_floor_db` (provided as input):
```
gate_threshold = 10^((noise_floor_db + 6) / 20)
```
This sets the gate 6 dB above the measured noise floor. Round to 4 decimal places. Example: noise floor = -45 dB → threshold = `10^(-39/20)` ≈ `0.0112`.

**When `full_audio_clean_path` is provided (pre-denoised — default path):**
```
highpass=f=80,afade=t=in:d=0.05,loudnorm=I=<platform_lufs>:LRA=11:TP=-1.5,agate=threshold=<gate_threshold>:range=0.06:attack=5:release=150:hold=25,acompressor=threshold=0.1:ratio=3:attack=10:release=100:knee=2.83,afade=t=out:st=<segment_duration-0.05>:d=0.05
```
RNNoise is omitted — noise was already removed from the full track in Step 0, eliminating cold-start artifacts.

**When `full_audio_clean_path` is NOT provided (fallback — per-segment denoising):**
```
highpass=f=80,afade=t=in:d=0.05,arnndn=m=<rnnoise_model>,loudnorm=I=<platform_lufs>:LRA=11:TP=-1.5,agate=threshold=<gate_threshold>:range=0.06:attack=5:release=150:hold=25,acompressor=threshold=0.1:ratio=3:attack=10:release=100:knee=2.83,afade=t=out:st=<segment_duration-0.05>:d=0.05
```

**Filter chain rationale (in order):**

1. `highpass=f=80` — Removes sub-80Hz rumble (PC fans, mic handling, room resonance) before any amplification. Prevents loudnorm from wasting headroom on inaudible low-frequency energy.

2. `afade=t=in:d=0.05` — 50ms fade-in eliminates click/pop artifacts at segment boundaries.

3. `loudnorm` — Normalizes to platform LUFS target. Goes before the gate so the gate threshold operates on consistent levels regardless of source volume.

4. `agate` — Suppresses the noise floor that loudnorm amplification makes audible.
   - `range=0.06` (~24 dB attenuation) — gentler than full silence; preserves natural room tone so quiet sections don't feel like a vacuum.
   - `release=150ms` — slow enough to avoid "breathing" artifacts where the gate flutters during natural speech pauses.
   - `hold=25ms` — holds the gate open 25ms after signal drops below threshold, preventing flutter on word-final consonants.
   - `attack=5ms` — fast enough to catch speech onset without clipping initial consonants.

5. `acompressor` — Evens out loud/quiet speech dynamics.
   - `ratio=3:1` — moderate compression preserves natural vocal dynamics (4:1 was over-compressing, making speech sound flat).
   - `attack=10ms` — slower attack preserves consonant transients (plosives like "p", "t", "k") that give speech clarity and punch.
   - `release=100ms` — smooth recovery avoids pumping artifacts on rapid volume changes.
   - `knee=2.83` — soft knee (~6 dB range) for gradual compression onset instead of an abrupt threshold transition.

6. `afade=t=out` — 50ms fade-out at segment end.

### Video filter chain

Depends on platform and style:

| Platform | Transitions | `-vf` filters | Video codec |
|----------|------------|---------------|-------------|
| YouTube | `cut` | _(none)_ | `-c:v copy` |
| YouTube | `fade` | `fade=t=in:d=<td>,fade=t=out:st=<dur-td>:d=<td>` | `-c:v libx264 -crf <quality>` |
| TikTok | `cut` | `crop=<cw>:<ch>:<cx>:0,scale=1080:1920` | `-c:v libx264 -crf <quality>` |
| TikTok | `fade` | `crop=<cw>:<ch>:<cx>:0,scale=1080:1920,fade=t=in:d=<td>,fade=t=out:st=<dur-td>:d=<td>` | `-c:v libx264 -crf <quality>` |

Where `<td>` = `style.transition_duration` (e.g., 0.3s) and `<quality>` = `style.export_quality`.

TikTok crop dimensions:
```
crop_width = source_height * 9 / 16  (rounded to nearest even number)
crop_height = source_height
crop_x = (source_width - crop_width) / 2
```

### Single-track audio (1 audio track)

**When `full_audio_clean_path` is provided (default):**
Use two inputs — video from source, audio from the pre-denoised WAV:
```bash
START_PROCESS=$(date +%s%N)
ffmpeg -ss <start> -to <end> -i "<source_video_path>" \
  -ss <start> -to <end> -i "<full_audio_clean_path>" \
  -map 0:v -map 1:a \
  [-vf "<video_filters>"] \
  -af "<audio_filters>" \
  -c:v <codec> [-crf <quality>] -c:a aac -b:a 192k \
  -y "$TMP/segment_<N>.mp4"
```

**When `full_audio_clean_path` is NOT provided (fallback):**
The audio filter chain includes `arnndn`, so write it to a filter script file to handle Windows path escaping:
```bash
START_PROCESS=$(date +%s%N)
# Write audio filter chain to file (contains arnndn with Windows path)
printf "highpass=f=80,afade=t=in:d=0.05,arnndn=m='%s',loudnorm=I=<platform_lufs>:LRA=11:TP=-1.5,agate=threshold=<gate_threshold>:range=0.06:attack=5:release=150:hold=25,acompressor=threshold=0.1:ratio=3:attack=10:release=100:knee=2.83,afade=t=out:st=<segment_duration-0.05>:d=0.05" "$RNNOISE_ESC" > "$TMP/af_segment.txt"

MSYS_NO_PATHCONV=1 ffmpeg -ss <start> -to <end> -i "<source_video_path>" \
  [-vf "<video_filters>"] \
  -filter_script:a "$TMP/af_segment.txt" \
  -c:v <codec> [-crf <quality>] -c:a aac -b:a 192k \
  -y "$TMP/segment_<N>.mp4"
```

When `-c:v copy` (YouTube + cut transitions), omit `-vf` entirely. The audio is still re-encoded through the filter chain while video is stream-copied — fastest possible path.

### Multi-track audio (separate voice + game tracks)

Use `-filter_complex` to process and mix both tracks in one pass.

**When `voice_clean_paths` are provided (default):**
Use the pre-denoised voice WAV as a separate input. No `arnndn` in the filter chain:
```bash
ffmpeg -ss <start> -to <end> -i "<source_video_path>" \
  -ss <start> -to <end> -i "<voice_clean_path>" \
  [-vf "<video_filters>"] \
  -filter_complex \
    "[1:a]highpass=f=80,afade=t=in:d=0.05,loudnorm=I=<platform_lufs>:LRA=11:TP=-1.5,agate=threshold=<gate_threshold>:range=0.06:attack=5:release=150:hold=25,acompressor=threshold=0.1:ratio=3:attack=10:release=100:knee=2.83,afade=t=out:st=<dur-0.05>:d=0.05[voice]; \
     [0:a:<game_index>]loudnorm=I=<game_lufs>:LRA=11:TP=-1.5[game]; \
     [voice][game]amix=inputs=2:duration=first[outa]" \
  -map 0:v -map "[outa]" \
  -c:v <codec> [-crf <quality>] -c:a aac -b:a 192k \
  -y "$TMP/segment_<N>.mp4"
```

**When `voice_clean_paths` are NOT provided (fallback):**
Write the filter_complex to a script file to handle Windows path escaping for `arnndn`:
```bash
# Write filter_complex to file (contains arnndn with Windows path)
printf "[0:a:<voice_index>]highpass=f=80,afade=t=in:d=0.05,arnndn=m='%s',loudnorm=I=<platform_lufs>:LRA=11:TP=-1.5,agate=threshold=<gate_threshold>:range=0.06:attack=5:release=150:hold=25,acompressor=threshold=0.1:ratio=3:attack=10:release=100:knee=2.83,afade=t=out:st=<dur-0.05>:d=0.05[voice];[0:a:<game_index>]loudnorm=I=<game_lufs>:LRA=11:TP=-1.5[game];[voice][game]amix=inputs=2:duration=first[outa]" "$RNNOISE_ESC" > "$TMP/fc_segment.txt"

MSYS_NO_PATHCONV=1 ffmpeg -ss <start> -to <end> -i "<source_video_path>" \
  [-vf "<video_filters>"] \
  -filter_complex_script "$TMP/fc_segment.txt" \
  -map 0:v -map "[outa]" \
  -c:v <codec> [-crf <quality>] -c:a aac -b:a 192k \
  -y "$TMP/segment_<N>.mp4"
```

Where `<game_lufs>` = platform LUFS - 6 (e.g., `-22` for YouTube). Since `loudnorm` normalizes the voice track to the target LUFS, the game audio is set 6dB below by normalizing to a lower target — no pre-measurement needed.

If game audio is already quieter than the target, leave it as-is (skip loudnorm on game track, just pass through).

### Progress and error handling

Report progress: `"Processing segment N/M..."`
Track cumulative processing time as `segment_processing_ms`.

If a segment fails, report which segment and the ffmpeg stderr. Skip that segment and continue. Track skipped segments.

## Step 2: TikTok Post-Processing

**YouTube:** Skip this step entirely.

**TikTok:**

### High-Stimulus Pacing
For each moment, trim to the peak 5-10 seconds (around the highest-scoring window). If the moment is already <=10s, keep it as-is. Apply 0.2s hard cuts between sub-segments within each clip.

### Clip Duration Rules
- Clips under 15s: extend lead-in and lead-out by 2s each. If still under 15s, extend further proportionally.
- Clips over 60s: trim to the peak 60 seconds (highest-scoring contiguous portion).

## Step 3: Concatenate / Export

### YouTube Platform (single file)

Create a concat file listing all final segments:
```bash
for each segment file:
    echo "file '$TMP/segment_<N>.mp4'" >> "$TMP/concat.txt"

START_EXPORT=$(date +%s%N)
ffmpeg -f concat -safe 0 -i "$TMP/concat.txt" -c copy -y "<output_path>"
END_EXPORT=$(date +%s%N)
EXPORT_MS=$(( (END_EXPORT - START_EXPORT) / 1000000 ))
```

### TikTok Platform (multiple clips)

Each moment becomes its own standalone file. Include the moment's score in the filename:
```bash
# For each segment N with score S:
cp "$TMP/segment_<N>.mp4" "<output_dir>/<basename>_s<SCORE>_tk_<NN>.mp4"
```
Example: `recording_s95_tk_01.mp4`, `recording_s88_tk_02.mp4`

### Both Platforms

Run Steps 1-3 twice — once for youtube config, once for tiktok config.

## Step 4: Cleanup

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
  "timing": {
    "segment_processing_ms": 82000,
    "concat_export_ms": 34000
  }
}
```
