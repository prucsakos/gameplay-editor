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

## Step 4: Install Sound Effects (optional)

Check if `assets/sfx/` in this plugin's directory already has `.mp3` files:
```bash
ls <plugin_dir>/assets/sfx/*.mp3 2>/dev/null | wc -l
```

If SFX files already exist, report: "SFX library: N files found. Skipping."

If no SFX files exist, ask the user:
> "No sound effects found. Generate a starter SFX pack using ffmpeg? These are synthetic approximations — you can replace them with real meme sounds later. [y/n]"

If the user declines, skip this step.

If the user agrees, generate the following sounds using ffmpeg:

```bash
SFX_DIR="<plugin_dir>/assets/sfx"

# bruh.mp3 — low descending tone (meme "bruh" approximation)
ffmpeg -f lavfi -i "sine=frequency=150:duration=0.8" -af "aformat=sample_rates=44100,volume=0.8,afade=t=in:d=0.05,afade=t=out:st=0.5:d=0.3,vibrato=f=5:d=0.5" -y "$SFX_DIR/bruh.mp3"

# sad_violin.mp3 — descending sine with vibrato (sad sting)
ffmpeg -f lavfi -i "sine=frequency=880:duration=2.5" -af "aformat=sample_rates=44100,volume=0.6,vibrato=f=6:d=0.8,afade=t=out:st=1.5:d=1.0,asetrate=44100*0.85,aresample=44100" -y "$SFX_DIR/sad_violin.mp3"

# fail_horn.mp3 — descending "wah wah wah wahhh" brass
ffmpeg -f lavfi -i "sine=frequency=400:duration=2.0" -af "aformat=sample_rates=44100,volume=0.7,tremolo=f=3:d=0.7,afade=t=out:st=1.0:d=1.0,asetrate=44100*0.7,aresample=44100" -y "$SFX_DIR/fail_horn.mp3"

# laugh_track.mp3 — burst of filtered noise (crowd laughter approximation)
ffmpeg -f lavfi -i "anoisesrc=d=1.5:c=pink:r=44100" -af "highpass=f=800,lowpass=f=3000,volume=0.4,tremolo=f=8:d=0.6,afade=t=in:d=0.1,afade=t=out:st=0.8:d=0.7" -y "$SFX_DIR/laugh_track.mp3"

# dramatic_boom.mp3 — deep impact hit
ffmpeg -f lavfi -i "sine=frequency=60:duration=1.5" -af "aformat=sample_rates=44100,volume=1.0,afade=t=in:d=0.01,afade=t=out:st=0.3:d=1.2,aecho=0.8:0.7:30:0.5" -y "$SFX_DIR/dramatic_boom.mp3"

# oof.mp3 — short low thud
ffmpeg -f lavfi -i "sine=frequency=200:duration=0.4" -af "aformat=sample_rates=44100,volume=0.9,afade=t=in:d=0.01,afade=t=out:st=0.1:d=0.3,asetrate=44100*0.6,aresample=44100" -y "$SFX_DIR/oof.mp3"

# airhorn.mp3 — harsh high-frequency blast
ffmpeg -f lavfi -i "sine=frequency=800:duration=1.0" -af "aformat=sample_rates=44100,volume=0.7,overdrive=gain=20,afade=t=in:d=0.01,afade=t=out:st=0.5:d=0.5" -y "$SFX_DIR/airhorn.mp3"

# vine_boom.mp3 — deep reverb hit (the classic Vine boom)
ffmpeg -f lavfi -i "sine=frequency=80:duration=1.0" -af "aformat=sample_rates=44100,volume=1.0,afade=t=in:d=0.005,afade=t=out:st=0.2:d=0.8,aecho=0.8:0.6:15:0.7,aecho=0.8:0.5:40:0.4" -y "$SFX_DIR/vine_boom.mp3"
```

After generation, report:
```
SFX library generated (8 files):
  bruh.mp3, sad_violin.mp3, fail_horn.mp3, laugh_track.mp3,
  dramatic_boom.mp3, oof.mp3, airhorn.mp3, vine_boom.mp3

Note: These are synthetic approximations. For authentic meme sounds,
replace them with real clips from freesound.org or pixabay.com/sound-effects.
```

## Step 5: Summary

Report the final status:
- ffmpeg: OK / MISSING (mandatory)
- ffprobe: OK / MISSING (mandatory)
- Whisper: OK / NOT INSTALLED (optional)
- SFX library: N files / EMPTY (optional)
- Detection tier: Full (with Whisper) / Minimal (ffmpeg only)
