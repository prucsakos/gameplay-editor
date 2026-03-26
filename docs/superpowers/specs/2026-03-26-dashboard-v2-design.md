# Dashboard v2 Design Spec

**Date:** 2026-03-26
**Scope:** gameplay-editor plugin — dashboard-builder agent + HTML template

## Summary

Replace the agent-generated HTML dashboard with a pre-built static template. Add per-clip low-resolution video players and redesign buttons for clarity and efficiency.

---

## 1. Template Folder

A new `templates/` folder is added at the plugin root containing one file: `templates/dashboard.html`. This is a complete, self-contained HTML page with all CSS and JS inline — no external dependencies.

The agent never generates HTML from scratch. It reads the template, copies it to `tmp_dir`, and injects two placeholders:

- `MOMENTS_DATA_PLACEHOLDER` → JSON-serialized moments array
- `SERVER_PORT_PLACEHOLDER` → actual local server port (existing mechanism, unchanged)

**SKILL.md change:** Add to the "How It Works" section:
> The dashboard template lives in `templates/dashboard.html` (plugin root). The dashboard-builder agent reads it and injects moment data — it never generates HTML from scratch.

---

## 2. Video Extraction

`dashboard-builder.md` Step 1 adds a video extraction pass for each moment alongside the existing audio work.

**Command per clip:**
```bash
ffmpeg -ss <start> -t <duration> -i "<source_video>" \
  -vf scale=-2:300 \
  -c:v libx264 -crf 28 -preset fast \
  -c:a aac -b:a 64k \
  -y "<tmp_dir>/preview_<id>.mp4"
```

- Resolution: 300p height, auto width (`scale=-2:300`)
- Codec: H.264 / AAC, fast preset, CRF 28
- Output: `preview_<id>.mp4` per moment

**Audio-only MP3 previews (`preview_<id>.mp3`) are dropped.** The MP4 includes audio, making separate audio-only files redundant. Extended context clips (`preview_ext_<id>.mp3`) are kept as MP3 — extracting extended video would be costly and is not needed for trim decisions.

**Audio source tradeoff:** The MP4 uses audio from `source_video` (raw, undenoised). This is intentional — frame-accurate scrubbing for trim decisions matters more than audio clarity in previews. The extended MP3 (unchanged) still uses `clean_voice_tracks` for quality listening.

`source_video` is already an input to the agent — no new plumbing needed.

Progress reporting: `"Extracting video preview N/M..."`

---

## 3. Dashboard HTML Template (`templates/dashboard.html`)

### Page Guide (top of page, always visible)

A slim banner at the very top of the page. Non-collapsible, minimal height. Content:

```
Clip Curation Guide
Play a clip · Pause and stamp Start/End to trim · Keep or Remove · Add a note · Save & Close when done
```

Single line of text, small font, subtle background — does not compete with clip cards.

### Header

- Source filename + total clip count
- Tier summary: "N strong · M maybe"
- `[ Approve All ]` — small text button, right-aligned, less prominent than Save & Close

### Clip Cards

One card per moment, ordered by score descending.

**Top row:**
- Clip number (`#1`, `#2`, ...)
- Confidence badge: `STRONG` (green) or `MAYBE` (amber)
- Score (0–100)
- Timestamps (`HH:MM:SS -> HH:MM:SS`) and duration

**Video player:**
- Native `<video controls>` element, 100% card width
- Source: `preview_<id>.mp4`
- Plays inline, no autoplay

**Trim controls (below player):**
- `[ Set Start Here ]` — stamps `video.currentTime` as new clip start
- `[ Set End Here ]` — stamps `video.currentTime` as new clip end
- Live timestamp display between the buttons, updates immediately on stamp
- Clamped to context zone boundaries (`context_start` / `context_end` from moment data)

**Extended audio:** `preview_ext_<id>.mp3` files are retained on disk but not exposed in the v2 UI. Reserved for a future context-listening panel.

**Decision buttons (large, side by side):**
- `[ Keep ]` — green highlight when active (default for all clips)
- `[ Remove ]` — red highlight when active; card dims to 30% opacity on remove

**Comment field:**
- Single text input, full width
- Placeholder: `"Note for export agent..."`
- Saves to JS state on every keystroke

**Card visual states:**
- Strong (score > 70): full opacity, green left border
- Maybe (score <= 70): 80% opacity, amber left border
- Removed: 30% opacity, red left border (card stays fully visible — no collapse, the video player must remain accessible for scrubbing)

### Footer

- Running tally: `"X kept · Y removed · Z with comments"`
- `[ Save & Close ]` — large, centered, prominent

### JavaScript State

```javascript
const decisions = moments.map(m => ({
  id: m.id,
  action: 'keep',
  start: m.start,
  end: m.end,
  original_start: m.start,
  original_end: m.end,
  comment: ''
}));
```

**Set Start Here / Set End Here:** Reads `videoEl.currentTime`. The MP4 was extracted with `-ss moment.start`, so `currentTime = 0` corresponds to `moment.start`. Absolute timestamp = `moment.start + currentTime`. Clamp to `[context_start, context_end]` (context_start is used only as the lower clamp bound, not as the time origin). Update `decisions[i].start` or `.end` and the displayed timestamp label.

**Keep / Remove toggle:** Toggles `action`, updates card opacity and border color, updates footer tally.

**Comment field:** Writes to `decisions[i].comment` on every input event.

**Save & Close:** POSTs decisions JSON to `http://localhost:SERVER_PORT/save`. The payload includes a `boundary_adjusted` count, computed as the number of moments where `start !== original_start || end !== original_end`. On success, replaces page with a "Saved — return to terminal" confirmation. On failure, shows an alert with the error.

**Approve All:** Sets all decisions to `action: 'keep'`, then calls `saveAndClose()`.

---

## 4. dashboard-builder.md Changes

### Step 1 (Extract Previews)
- Add video extraction loop: 300p MP4 per clip using ffmpeg
- Remove core audio MP3 extraction
- Keep extended context MP3 extraction unchanged

### Step 2 (Generate Dashboard)
Replace "Generate a single self-contained HTML file" with:
> Read `templates/dashboard.html` from this plugin's `templates/` folder. The template path is resolved relative to this agent file: `<plugin_root>/templates/dashboard.html`, where `plugin_root` is derived via `os.path.dirname(os.path.dirname(os.path.abspath(__file__)))` or equivalent. If the file does not exist, abort immediately with: `"Dashboard template not found at <resolved_path>. Re-install the gameplay-editor plugin."` Do not fall back to generating HTML. Copy the template to `<tmp_dir>/dashboard.html`. Replace `MOMENTS_DATA_PLACEHOLDER` with JSON-serialized moments array. Replace `SERVER_PORT_PLACEHOLDER` with actual port.

### Step 3 (Server + Open)
Server logic, polling, and timeout are identical. One addition: after `saved.wait()`, the polling code reads `boundary_adjusted` from the saved JSON (computed in the browser and included in the POST body at top level alongside `moments`) and includes it in the printed summary alongside kept/removed/commented counts.

---

## 5. Files Changed

| File | Change |
|------|--------|
| `templates/dashboard.html` | **NEW** — pre-built self-contained dashboard |
| `agents/dashboard-builder.md` | Step 1 (video extraction), Step 2 (template injection), Step 3 (boundary_adjusted read-back) |
| `skills/gameplay-editor/SKILL.md` | One line noting the templates folder |
