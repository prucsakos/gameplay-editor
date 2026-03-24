---
name: clean
transitions: fade
transition_duration: 0.3
audio_normalization: standard
aspect_ratio: per-platform
export_codec: libx264
export_quality: 20
transcript_signals:
  - humor
  - dramatic reactions
  - surprised exclamations
  - banter
  - storytelling peaks
  - emotional outbursts
  - comedic timing
shorts_count: 3
shorts_duration_range: [15, 60]
detection_threshold: 30
---

# Clean Style

Minimal editing. Content speaks for itself.

Hard cuts with subtle fade transitions (0.3s). No overlays or effects.

Audio pipeline is always-on: noise reduction, per-track normalization,
dynamic compression, and game audio leveling are applied automatically.

Aspect ratio and audio LUFS are determined by the target platform (youtube or tiktok).
