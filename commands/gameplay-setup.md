---
description: One-time setup for gameplay-editor — installs Whisper and verifies ffmpeg
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(pip:*), Bash(pip3:*), Bash(python:*), Bash(python3:*), Bash(whisper:*), Bash(where:*), Bash(which:*)
---

# Gameplay Editor Setup

Run the following checks and installations in order. Report each result clearly.

## Step 1: Verify ffmpeg

Run `ffmpeg -version` and check it succeeds. If it fails, tell the user:
"ffmpeg is not installed or not in PATH. Install it: winget install Gyan.FFmpeg (Windows), brew install ffmpeg (macOS), apt install ffmpeg (Linux)."
Stop here if ffmpeg is missing — it is mandatory.

Also run `ffprobe -version` to verify ffprobe is available.

## Step 2: Check Python

Run `python --version` or `python3 --version`. If neither works, tell the user:
"Python 3.8+ is required for Whisper transcription. Install from python.org. Without Python, the plugin will use volume-based analysis only (reduced accuracy)."
Continue even if Python is missing — it is optional.

## Step 3: Install openai-whisper

If Python is available, run:
```bash
pip install openai-whisper
```
Or `pip3 install openai-whisper` if `pip` is not available.

If installation succeeds, verify by running:
```bash
whisper --help
```

If `whisper` CLI works, report: "Whisper installed successfully. The small model (~466MB) will download automatically on first use."

If installation fails, report the error and tell the user: "Whisper installation failed. The plugin will use volume-based analysis only. You can retry later with /gameplay-setup."

## Step 4: Summary

Report the final status:
- ffmpeg: OK / MISSING (mandatory)
- ffprobe: OK / MISSING (mandatory)
- Whisper: OK / NOT INSTALLED (optional)
- Detection tier: Full (with Whisper) / Minimal (ffmpeg only)
