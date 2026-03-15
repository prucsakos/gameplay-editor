---
name: audio-analyzer
description: |
  Use this agent to analyze audio tracks in a gameplay video and detect exciting moments.
  Returns a scored, ranked list of moments with timestamps and descriptions.
model: sonnet
---

# Audio Analyzer Agent

You analyze gameplay video audio to find the most exciting moments — laughter, screaming, crosstalk, exclamations, and tension-release patterns.

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

## Stage 2: Sound Quality Preprocessing

After track detection, preprocess all voice tracks to improve Whisper accuracy and scoring quality.

### Extract Voice Tracks

For each voice track:
```bash
START_PREPROC=$(date +%s%N)
ffmpeg -i "<video_path>" -map 0:a:<voice_index> -ac 1 -ar 16000 -y "<tmp_dir>/voice.wav"
```
If multiple voice tracks: `voice.wav`, `voice_2.wav`, etc.

### Measure Noise Floor

Detect all silent sections:
```bash
ffmpeg -i "<tmp_dir>/voice.wav" -af "silencedetect=noise=-35dB:d=1" -f null - 2>&1
```

Parse all `silence_start` / `silence_end` pairs. Find the **longest** quiet section. Measure its RMS level:
```bash
ffmpeg -i "<tmp_dir>/voice.wav" -ss <longest_silence_start> -t <longest_silence_duration> \
  -af "astats=metadata=1:reset=1,ametadata=print:key=lavfi.astats.Overall.RMS_level:file=<tmp_dir>/noise_floor.txt" -f null - 2>&1
```

Parse the average RMS level — this is the session-wide `noise_floor_db`.

### Apply Noise Suppression and Voice Enhancement

For each voice track:
```bash
ffmpeg -i "<tmp_dir>/voice.wav" \
  -af "afftdn=nf=<noise_floor_db>,highpass=f=80,lowpass=f=12000,acompressor=threshold=0.05:ratio=3:attack=5:release=50" \
  -y "<tmp_dir>/voice_clean.wav"
```

- `afftdn=nf=<noise_floor_db>` — FFT-based noise reduction tuned to measured floor
- `highpass=f=80` — removes rumble/hum below 80Hz
- `lowpass=f=12000` — removes hiss above speech range
- `acompressor` — evens out quiet/loud speech

Output: `voice_clean.wav` (and `voice_2_clean.wav` etc.). Raw `voice.wav` is kept but not used further. Whisper SRT output filename stays `voice.srt`.

```bash
END_PREPROC=$(date +%s%N)
PREPROC_MS=$(( (END_PREPROC - START_PREPROC) / 1000000 ))
```

Report: `"Noise floor: <noise_floor_db> dB, preprocessing complete"`
Report timing: `"preprocessing_ms": <PREPROC_MS>`

## Stage 3: Whisper Transcription (uses cleaned audio)

Check if `whisper` CLI is available by running `whisper --help`.

If available:

1. Run Whisper with the specified language. **Important:** Set `PYTHONIOENCODING=utf-8` to ensure correct character encoding for languages with non-ASCII characters (e.g., Hungarian á, é, ő, ű):
   ```bash
   START_WHISPER=$(date +%s%N)
   PYTHONIOENCODING=utf-8 whisper "<tmp_dir>/voice_clean.wav" --model small --language <language> --output_format srt --output_dir "<tmp_dir>/"
   END_WHISPER=$(date +%s%N)
   WHISPER_MS=$(( (END_WHISPER - START_WHISPER) / 1000000 ))
   ```
   This produces `voice_clean.srt`. Rename it to `voice.srt`:
   ```bash
   mv "<tmp_dir>/voice_clean.srt" "<tmp_dir>/voice.srt"
   ```

2. **Verify encoding:** Check that the output files are valid UTF-8:
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

3. Parse `voice.srt` for segment-level timestamps and text.

**When reading Whisper output files**, always open them with explicit `encoding='utf-8'`. On Windows, the default encoding may not be UTF-8.

If Whisper is not available, skip this stage and note: `"Whisper unavailable, using Minimal tier."`

If Whisper starts but fails mid-run (non-zero exit code), check how much of the `.srt` was produced. Report: `"Whisper failed at approximately <last_timestamp>. Using partial transcript + Minimal tier for remaining duration."` Use whatever transcript was generated for the covered portion, and fall back to Minimal tier scoring for the rest.

Report timing: `"whisper_ms": <WHISPER_MS>`

## Stage 4: Energy Scoring (uses cleaned audio)

### Loudness Analysis

Extract per-second loudness data for the voice track(s):
```bash
START_SCORING=$(date +%s%N)
ffmpeg -i "<tmp_dir>/voice_clean.wav" -af "ebur128=metadata=1,ametadata=print:key=lavfi.r128.M:file=<tmp_dir>/loudness.txt" -f null - 2>&1
```
Parse the output to get momentary loudness (M) values per measurement interval. Group into 5-second windows.

Compute the session mean and standard deviation of loudness across all windows.

### High-Frequency Energy (Laughter/Screaming)

