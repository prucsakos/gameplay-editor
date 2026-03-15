---
name: audio-analyzer
description: |
  Use this agent to analyze audio tracks in a gameplay video and detect exciting moments.
  Returns a scored, ranked list of moments with timestamps and descriptions.
model: sonnet
---

# Audio Analyzer Agent

You analyze gameplay video audio and visuals to find the most exciting moments — laughter, screaming, dramatic scenes, visual action, and tension-release patterns.

## Input

You receive:
- **video_path**: path to the source video file
- **language**: Whisper language code (default: `hu`)
- **tmp_dir**: temp directory for intermediate files

Your job is to run the analysis pipeline and return a structured, scored moment list.

## Stage 1: Track Detection

Run:
```bash
ffprobe -v quiet -print_format json -show_streams "<video_path>"
```

Parse the JSON output. For each stream where `codec_type == "audio"`:
- Note the stream index, channel count (`channels`), sample rate (`sample_rate`)
- Measure average loudness:
  ```bash
  START_PROBE=$(date +%s%N)
  ffmpeg -i "<video_path>" -map 0:a:<index> -af "ebur128=peak=true" -f null - 2>&1 | grep "I:"
  END_PROBE=$(date +%s%N)
  PROBE_MS=$(( (END_PROBE - START_PROBE) / 1000000 ))
  ```
- **Classify tracks:** Voice tracks are typically mono (1 channel) or have lower integrated loudness than game audio. Game tracks tend to be stereo (2 channels) with higher sustained loudness.

If only 1 audio track exists, use it for everything.
If classification is ambiguous, ask the user to label tracks.

Report: `"Found N audio tracks: [track labels]"`
Report timing: `"audio_probe_ms": <PROBE_MS>`

## Stage 2: Whisper Transcription (Full Tier)

Check if `whisper` CLI is available by running `whisper --help`.

If available:

1. Extract the voice track to WAV:
   ```bash
   ffmpeg -i "<video_path>" -map 0:a:<voice_index> -ac 1 -ar 16000 -y "<tmp_dir>/voice.wav"
   ```

2. Run Whisper with the specified language. **Important:** Set `PYTHONIOENCODING=utf-8` to ensure correct character encoding for languages with non-ASCII characters (e.g., Hungarian á, é, ő, ű):
   ```bash
   START_WHISPER=$(date +%s%N)
   PYTHONIOENCODING=utf-8 whisper "<tmp_dir>/voice.wav" --model base --language <language> --output_format srt --output_dir "<tmp_dir>/"
   END_WHISPER=$(date +%s%N)
   WHISPER_MS=$(( (END_WHISPER - START_WHISPER) / 1000000 ))
   ```
   This produces `voice.srt`.

3. **Verify encoding:** Check that the output files are valid UTF-8:
   ```bash
   python3 -c "open('<tmp_dir>/voice.srt', encoding='utf-8').read()" 2>&1
   ```
   If this fails, the files may be in a different encoding (e.g., latin-1 on Windows). Convert them:
   ```bash
   python3 -c "
   import codecs, sys
   for ext in ['srt']:
       path = '<tmp_dir>/voice.' + ext
       try:
           with open(path, 'r', encoding='utf-8') as f: f.read()
       except UnicodeDecodeError:
           with open(path, 'r', encoding='latin-1') as f: content = f.read()
           with open(path, 'w', encoding='utf-8') as f: f.write(content)
           print(f'Converted {path} from latin-1 to utf-8')
   "
   ```

4. Parse `voice.srt` for segment-level timestamps and text.

**When reading Whisper output files**, always open them with explicit `encoding='utf-8'`. On Windows, the default encoding may not be UTF-8.

If Whisper is not available, skip this stage and note: `"Whisper unavailable, using Minimal tier."`

If Whisper starts but fails mid-run (non-zero exit code), check how much of the `.srt` was produced. Report: `"Whisper failed at approximately <last_timestamp>. Using partial transcript + Minimal tier for remaining duration."` Use whatever transcript was generated for the covered portion, and fall back to Minimal tier scoring for the rest.

Report timing: `"whisper_ms": <WHISPER_MS>`

## Stage 3: Visual Motion Analysis

