---
name: gameplay-editor
description: Use when the user wants to edit gameplay videos, create highlight reels from recordings, detect exciting moments in video, trim long recordings into short clips, or make short-form content from gameplay footage.
---

# Gameplay Video Editor

Transforms long gameplay recordings (1-4 hours) into highlight reels (5-10 min) or short-form clips (15-60s).

## When to Use

- User says "edit this gameplay video" or "make a highlight reel"
- User provides a video file and wants it trimmed/edited
- User asks to find the best/funniest moments in a recording
- User wants to create TikTok/Shorts/Reels from gameplay
- User mentions OBS recordings, gameplay clips, or highlight reels

## How It Works

This skill delegates to two commands and two agents:

1. **`/gameplay-setup`** — One-time setup (install Whisper, verify ffmpeg)
2. **`/gameplay-edit <path>`** — Main workflow:
   - Prompts for language, style, platform, and duration
   - **audio-analyzer agent** detects exciting moments via audio + visual analysis
   - User reviews and approves the moment plan (with funny edit suggestions if style=funny)
   - **edit-assembler agent** processes audio, applies edits, and exports the final video

## Quick Start

If the user provides a video path, invoke `/gameplay-edit` with it.
If the user asks about setup or dependencies, invoke `/gameplay-setup`.

## Styles

Two style presets in `styles/` directory (user-editable):
- **clean** — Minimal cuts, fade transitions, no effects. Content speaks for itself.
- **funny** — Meme-style editing: text pop-ups, sound effects, slow-mo, instant replay. All suggestions require user approval.

## Platforms

Target platform is asked separately from style:
- **youtube** — 16:9, single highlight reel (default 5m)
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
- **Full** (Whisper installed, default): transcription + all signals + captions + word frequency detection
- **Minimal** (ffmpeg only): volume + visual motion based detection, no captions
