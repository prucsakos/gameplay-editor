---
description: Edit gameplay videos into highlight reels and short-form clips
argument-hint: <video_path> [--language hu] [--mode analyze|auto] [--output path]
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(python:*), Bash(python3:*), Bash(ls:*), Bash(mkdir:*), Bash(rm:*), Bash(cat:*), Bash(head:*), Bash(date:*), Bash(stat:*), Bash(mktemp:*), Bash(df:*), Bash(start:*)
---

# Gameplay Video Editor (v4)

You are editing a gameplay video into a highlight reel and short-form clips using a 4-phase pipeline: Prepare, Detect, Curate, Export.

## Parse Arguments

From the user's input, extract:
- **video_path** (required): path to the source video file
- **language** (default: `hu`): Whisper language code
- **mode** (default: `analyze`): one of `analyze`, `auto`
  - `analyze`: runs all 4 phases including the dashboard for curation
  - `auto`: skips Phase 3 (dashboard), auto-selects strong clips (score > 70), exports immediately
- **output** (default: `Outs/` relative to source video's parent directory)

**Removed from v3:** `--platform`, `--score-threshold`, `--duration`. The v4 pipeline always produces both a highlight reel (YouTube-ready) and shorts (TikTok-ready). Score threshold is now in the style config (`detection_threshold`).

## Windows Compatibility

**PYTHONUTF8=1**: All `python3`/`python` invocations MUST be prefixed with `PYTHONUTF8=1`.

**ASCII-only output**: All printed output from Python scripts must use ASCII characters only.

## Validate

1. Check the video file exists: `ls -la "<video_path>"`
2. Check ffmpeg is available: `ffmpeg -version`
3. Check video has audio tracks: `ffprobe -v quiet -print_format json -show_streams "<video_path>"` — if no audio, report error and stop
4. Read the style file from this plugin's `styles/clean.md` — parse the YAML frontmatter for all parameters including v4 fields (`shorts_count`, `shorts_duration_range`, `detection_threshold`)
5. Create the output directory if it doesn't exist
6. Estimate required disk space (source_size * 2) and warn if <2GB free

## Check faster-whisper Availability

```bash
PYTHONUTF8=1 python3 -c "from faster_whisper import WhisperModel; print('ok')"
```

If it works: `"Full tier — transcription + content analysis enabled"`
If not: `"Minimal tier — audio-only analysis. Run /gameplay-setup for better results."`

## Generate Session ID

```bash
SESSION_ID=$(PYTHONUTF8=1 python3 -c "import uuid; print(uuid.uuid4().hex[:12])")
```

## Single Prompt (analyze mode only)

If mode is `analyze` and language was not provided via CLI:

```
Quick setup:
- Language? (default: hu)
```

Parse flexibly. If mode is `auto` or language was provided, skip.

## Create Temp Directory

```bash
TMP=$(PYTHONUTF8=1 python3 -c "import tempfile; print(tempfile.mkdtemp(prefix='gameplay-editor-'))")
```

Verify: `ls -d "$TMP"`

Record start time:
```bash
PIPELINE_START=$(date +%s%N)
```

## Phase 1: PREPARE

Report: `"[1/4] Preparing audio — detecting tracks, denoising, normalizing..."`

Dispatch the **audio-preparer** agent with:
- video_path
- tmp_dir: $TMP
- session_id: $SESSION_ID

The agent returns: analysis_mix path, clean voice track paths, track map, noise profiles, LUFS offsets, source metadata, timing.

## Phase 2: DETECT

Report: `"[2/4] Detecting moments — scoring energy, transcribing, analyzing content..."`

Dispatch the **moment-detector** agent with:
- analysis_mix: from Phase 1 output
- clean_voice_tracks: from Phase 1 output
- track_map: from Phase 1 output
- source_duration: from Phase 1 output
- language
- tmp_dir: $TMP
- transcript_signals: from style config
- detection_threshold: from style config (default 30)
- has_whisper: true/false from availability check

The agent returns: moment list with scores, descriptions, context zones, transcript data, timing.

## Phase 3: CURATE (analyze mode only)

**If mode is `auto`:** Skip Phase 3 entirely. Filter moments to only those with score > 70 (strong confidence). Construct a synthetic decisions object with all strong moments set to `action: "keep"`.

**If mode is `analyze`:**

Report: `"[3/4] Building clip dashboard — N moments detected (M strong, K maybe)..."`

Dispatch the **dashboard-builder** agent with:
- moments: from Phase 2 output
- clean_voice_tracks: from Phase 1 output
- track_map: from Phase 1 output
- session_id: $SESSION_ID
- source_video: video_path
- tmp_dir: $TMP
- has_transcript: from Phase 2 output

The agent returns: decisions (kept/removed/commented moments), summary, timing.

## Phase 4: EXPORT

Report: `"[4/4] Exporting highlight reel and shorts..."`

Dispatch the **export-assembler** agent with:
- source_video_path: video_path
- decisions: from Phase 3 output (or synthetic decisions in auto mode)
- moments: from Phase 2 output (for scores, descriptions)
- noise_profiles: from Phase 1 output
- track_map: from Phase 1 output
- lufs_offsets: from Phase 1 output
- style: parsed style config
- output_dir: resolved output path
- source_width, source_height, source_fps: from Phase 1 output
- shorts_count: from style config
- shorts_duration_range: from style config

The agent returns: output file paths, sizes, durations, timing.

## Pipeline Summary

```bash
PIPELINE_END=$(date +%s%N)
TOTAL_MS=$(( (PIPELINE_END - PIPELINE_START) / 1000000 ))
```

Present:
```
=== Pipeline Summary ===
Source: <filename> (<source_duration>)

Highlight reel: <highlight_path> (<duration>, <size>)
Shorts: <N> clips generated
  <short_1_path> (<duration>, <size>) — Score: <score>
  <short_2_path> (<duration>, <size>) — Score: <score>

Moments: <included> included, <removed> removed by user
         <commented> had comments processed

Timing:
  Phase 1 (Prepare):  <time> (denoise + normalize)
  Phase 2 (Detect):   <time> (scoring + transcription)
  Phase 3 (Curate):   <time> (user curation in dashboard)
  Phase 4 (Export):    <time> (mastering + encoding)
  ─────────────────────────
  Total:               <total time>

Tier: <Full|Minimal> | Language: <language>
```

## Quick-Fix Loop (optional, one round)

After presenting the summary, ask:

```
Watch the results and let me know if you'd like any corrections (one round):
- "remove the clip at 1:45 in the highlight reel"
- "short #2 audio is too quiet"
- "add 5 more seconds to the ending"

Or say "done" to finish.
```

If the user provides corrections, the **command itself** (not the export-assembler) transforms them into modified decisions:
- "remove the clip at 1:45" → find the moment containing 1:45, set `action: "remove"` in decisions
- "short #2 audio is too quiet" → no decision change needed; pass a `corrections` note to the agent for re-mastering that short at +3dB
- "add 5 more seconds to the ending" → extend the last moment's `end` by 5s in decisions

Then re-dispatch the export-assembler with the modified decisions object. The export-assembler treats every dispatch identically — it has no concept of "first run" vs "correction run." It simply processes whatever decisions it receives.

Present updated summary after re-export.

If the user says "done" or similar, proceed to cleanup.

## Cleanup

```bash
rm -rf "$TMP"
```

Report: `"Temp files cleaned up. Done!"`

## Error Handling

- Source video not found → report error, stop
- ffmpeg not available → report install instructions, stop
- No audio tracks → report error, stop
- faster-whisper not available → warn, continue with Minimal tier
- No moments above detection_threshold → report, suggest checking audio quality
- Dashboard timeout → report, suggest re-running `/gameplay-edit`
- Segment processing fails → skip segment, continue, report at end
- Disk space < 2GB → warn before starting