Extract high-frequency energy (2-4kHz band) per 5-second window:
```bash
ffmpeg -i "<tmp_dir>/voice_clean.wav" -af "highpass=f=2000,lowpass=f=4000,ebur128=metadata=1" -f null - 2>&1
```
High values in this band correlate with laughter and screaming (sharp, bright sounds vs. normal speech).

### Silence Detection

```bash
ffmpeg -i "<tmp_dir>/voice_clean.wav" -af "silencedetect=noise=-35dB:d=3" -f null - 2>&1
```
Parse `silence_start` and `silence_end` timestamps.

### Tension-Release Detection

For each silence period ≥3s (from Silence Detection), analyze the 5-second window immediately after silence ends:
- Measure loudness delta vs session mean
- Score = min(silence_duration / 3.0, 1.5) × loudness_delta
- Normalize to 0-1 range across all windows (min-max)

### Crosstalk Detection

If 2+ voice tracks exist, for each 5-second window, check if both tracks have loudness above session_mean - 6dB simultaneously. Count these as crosstalk windows.

If single track with Whisper: check for Whisper segments with gaps < 0.3s between them AND elevated volume. Heuristic for overlapping speakers.

### Exclamation Detection (requires Whisper transcript)

If a Whisper transcript (`.srt`) was produced (full or partial), scan it for exclamation markers:
- `!` punctuation
- ALL CAPS words (3+ characters)
- Repeated letters ("NOOO", "WHAAAAT", "hahaha")

For each 5-second window, count occurrences that fall within that window's timestamp range. Each occurrence adds +0.1 to the window's final score, capped at +0.3 per window. Applied after the weighted sum.

### Composite Scoring

Score each 5-second window using the **Minimal Tier** formula:

    score = 0.30 × Volume + 0.25 × HighFreq + 0.20 × Crosstalk + 0.15 × TensionRelease + 0.10 × Breadth
    + exclamation bonus

| Dimension | Weight | Source |
|-----------|--------|--------|
| Volume | 0.30 | Per-window RMS loudness relative to session mean (from Loudness Analysis) |
| High Freq | 0.25 | Ratio of high-frequency energy >2kHz (from High-Frequency Energy) |
| Crosstalk | 0.20 | Simultaneous speakers per window (from Crosstalk Detection) |
| Tension-Release | 0.15 | Silence→loudness spike score (from Tension-Release Detection) |
| Breadth | 0.10 | Signal diversity bonus: `active_dimensions / 4`. A dimension is "active" if its normalized value > 0.08. Counts: Volume, High Freq, Crosstalk, Tension-Release. |

Each dimension is normalized to 0-1 range (min-max across all windows in the session). The weighted sum is the raw score per window. Then add exclamation bonus (+0.1 per occurrence, capped +0.3).

**Note:** This is always the Minimal Tier formula. When a valid transcript exists, the calling command (gameplay-edit) recomputes composite scores using the Full Tier 6-dimension formula with the transcript-analyzer's Transcript scores.

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
  "has_transcript": true,
  "transcript_srt": "<tmp_dir>/voice.srt",
  "timing": {
    "audio_probe_ms": 4200,
    "preprocessing_ms": 12300,
    "whisper_ms": 758000,
    "scoring_ms": 800
  },
  "window_dimensions": [
    { "window_start": 0.0, "window_end": 5.0, "volume": 0.32, "high_freq": 0.15, "crosstalk": 0.00, "tension_release": 0.22 },
    { "window_start": 5.0, "window_end": 10.0, "volume": 0.41, "high_freq": 0.67, "crosstalk": 0.55, "tension_release": 0.38 }
  ],
  "moments": [
    {
      "index": 1,
      "start": "00:12:28",
      "end": "00:13:07",
      "duration": 39.0,
      "score": 95,
      "signals": ["volume_spike", "high_freq", "crosstalk"],
      "audio_description": "Mass laughter, 3 voices overlapping, volume spike +12dB above mean",
      "transcript_excerpt": "DID HE JUST— NO WAY! [laughing]"
    },
    {
      "index": 2,
      "start": "00:34:12",
      "end": "00:34:55",
      "duration": 43.0,
      "score": 88,
      "signals": ["tension_release", "volume_spike"],
      "audio_description": "4s silence followed by sudden shouting, dramatic tension-release pattern",
      "transcript_excerpt": "NEEEEM! Várj... WHAT?!"
    }
  ]
}
```

**New fields:**
- `has_transcript`: `true` if Whisper produced a `.srt` with at least one speech segment. `false` if Whisper failed, was unavailable, or the `.srt` contains zero segments.
- `window_dimensions`: Per-window normalized (0-1) values for each of the 4 audio dimensions (volume, high_freq, crosstalk, tension_release). These are the raw dimension values computed during Stage 4 scoring, before moment merging. Always included regardless of tier. The calling command uses these to compute 6-dimension composite scores when a transcript-analyzer provides the Transcript dimension.

**Important:** Return ALL detected moments sorted by score descending, not just high-scoring ones. The calling command handles threshold filtering. Each moment MUST include:
- `audio_description`: What's happening in the audio (volume spikes, laughter, silence, crosstalk, tension patterns, etc.)
- `transcript_excerpt`: Relevant Whisper transcript text (if available)
