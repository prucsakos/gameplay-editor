# Gameplay Editor

A Claude Code plugin that transforms long gameplay recordings into highlight reels and short-form clips using AI-powered audio + visual analysis.

## Installation

```bash
# From GitHub
/plugin install github:prucsakos/gameplay-editor

# From a specific release
/plugin install github:prucsakos/gameplay-editor@v2.0.0
```

## Setup

Run once after installation to install Whisper (optional but recommended):

```bash
/gameplay-setup
```

Requires: ffmpeg in PATH (mandatory), Python 3.8+ (for Whisper, optional).

## Usage

```bash
# Basic — will prompt for language, style, platform, duration
/gameplay-edit path/to/recording.mkv

# Skip the prompt with flags
/gameplay-edit recording.mkv --style funny --platform tiktok --language hu

# Auto-edit (no prompts, no approval step, all defaults)
/gameplay-edit recording.mkv --mode auto

# YouTube + TikTok exports from the same moments
/gameplay-edit recording.mkv --style funny --platform both --duration 10m

# Custom output path
/gameplay-edit recording.mkv --output path/to/output.mp4
```

## Styles

Edit the files in `styles/` to customize:

- **clean** — Minimal cuts, subtle fades, no effects. Best for longer highlight reels where the content speaks for itself.
- **funny** — Meme-style editing with AI-suggested text pop-ups, sound effects, slow-mo, and instant replay. All suggestions shown for approval before applying.

## Platforms

Target platform is separate from style:

- **youtube** — 16:9 original aspect ratio, single highlight reel (default 5m)
- **tiktok** — 9:16 center crop, multiple individual short clips (15-60s), auto-captions, louder audio for mobile
- **both** — exports both formats from the same moments

## Sound Effects

The `assets/sfx/` directory contains bundled sound effects for the funny style. You can also place a `sfx/` directory next to your source video with custom sounds — user sounds override bundled ones with the same filename.

## How It Works

1. Probes audio tracks to find voice vs. game audio
2. Runs Whisper transcription with specified language (falls back to volume analysis if unavailable)
3. Analyzes visual motion via scene detection and frame difference
4. Scores 5-second windows by: laughter/screaming (30%), drama (25%), volume spikes (20%), visual motion (15%), crosstalk (10%)
5. Detects frequently spoken words and suggests running counter overlays
6. Presents a ranked plan with optional funny edit suggestions (if style=funny)
7. Processes audio: noise reduction, normalization, compression, game audio leveling
8. Assembles the edit with ffmpeg (transitions, effects, crop, captions)
9. Exports MP4 with a detailed pipeline timing summary

## Requirements

- **ffmpeg** (mandatory) — video/audio processing
- **Python 3.8+ + openai-whisper** (optional) — better moment detection, auto-captions, word counter
