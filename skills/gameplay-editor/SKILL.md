---
name: gameplay-editor
description: Use when the user wants to edit gameplay videos, create highlight reels from recordings, detect exciting moments in video, trim long recordings into short clips, or make short-form content from gameplay footage.
---

# Gameplay Video Editor

Transforms long gameplay recordings (30 min - 5 hours) into highlight reels and short-form clips using a 4-phase pipeline: audio preparation, moment detection, browser-based curation, and deferred export.

## When to Use

- User says "edit this gameplay video" or "make a highlight reel"
- User provides a video file and wants it trimmed/edited
- User asks to find the best moments in a recording
- User wants to create TikTok/Shorts/Reels from gameplay
- User mentions OBS recordings, gameplay clips, or highlight reels

## How It Works

This skill delegates to two commands and four agents:

1. **`/gameplay-setup`** — One-time setup (install Whisper, verify ffmpeg)
2. **`/gameplay-edit <path>`** — Main 4-phase workflow:
   - **Phase 1 (Prepare):** audio-preparer agent — detects tracks, denoises, normalizes voice levels
   - **Phase 2 (Detect):** moment-detector agent — scores energy + transcripts, over-detects moments
   - **Phase 3 (Curate):** dashboard-builder agent — opens HTML dashboard in browser with audio previews for keep/remove/comment decisions
   - **Phase 4 (Export):** export-assembler agent — masters audio, assembles highlight reel + auto-generates shorts

## Quick Start

If the user provides a video path, invoke `/gameplay-edit` with it.
If the user asks about setup or dependencies, invoke `/gameplay-setup`.

## Modes

- **analyze** (default): All 4 phases — includes browser dashboard for curation
- **auto**: Skips Phase 3 — auto-selects strong clips (score > 70), exports immediately

## Output

Always produces both:
- **Highlight reel** — single MP4, YouTube-ready (16:9, -16 LUFS)
- **Shorts** — multiple MP4s, TikTok/Reels-ready (9:16, -12 LUFS, 15-60s each)

## Detection

Voice is the content. Scoring prioritizes:
- Transcript analysis via LLM (40% weight in Full tier)
- High-frequency energy — laughter, screaming (20%)
- Volume spikes (15%)
- Signal breadth — multiple signals firing (15%)
- Dynamic range (10%)

Two tiers:
- **Full** (faster-whisper installed): transcription + all signals + LLM analysis
- **Minimal** (ffmpeg only): audio energy-based detection
