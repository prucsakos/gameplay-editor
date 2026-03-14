---
description: One-time setup for gameplay-editor — installs Whisper and verifies ffmpeg
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(pip:*), Bash(pip3:*), Bash(python:*), Bash(python3:*), Bash(whisper:*), Bash(where:*), Bash(which:*), Bash(curl:*), Bash(ls:*), Bash(mkdir:*)
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

## Step 4: Download Sound Effects (optional)

Check if `assets/sfx/` in this plugin's directory already has `.mp3` files:
```bash
ls <plugin_dir>/assets/sfx/*.mp3 2>/dev/null | wc -l
```

If SFX files already exist, report: "SFX library: N files found. Skipping."

If no SFX files exist, ask the user:
> "No sound effects found. Download the meme SFX starter pack? (8 files, ~500KB total, from Internet Archive and GitHub) [y/n]"

If the user declines, skip this step.

If the user agrees, download the following sounds using curl:

```bash
SFX_DIR="<plugin_dir>/assets/sfx"

# Source 1: Internet Archive — Meme Sound Effect collection
# https://archive.org/details/final_20220209
IA_BASE="https://archive.org/download/final_20220209"

curl -L -o "$SFX_DIR/bruh.mp3"        "$IA_BASE/BRUH.mp3"
curl -L -o "$SFX_DIR/vine_boom.mp3"   "$IA_BASE/VINE%20BOOM%20SOUND.mp3"
curl -L -o "$SFX_DIR/airhorn.mp3"     "$IA_BASE/Airhorn.mp3"
curl -L -o "$SFX_DIR/fail_horn.mp3"   "$IA_BASE/Sad%20Trombone.mp3"
curl -L -o "$SFX_DIR/laugh_track.mp3" "$IA_BASE/Laughing%20track.mp3"

# Source 2: GitHub — Lexz-08/YouTube-Memes (free to use for YouTube)
# https://github.com/Lexz-08/YouTube-Memes
GH_BASE="https://raw.githubusercontent.com/Lexz-08/YouTube-Memes/main"

curl -L -o "$SFX_DIR/sad_violin.mp3"    "$GH_BASE/Sadness-1.mp3"
curl -L -o "$SFX_DIR/dramatic_boom.mp3"  "$GH_BASE/Metal%20Boom.mp3"
curl -L -o "$SFX_DIR/oof.mp3"            "$GH_BASE/MINECRAFT%20OOF.mp3"
```

After each download, verify the file exists and is >0 bytes. If any download fails, report which one and continue with the rest.

After downloading, trim all files to max 5 seconds and normalize volume using ffmpeg (keeps file sizes small and consistent):
```bash
for f in "$SFX_DIR"/*.mp3; do
  ffmpeg -i "$f" -t 5 -af "loudnorm=I=-16:LRA=11:TP=-1.5" -y "$f.tmp" && mv "$f.tmp" "$f"
done
```

Report:
```
SFX library downloaded (8 files):
  bruh.mp3, sad_violin.mp3, fail_horn.mp3, laugh_track.mp3,
  dramatic_boom.mp3, oof.mp3, airhorn.mp3, vine_boom.mp3

Sources:
  - Internet Archive: archive.org/details/final_20220209
  - GitHub: github.com/Lexz-08/YouTube-Memes
```

## Step 5: Summary

Report the final status:
- ffmpeg: OK / MISSING (mandatory)
- ffprobe: OK / MISSING (mandatory)
- Whisper: OK / NOT INSTALLED (optional)
- SFX library: N files / EMPTY (optional)
- Detection tier: Full (with Whisper) / Minimal (ffmpeg only)
