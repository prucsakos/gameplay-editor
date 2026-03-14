---
name: shortform
transitions: cut
transition_duration: 0
text_overlays: captions
audio_normalization: loud
target_lufs: -12
voice_level: 0
game_level: -15
aspect_ratio: "9:16"
crop_mode: center
crop_offset_x: 0
export_codec: libx264
export_quality: crf_20
max_duration: 60
---

# Shortform Style

Optimized for TikTok, YouTube Shorts, and Instagram Reels.
Hard cuts, no transitions. Auto-generated word-level captions
from Whisper transcript burned onto video.
9:16 aspect ratio via smart crop from source frame.
Loud audio normalization to -12 LUFS for mobile viewing.
Default max duration: 60 seconds.

## Crop Configuration

Adjust `crop_offset_x` to shift the crop window horizontally.
- `0` = center of frame (default)
- Positive values shift right, negative shift left
- Or set `crop_mode: custom` and specify exact pixel coordinates

For most party games, center crop works well since UI is centered.