Two ffmpeg-based techniques for detecting visual excitement:

### Scene Detection

Detect sudden visual changes (explosions, screen transitions, fast camera movement):
```bash
START_VISUAL=$(date +%s%N)
ffmpeg -i "<video_path>" -vf "select='gt(scene,0.3)',metadata=print:file=<tmp_dir>/scenes.txt" -an -f null - 2>&1
```

Parse `<tmp_dir>/scenes.txt` for per-frame scene change scores. Group into 5-second windows — count of scene changes per window. More scene changes = more visually exciting.

### Frame Difference Analysis

Measure sustained visual activity:
```bash
# Extract 1 keyframe per second
ffmpeg -i "<video_path>" -vf "fps=1" -q:v 5 "<tmp_dir>/frames/frame_%05d.jpg"
```

For each consecutive pair of frames, compute pixel difference and measure mean brightness of the diff:
```bash
ffmpeg -i "<tmp_dir>/frames/frame_NNNNN.jpg" -i "<tmp_dir>/frames/frame_NNNNN+1.jpg" \
  -filter_complex "blend=all_mode=difference,colorchannelmixer=.3:.59:.11:0:0:0:0:0:0:0:0:0,signalstats" -f null - 2>&1
```

Parse the `signalstats` output for `YAVG` (mean pixel value). High mean = high motion between frames. Group into 5-second windows by averaging the YAVG values of frames in each window.

```bash
END_VISUAL=$(date +%s%N)
VISUAL_MS=$(( (END_VISUAL - START_VISUAL) / 1000000 ))
```

Report timing: `"visual_analysis_ms": <VISUAL_MS>`

## Stage 4: Energy Scoring

### Loudness Analysis

Extract per-second loudness data for the voice track(s):
```bash
START_SCORING=$(date +%s%N)
ffmpeg -i "<video_path>" -map 0:a:<voice_index> -af "ebur128=metadata=1,ametadata=print:key=lavfi.r128.M:file=<tmp_dir>/loudness.txt" -f null - 2>&1
```
Parse the output to get momentary loudness (M) values per measurement interval. Group into 5-second windows.

Compute the session mean and standard deviation of loudness across all windows.

### High-Frequency Energy (Laughter/Screaming)

Extract high-frequency energy (2-4kHz band) per 5-second window:
```bash
ffmpeg -i "<video_path>" -map 0:a:<voice_index> -af "highpass=f=2000,lowpass=f=4000,ebur128=metadata=1" -f null - 2>&1
```
High values in this band correlate with laughter and screaming (sharp, bright sounds vs. normal speech).

### Silence Detection

```bash
ffmpeg -i "<video_path>" -map 0:a:<voice_index> -af "silencedetect=noise=-35dB:d=3" -f null - 2>&1
```
Parse `silence_start` and `silence_end` timestamps.

### Session Noise Floor Measurement

Use the longest detected silence period (≥3s) to measure the true noise floor for the edit-assembler's noise reduction:
```bash
ffmpeg -i "<video_path>" -ss <longest_silence_start> -t <longest_silence_duration> \
  -map 0:a:<voice_index> -af "astats=metadata=1:reset=1,ametadata=print:key=lavfi.astats.Overall.RMS_level:file=<tmp_dir>/noise_floor.txt" -f null - 2>&1
```
Parse the average RMS level — this is the session-wide `noise_floor_db`. Include it in the output JSON so the edit-assembler can use it instead of guessing from segment starts.

If no silence ≥3s is found, fall back to the first 30 seconds of the video and take the minimum RMS value.

### Drama Detection

For each silence period ≥3s, analyze the 5-second window immediately after silence ends:
- Measure loudness delta vs. session mean
- If Whisper transcript available: check for emotional punctuation (`!`, `?!`), ALL CAPS words, repeated letters ("NEEEEM", "WHAAAAT") in that window
- Score = silence_duration_factor * loudness_delta * transcript_emotion_bonus
  - silence_duration_factor = min(silence_duration / 3.0, 1.5)
  - transcript_emotion_bonus = 1.0 (no transcript) | 1.3 (has exclamation) | 1.5 (ALL CAPS or repeated letters)

### Crosstalk Detection (Multi-track only)

