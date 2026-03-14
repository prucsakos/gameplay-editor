# Gameplay Editor

A Claude Code plugin that transforms long gameplay recordings into highlight reels and short-form clips using AI-powered audio analysis.

## Installation

```bash
# From GitHub
/plugin install github:prucs/gameplay-editor

# From a specific release
/plugin install github:prucs/gameplay-editor@v1.0.0
```

## Setup

Run once after installation to install Whisper (optional but recommended):

```bash
/gameplay-setup
```

Requires: ffmpeg in PATH (mandatory), Python 3.8+ (for Whisper, optional).

## Usage

```bash
# Analyze a recording and create a 5-min highlight reel (default)
/gameplay-edit path/to/recording.mkv

# Specific style and duration
/gameplay-edit recording.mkv --style polished --duration 10m

# Short-form clip for TikTok/Shorts
/gameplay-edit recording.mkv --style shortform --duration 60s

# Auto-edit (no approval step)
/gameplay-edit recording.mkv --mode auto --style clean

# Manual timestamps
/gameplay-edit recording.mkv --mode manual --timestamps "01:23:00-01:24:30,02:10:00-02:11:15"
```

## Styles

Edit the files in `styles/` to customize:

- **clean** — Minimal cuts, subtle fades, no text. Best for longer highlight reels.
- **polished** — Dissolve transitions, context text overlays, speed ramps. YouTube style.
- **shortform** — 9:16 crop, auto-captions, fast cuts. TikTok/Shorts/Reels.

## How It Works

1. Probes audio tracks to find voice vs. game audio
2. Runs Whisper transcription on voice tracks (falls back to volume analysis if Whisper unavailable)
3. Scores 5-second windows by: volume spikes, laughter/screaming, simultaneous speakers, silence→noise patterns
4. Presents a ranked plan of the best moments
5. Assembles the edit with ffmpeg (transitions, audio normalization, captions, crop)
6. Exports MP4

## Requirements

- **ffmpeg** (mandatory) — video/audio processing
- **Python 3.8+ + openai-whisper** (optional) — better moment detection + auto-captions
