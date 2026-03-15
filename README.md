# Gameplay Editor

A Claude Code plugin that transforms long gameplay recordings into highlight reels and short-form clips using AI-powered audio + visual analysis.

## Prerequisites

Before installing this plugin, make sure you have these on your system:

| Dependency | Required | Purpose |
|------------|----------|---------|
| **[Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview)** | Yes | Plugin host — this is a Claude Code plugin |
| **[ffmpeg](https://ffmpeg.org/download.html)** | Yes | All video/audio processing |
| **Python 3.8+** | No (recommended) | Required only for Whisper transcription |
| **[openai-whisper](https://github.com/openai/whisper)** | No (recommended) | Better moment detection via transcription |
| **[PyTorch](https://pytorch.org/)** | No (auto-installed with Whisper) | Hard dependency of Whisper — installed automatically via `pip install openai-whisper`. For GPU acceleration, install the CUDA build from [pytorch.org](https://pytorch.org/get-started/locally/) |

Verify ffmpeg is available:

```bash
ffmpeg -version
```

## Setting Up the Marketplace

If you want to publish or browse this plugin via a marketplace registry, the config lives in `.claude-plugin/marketplace.json`. This file declares the plugin owner and source so that marketplace tooling can resolve it:

```json
{
  "name": "prucsakos",
  "owner": { "name": "prucsakos" },
  "plugins": [
    {
      "name": "gameplay-editor",
      "description": "Transform long gameplay recordings into highlight reels and short-form clips using AI-powered audio + visual analysis",
      "version": "3.0.0",
      "source": { "source": "github", "repo": "prucsakos/gameplay-editor" }
    }
  ]
}
```

No additional setup is needed — the marketplace entry points back to the GitHub repo where the plugin is hosted.

## Installation

```bash
# 1. Add the marketplace
/plugin marketplace add prucsakos/gameplay-editor

# 2. Install the plugin
/plugin install gameplay-editor@prucsakos-gameplay-editor
```

After installing, run the one-time setup command to verify dependencies and optionally install Whisper:

```bash
/gameplay-setup
```

This checks that ffmpeg is in your PATH and walks you through installing openai-whisper if Python is available. Whisper is optional but enables the **Full tier** of detection (transcription + emotion analysis). Without it, the plugin falls back to the **Minimal tier** (volume + visual motion only).

## Usage

```bash
# Basic — will prompt for language, platform, score threshold
/gameplay-edit path/to/recording.mkv

# Skip the prompt with flags
/gameplay-edit recording.mkv --platform tiktok --language hu --score-threshold 60

# Auto-edit (no prompts, no approval step, default threshold 70)
/gameplay-edit recording.mkv --mode auto

# Use target duration instead of score threshold
/gameplay-edit recording.mkv --duration 5m --platform both

# Custom output path
/gameplay-edit recording.mkv --output path/to/output.mp4
```

## Clip Selection

The plugin uses **score-based selection** instead of targeting a specific output duration:

- **Default**: Include all moments scoring ≥ 70 (configurable via `--score-threshold`)
- **Duration mode**: If `--duration` is provided, selects the highest-scoring moments that fit within the target
- **Analyze mode**: Shows ALL detected moments with detailed descriptions (audio, screen, transcript). You choose which to include by adjusting the threshold or cherry-picking.

## Platforms

Target platform is separate from clip selection:

- **youtube** — 16:9 original aspect ratio, single highlight reel
- **tiktok** — 9:16 center crop, multiple individual short clips (15-60s), louder audio for mobile
- **both** — exports both formats from the same moments

## How It Works

1. Probes audio tracks to find voice vs. game audio
2. Runs Whisper transcription with specified language (falls back to volume analysis if unavailable)
3. Analyzes visual motion via scene detection and frame difference
4. Scores 5-second windows by: laughter/screaming (30%), drama (25%), volume spikes (20%), visual motion (15%), crosstalk (10%)
5. Returns ALL moments with detailed audio/screen/transcript descriptions
6. Filters by score threshold (or duration target) with user approval
7. Processes audio: noise reduction (from silent-section noise floor), normalization, compression, game audio leveling
8. Assembles the edit with ffmpeg (transitions, crop)
9. Exports MP4 with score in filenames and a pipeline timing summary

## Plugin Structure

```
gameplay-editor/
├── .claude-plugin/
│   ├── plugin.json          # Plugin metadata (name, version, author)
│   └── marketplace.json     # Marketplace registry entry
├── commands/
│   ├── gameplay-edit.md      # Main editing command
│   └── gameplay-setup.md     # One-time setup command
├── agents/
│   ├── audio-analyzer.md     # Moment detection agent
│   └── edit-assembler.md     # Video assembly agent
├── skills/
│   └── gameplay-editor/
│       └── SKILL.md          # Skill trigger definition
└── styles/
    └── clean.md              # Clean style preset
```
