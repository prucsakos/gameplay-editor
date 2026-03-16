---
description: One-time setup for gameplay-editor — installs faster-whisper, downloads RNNoise model, and verifies ffmpeg and CUDA
allowed-tools: Bash(ffmpeg:*), Bash(ffprobe:*), Bash(pip:*), Bash(pip3:*), Bash(python:*), Bash(python3:*), Bash(where:*), Bash(which:*), Bash(ls:*), Bash(mkdir:*), Bash(curl:*)
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
"Python 3.8+ is required for faster-whisper transcription. Install from python.org. Without Python, the plugin will use volume-based analysis only (reduced accuracy)."
Continue even if Python is missing — it is optional.

## Step 3: Download RNNoise Model

The audio pipeline uses ffmpeg's `arnndn` filter with an RNNoise neural network model for noise reduction.

Check if the model exists:
```bash
ls "$HOME/.cache/gameplay-editor/sh.rnnn" 2>/dev/null && echo "RNNoise model: OK" || echo "MISSING"
```

If missing, download it:
```bash
mkdir -p "$HOME/.cache/gameplay-editor"
curl -sL "https://github.com/richardpl/arnndn-models/raw/master/sh.rnnn" -o "$HOME/.cache/gameplay-editor/sh.rnnn"
```

Verify the download succeeded (file should be ~1MB):
```bash
ls -lh "$HOME/.cache/gameplay-editor/sh.rnnn"
```

Also verify ffmpeg supports the `arnndn` filter:
```bash
ffmpeg -filters 2>/dev/null | grep arnndn
```

If the filter is not found, tell the user: "Your ffmpeg build does not include the arnndn filter. Install a full build: winget install Gyan.FFmpeg (Windows), brew install ffmpeg (macOS), or apt install ffmpeg (Linux)."

## Step 4: Install faster-whisper

If Python is available, run:
```bash
pip install faster-whisper
```
Or `pip3 install faster-whisper` if `pip` is not available.

If installation succeeds, verify by running:
```bash
PYTHONUTF8=1 python3 -c "from faster_whisper import WhisperModel; print('faster-whisper OK')"
```
Or use `python` if `python3` is not available.

If import works, report: "faster-whisper installed successfully. The small model (~500MB) will download automatically on first use."

If installation fails, report the error and tell the user: "faster-whisper installation failed. The plugin will use volume-based analysis only. You can retry later with /gameplay-setup."

## Step 4b: Verify CUDA availability

faster-whisper uses CTranslate2 as its backend. Check GPU availability:

```bash
PYTHONUTF8=1 python3 -c "
try:
    import ctranslate2
    print(f'CTranslate2 {ctranslate2.__version__}')
    cuda = 'cuda' in ctranslate2.get_supported_compute_types('cuda')
    print(f'CUDA available: {cuda}')
except Exception as e:
    print(f'CTranslate2 check: {e}')

try:
    import torch
    print(f'PyTorch {torch.__version__}')
    if torch.cuda.is_available():
        print(f'GPU: {torch.cuda.get_device_name(0)}')
    else:
        print('GPU: none (CPU mode)')
except ImportError:
    print('PyTorch not installed (optional for faster-whisper)')
"
```

Report the result:
- If CUDA is available: "CUDA detected — faster-whisper will use GPU acceleration."
- If CUDA is not available: "CPU-only mode — faster-whisper will run on CPU (slower but functional). For GPU acceleration, install CUDA toolkit and reinstall: pip install faster-whisper[cuda]"

## Step 5: Summary

Report the final status:
- ffmpeg: OK / MISSING (mandatory)
- ffprobe: OK / MISSING (mandatory)
- arnndn filter: OK / MISSING (mandatory)
- RNNoise model: OK / MISSING (mandatory)
- Python: OK (<version>) / MISSING (optional)
- faster-whisper: OK / NOT INSTALLED (optional)
- CUDA/GPU: OK (<GPU name>) / CPU-only / N/A
- Detection tier: Full (with faster-whisper) / Minimal (ffmpeg only)
