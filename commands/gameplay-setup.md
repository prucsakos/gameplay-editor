---
description: One-time setup for gameplay-editor — installs Whisper and verifies ffmpeg, torch, and CUDA
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(pip:*), Bash(pip3:*), Bash(python:*), Bash(python3:*), Bash(whisper:*), Bash(where:*), Bash(which:*), Bash(ls:*), Bash(mkdir:*)
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

If `whisper` CLI works, report: "Whisper installed successfully. The base model (~74MB) will download automatically on first use."

If installation fails, report the error and tell the user: "Whisper installation failed. The plugin will use volume-based analysis only. You can retry later with /gameplay-setup."

## Step 3b: Verify PyTorch and CUDA

PyTorch (torch) is a hard dependency of openai-whisper — Whisper cannot run without it. If Whisper installed successfully, torch is already present. Verify and check GPU availability:

```bash
python -c "import torch; print(f'PyTorch {torch.__version__}'); print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}' if torch.cuda.is_available() else 'GPU: none (CPU mode)')"
```
Or use `python3` if `python` is not available.

Report the result:
- If CUDA is available: "PyTorch <version> with CUDA — Whisper will use GPU acceleration (faster transcription)."
- If CUDA is not available: "PyTorch <version> CPU-only — Whisper will run on CPU (slower). For GPU acceleration, install the CUDA version of PyTorch: visit https://pytorch.org/get-started/locally/ and select your CUDA version."

If torch import fails but whisper installed, something went wrong — report: "PyTorch not found despite Whisper being installed. Try: pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121 (for CUDA 12.1) or pip install torch (CPU-only)."

## Step 4: Summary

Report the final status:
- ffmpeg: OK / MISSING (mandatory)
- ffprobe: OK / MISSING (mandatory)
- Python: OK (<version>) / MISSING (optional)
- PyTorch: OK (<version>) / NOT INSTALLED (optional)
- CUDA/GPU: OK (<GPU name>) / CPU-only / N/A
- Whisper: OK / NOT INSTALLED (optional)
- Detection tier: Full (with Whisper) / Minimal (ffmpeg only)
