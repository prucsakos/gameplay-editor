---
name: gameplay-editor
description: Use when the user wants to edit gameplay videos, create highlight reels from recordings, detect exciting moments in video, trim long recordings into short clips, or make short-form content from gameplay footage.
---

# Gameplay Video Editor

Transforms long gameplay recordings (1-4 hours) into highlight reels or short-form clips (15-60s) using score-based moment selection.

## When to Use

- User says "edit this gameplay video" or "make a highlight reel"
- User provides a video file and wants it trimmed/edited
- User asks to find the best moments in a recording
- User wants to create TikTok/Shorts/Reels from gameplay
- User mentions OBS recordings, gameplay clips, or highlight reels

## How It Works

This skill delegates to two commands and two agents:

1. **`/gameplay-setup`** — One-time setup (install Whisper, verify ffmpeg)
2. **`/gameplay-edit <path>`** — Main workflow:
   - Prompts for language, platform, and score threshold
   - **audio-analyzer agent** detects and scores ALL moments via audio + visual analysis
   - In analyze mode: presents ALL moments with detailed descriptions (audio + screen + transcript), marks ones above threshold
   - User can approve, adjust threshold, or set a target duration to filter by top scores
   - **edit-assembler agent** processes audio and exports the final video

## Quick Start

If the user provides a video path, invoke `/gameplay-edit` with it.
If the user asks about setup or dependencies, invoke `/gameplay-setup`.

## Clip Selection

Score-threshold-based (not duration-based):
- **Auto mode**: Include all moments above `--score-threshold` (default 70)
- **Analyze mode**: Show ALL moments with descriptions, mark ones above threshold. User can:
  - Approve the threshold selection
  - Change to a different score threshold
  - Set a target duration (top-scoring moments to fill the time)
  - Include/exclude specific moments manually

## Platforms

Target platform is asked separately:
- **youtube** — 16:9, single highlight reel
- **tiktok** — 9:16, multiple individual short clips (15-60s each), auto-captions
- **both** — exports both formats from the same moments

## Detection

Audio + visual analysis scores moments by:
- Laughter/screaming via high-frequency energy (30%)
- Drama detection: silence→noise + transcript emotion (25%)
- Voice volume spikes (20%)
- Visual motion: scene changes + frame difference (15%)
- Simultaneous speakers / crosstalk (10%)

Two tiers:
- **Full** (Whisper installed, default): transcription + all signals + captions
- **Minimal** (ffmpeg only): volume + visual motion based detection, no captions
