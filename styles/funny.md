---
name: funny
transitions: cut
transition_duration: 0
text_overlays: suggested
funny_edits: true
audio_normalization: standard
aspect_ratio: per-platform
export_codec: libx264
export_quality: crf_20
---

# Funny Style

Meme-style editing with AI-suggested edits. All suggestions are presented inline
in the moment plan for user approval — nothing is added without consent.

## Edit Types

- **Text pop-ups**: Big bold text overlay (1-2s) when Whisper detects dramatic/funny
  phrases (exclamations, ALL CAPS, repeated letters like "NEEEEM!")
- **Sound effects**: SFX from bundled library (`assets/sfx/`) or user library (`sfx/`
  next to source video). Agent picks contextually — sad_violin for fails,
  dramatic_boom for comebacks, etc.
- **Slow-mo**: 0.5x speed for 2-3s around peak loudness spikes. Audio uses comedic
  pitch drop (uncorrected `setpts=2*PTS`) for humor.
- **Replay**: Instant replay of peak 2-3s at 0.7x speed with flash transition.
  Only suggested for top-scoring moments (score > 90).

## Word Counter

Available in this style (and clean). After transcription, any word appearing 10+ times
is flagged. If user approves, a running counter pop-up renders each time the word
is spoken, using Whisper word-level timestamps.

Hard cuts, no transitions. Aspect ratio and audio LUFS determined by target platform.
