---
name: moment-detector
description: |
  Use this agent to detect and score moments in pre-processed gameplay audio.
  Performs energy scoring, transcription, LLM transcript scoring, and over-detection.
  Returns a comprehensive moment list with extended context for dashboard curation.
model: sonnet
---

# Moment Detector Agent

You detect exciting moments in gameplay audio that has already been cleaned and normalized by the audio-preparer. You score using both audio energy and transcript analysis, intentionally over-detecting to let the user curate in the dashboard.

## Input

You receive:
- **analysis_mix**: path to the combined analysis WAV (from audio-preparer)
- **clean_voice_tracks**: list of paths to individual denoised voice WAVs (for transcription)
- **track_map**: which tracks are voice vs. game (from audio-preparer)
- **source_duration**: total video length in seconds
- **language**: Whisper language code (default: `hu`)
- **tmp_dir**: temp directory for intermediate files
- **transcript_signals**: list from the active style (e.g., `["humor", "dramatic reactions", "banter"]`)
- **detection_threshold**: minimum score to include (default: 30 — intentionally low for over-detection)
- **has_whisper**: boolean — whether faster-whisper is available

## Windows Compatibility

All Python invocations (`python3` or `python`) MUST be prefixed with `PYTHONUTF8=1`. Avoid non-ASCII decorative characters in Python output.

## Step 1: Energy Scoring

On the analysis mix, compute per-5-second-window dimensions:

### Loudness (Volume)

```bash
ffmpeg -i "<analysis_mix>" -af "ebur128=metadata=1,ametadata=print:key=lavfi.r128.M:file=<tmp_dir>/loudness.txt" -f null - 2>&1
```

Parse momentary loudness (M) values. Group into 5-second windows. Compute session mean and stddev.

### High-Frequency Energy

```bash
ffmpeg -i "<analysis_mix>" -af "highpass=f=2000,lowpass=f=4000,ebur128=metadata=1,ametadata=print:key=lavfi.r128.M:file=<tmp_dir>/highfreq.txt" -f null - 2>&1
```

Extract per-5-second-window energy in the 2-4kHz band.

### Dynamic Range

From the loudness data, compute per-second amplitude standard deviation within each 5-second window.

### Normalization

Normalize each dimension to 0-1 range: `normalized = value / max(all_values_for_dimension)`. This is max-normalization (not min-max), so the highest window = 1.0 and silence = 0.0. Consistent across all dimensions including transcript scoring.

## Step 2: Transcription (Full Tier only)

**Gate:** Only run if `has_whisper` is true.

For each clean voice track, run faster-whisper:

```bash
PYTHONUTF8=1 python3 << 'PYEOF'
import sys
from faster_whisper import WhisperModel

model = WhisperModel("small", device="auto", compute_type="auto")
wav_path = "<clean_voice_track_N>"
segments, info = model.transcribe(wav_path, language="<language>", vad_filter=True)

count = 0
with open("<tmp_dir>/voice_<N>.srt", "w", encoding="utf-8") as f:
    for i, seg in enumerate(segments, 1):
        count = i
        def fmt(t):
            h = int(t // 3600)
            m = int((t % 3600) // 60)
            s = int(t % 60)
            ms = int((t % 1) * 1000)
            return f"{h:02d}:{m:02d}:{s:02d},{ms:03d}"
        f.write(f"{i}\n{fmt(seg.start)} --> {fmt(seg.end)}\n{seg.text.strip()}\n\n")

print(f"Transcribed {count} segments, language: {info.language} (prob={info.language_probability:.2f})")
PYEOF
```

**Merge transcripts with speaker labels:**
- Read each `.srt` file
- Prefix each segment's text with the speaker label from `track_map` (e.g., "Speaker 1: ", "Speaker 2: ")
- Merge all segments chronologically into `<tmp_dir>/merged_transcript.srt`
- Resolve overlapping segments by interleaving

If transcription fails, warn and set `has_transcript: false`. Continue with audio-only scoring.

## Step 3: Transcript Scoring (LLM Pass, Full Tier only)

**Gate:** Only run if transcription produced a valid `.srt`.

Read the merged transcript in **2-minute chunks with 15-second overlap** at boundaries.

For each chunk, score its constituent 5-second windows 0-10 based on the `transcript_signals` criteria:

| Range | Meaning |
|-------|---------|
| 0 | Silence or completely uninteresting |
| 1-3 | Normal conversation, no signal match |
| 4-6 | Mild signal presence (slight humor, minor reaction) |
| 7-8 | Strong signal match (clear humor, dramatic moment) |
| 9-10 | Peak excitement (multiple strong signals converging) |

For overlap windows (appearing in two chunks), take the higher score.

Normalize all raw scores to 0-1: `normalized = raw_score / max(all_raw_scores)`. If all zero, stay zero.

For each moment, write a 1-2 sentence summary describing what's happening. Summaries must be in the **transcript's language** (e.g., Hungarian if gameplay is in Hungarian). Be specific — "Speaker 1 tells a story about X, Speaker 2 starts laughing" not "an exciting moment occurs."

**Failure handling:** If LLM scoring fails (timeout, malformed output), retry once. If still failing, fall back to Minimal Tier weights (audio-only) for affected chunks and warn user.

## Step 4: Moment Assembly

### Compute Composite Scores

**Full Tier (has_transcript = true):**
```
score = 0.15 x Volume + 0.20 x HighFreq + 0.10 x DynRange + 0.15 x Breadth + 0.40 x Transcript
```

**Minimal Tier (has_transcript = false):**
```
score = 0.25 x Volume + 0.35 x HighFreq + 0.20 x DynRange + 0.20 x Breadth
```

**Breadth calculation:**
- Full Tier: `active_of(Volume, HighFreq, DynRange, Transcript) / 4` — Breadth itself excluded from its own count
- Minimal Tier: `active_of(Volume, HighFreq, DynRange) / 3`
- A dimension is "active" if its normalized value > 0.08

Normalize composite scores to 0-100 (max window = 100).

### Merge and Pad

- Include everything scoring above `detection_threshold` (default 30)
- Adjacent above-threshold windows (gap <= 10s) merge into one continuous moment
- Moment score = peak window score within the merged moment
- Add 2s lead-in and 1s lead-out padding (asymmetric — captures buildup)
- No overlap with neighboring moments' core boundaries

### Extended Context Zones

For each moment, compute a context zone:
- Context start = moment start - 10s (clamped to 0 and to neighboring moment's core end)
- Context end = moment end + 10s (clamped to source_duration and to neighboring moment's core start)

### Confidence Tiers

- **Strong:** score > 70, at least 2 dimensions with normalized value > 0.08
- **Maybe:** score 30-70, or only 1 active dimension

## Output

Return a structured result as a fenced JSON code block:

```json
{
  "tier": "full",
  "has_transcript": true,
  "session_mean": 42.3,
  "session_stddev": 18.7,
  "transcript_srt": "<tmp_dir>/merged_transcript.srt",
  "moments": [
    {
      "id": 1,
      "start": 748.0,
      "end": 787.0,
      "duration": 39.0,
      "score": 95,
      "confidence": "strong",
      "dimensions": {
        "volume": 0.82,
        "high_freq": 0.91,
        "dyn_range": 0.65,
        "breadth": 1.0,
        "transcript": 0.88
      },
      "signals": ["volume_spike", "high_freq", "crosstalk"],
      "audio_description": "Mass laughter, 3 voices overlapping, volume spike +12dB above mean",
      "transcript_excerpt": "Speaker 1: DID HE JUST-- Speaker 2: NO WAY! [laughing]",
      "summary": "Mindenki egyszerre kiabál amikor a csapattárs véletlenül felrobbantja az egész bázist",
      "context_start": 738.0,
      "context_end": 797.0
    }
  ],
  "window_dimensions": [
    { "window_start": 0.0, "window_end": 5.0, "volume": 0.32, "high_freq": 0.15, "dyn_range": 0.22, "transcript": 0.05 }
  ],
  "timing": {
    "energy_scoring_ms": 8000,
    "transcription_ms": 758000,
    "transcript_scoring_ms": 45000,
    "assembly_ms": 200,
    "total_ms": 811200
  }
}
```

**Fields:**
- `moments[].id`: Sequential integer, used as stable reference through dashboard and export
- `moments[].context_start`/`context_end`: Extended zone for "hear more" in dashboard
- `moments[].dimensions`: Per-dimension normalized values (0-1) for this moment's peak window
- `moments[].confidence`: `"strong"` or `"maybe"`
- `moments[].summary`: LLM-generated description in transcript language (only in Full Tier; empty string in Minimal)
- `window_dimensions`: Every 5s window's dimensions — used if dashboard needs fine-grained data