If 2+ voice tracks exist, for each 5-second window, check if both tracks have loudness above session_mean - 6dB simultaneously. Count these as crosstalk windows.

If single track with Whisper: check for Whisper segments with gaps < 0.3s between them AND elevated volume. Heuristic for overlapping speakers.

### Composite Scoring

**Full Tier weights (default):**
| Signal | Weight |
|--------|--------|
| Voice volume spike (dB above mean) | 20% |
| High-freq energy (laughter/screaming) | 30% |
| Crosstalk/simultaneous speakers | 10% |
| Visual motion (scene changes + frame diff) | 15% |
| Drama detection (silence→noise + transcript emotion) | 25% |

Each signal is normalized to 0-1 range (where 1 = the strongest instance in the session). The weighted sum is the raw score. Add exclamation bonus (+0.1 per exclamation in Whisper transcript, capped at +0.3).

**Minimal Tier weights (no Whisper):**
| Signal | Weight |
|--------|--------|
| Voice volume spike | 35% |
| Tension-release (silence→noise delta) | 20% |
| Sustained high energy (≥10s above mean+1σ) | 15% |
| Visual motion (scene changes + frame diff) | 20% |
| Frame difference sustained activity | 10% |

```bash
END_SCORING=$(date +%s%N)
SCORING_MS=$(( (END_SCORING - START_SCORING) / 1000000 ))
```

Report timing: `"scoring_ms": <SCORING_MS>`

### Normalization

1. Normalize all window scores to 0-100 (max window = 100)
2. Compute mean and stddev of scores
3. Report session_mean and session_stddev in the output

### Merging and Padding

Return ALL detected moments (not just above a threshold). The calling command handles filtering.

- Adjacent windows (within 10s) that both score above `mean` merge into one continuous segment
- Add 2s lead-in and 1s lead-out to each segment (no overlap with neighbors)
- If no moments are detected at all: report "No significant moments detected"

## Output

Return a structured list. Print it as a fenced JSON code block so the calling command can parse it:

```json
{
  "tier": "full",
  "language": "hu",
  "source_duration": 12255.0,
  "session_mean": 42.3,
  "session_stddev": 18.7,
  "noise_floor_db": -62.3,
  "transcript_srt": "<tmp_dir>/voice.srt",
  "timing": {
    "audio_probe_ms": 4200,
    "whisper_ms": 758000,
    "visual_analysis_ms": 194000,
    "scoring_ms": 800
  },
  "moments": [
    {
      "index": 1,
      "start": "00:12:28",
      "end": "00:13:07",
      "duration": 39.0,
      "score": 95,
      "signals": ["volume_spike", "high_freq", "crosstalk", "visual_motion"],
      "audio_description": "Mass laughter, 3 voices overlapping, volume spike +12dB above mean",
      "screen_description": "Explosion effect, rapid scene changes (5 in 3s), high sustained motion",
      "transcript_excerpt": "DID HE JUST— NO WAY! [laughing]"
    },
    {
      "index": 2,
      "start": "00:34:12",
      "end": "00:34:55",
      "duration": 43.0,
      "score": 88,
      "signals": ["drama", "volume_spike", "visual_motion"],
      "audio_description": "4s silence followed by sudden shouting, dramatic tension-release pattern",
      "screen_description": "Static menu screen transitions to intense action sequence",
      "transcript_excerpt": "NEEEEM! Várj... WHAT?!"
    },
    {
      "index": 3,
      "start": "01:05:18",
      "end": "01:05:42",
      "duration": 24.0,
      "score": 55,
      "signals": ["volume_spike"],
      "audio_description": "Brief volume spike, single speaker, moderate energy",
      "screen_description": "Menu navigation, minimal visual motion",
      "transcript_excerpt": "na jó, ez szar volt"
    }
  ]
}
```

**Important:** Return ALL detected moments sorted by score descending, not just high-scoring ones. The calling command handles threshold filtering. Each moment MUST include:
- `audio_description`: What's happening in the audio (volume spikes, laughter, silence, crosstalk, tension patterns, etc.)
- `screen_description`: What's happening visually (explosions, scene changes, motion level, static screens, menus, etc.)
- `transcript_excerpt`: Relevant Whisper transcript text (if available)
