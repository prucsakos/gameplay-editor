---
name: polished
transitions: dissolve
transition_duration: 0.5
text_overlays: context
audio_normalization: standard
target_lufs: -14
voice_level: 0
game_level: -12
aspect_ratio: original
crop_mode: center
export_codec: libx264
export_quality: crf_20
max_duration: null
---

# Polished Style

YouTube-ready editing. Dissolve transitions (0.5s) between moments.
Context text overlays show elapsed source time (e.g., "20 minutes in...")
when there is a gap >5 minutes between moments. Speed ramps (4x) bridge
moments that are less than 2 minutes apart in source.
Audio normalized to -14 LUFS.
