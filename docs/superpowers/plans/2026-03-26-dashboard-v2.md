# Dashboard v2 Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the agent-generated HTML dashboard with a pre-built static template that includes per-clip 300p video players, playhead-based trim stamping, and clear Keep/Remove buttons.

**Architecture:** A static `templates/dashboard.html` lives in the plugin repo with two string placeholders (`MOMENTS_DATA_PLACEHOLDER`, `SERVER_PORT_PLACEHOLDER`). The dashboard-builder agent copies it to the temp directory and injects runtime data via simple string replacement. Clip video previews are extracted per-moment at 300p using ffmpeg from the source video.

**Tech Stack:** HTML/CSS/JS (vanilla, inline, no external dependencies), ffmpeg (video extraction), Python (HTTP server — unchanged), Markdown (agent instruction files)

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `templates/dashboard.html` | **CREATE** | Complete self-contained dashboard — all CSS and JS inline, two string placeholders for runtime injection |
| `agents/dashboard-builder.md` | **MODIFY** | Step 1: swap MP3 core extraction with MP4 video; Step 2: template injection; Step 3: boundary_adjusted read-back |
| `skills/gameplay-editor/SKILL.md` | **MODIFY** | Add one line noting the templates/ folder |

---

## Chunk 1: templates/dashboard.html

### Task 1: Create the templates/ folder and dashboard.html template

**Files:**
- Create: `templates/dashboard.html`

**Placeholder contract:**
- `MOMENTS_DATA_PLACEHOLDER` — replaced by a JSON object: `{ session_id, source_video, moments[] }`
- `SERVER_PORT_PLACEHOLDER` — replaced by the integer port number (existing mechanism, unchanged)

The JS reads the injected config as `const CONFIG = MOMENTS_DATA_PLACEHOLDER;`, then extracts `CONFIG.session_id`, `CONFIG.source_video`, `CONFIG.moments`.

All card DOM construction uses `createElement` / `textContent` / `appendChild` — no innerHTML.

- [ ] **Step 1: Create `templates/dashboard.html`**

Create the file at:
`C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0/templates/dashboard.html`

Full content — copy exactly:

```
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Clip Curation Dashboard</title>
<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: 'Segoe UI', system-ui, sans-serif; background: #111; color: #ddd; min-height: 100vh; }

#guide {
  background: #1a221a;
  border-bottom: 1px solid #253525;
  padding: 7px 20px;
  font-size: 12px;
  color: #7aaa7a;
  position: sticky;
  top: 0;
  z-index: 100;
}
#guide strong { color: #9aca9a; }

#page-header {
  padding: 14px 20px;
  background: #161616;
  border-bottom: 1px solid #222;
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 8px;
}
#page-header h1 { font-size: 15px; font-weight: 600; color: #eee; }
.meta { font-size: 12px; color: #777; margin-top: 2px; }
#approve-all {
  background: none;
  border: 1px solid #383838;
  color: #777;
  padding: 4px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 11px;
}
#approve-all:hover { border-color: #555; color: #aaa; }

#cards { padding: 16px 20px; max-width: 860px; margin: 0 auto; display: flex; flex-direction: column; gap: 14px; }

.card {
  background: #1a1a1a;
  border-radius: 7px;
  border-left: 4px solid #2a5a2a;
  overflow: hidden;
  transition: opacity 0.2s;
}
.card.maybe   { border-left-color: #6a4e00; opacity: 0.80; }
.card.removed { border-left-color: #5a1818; opacity: 0.3;  }

.card-header {
  padding: 9px 13px;
  display: flex;
  align-items: center;
  gap: 9px;
  flex-wrap: wrap;
  background: #161616;
}
.clip-num { font-size: 13px; font-weight: 700; color: #ccc; min-width: 28px; }
.badge {
  font-size: 10px;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 3px;
  letter-spacing: 0.05em;
}
.badge.strong { background: #183018; color: #6aca6a; }
.badge.maybe  { background: #32240a; color: #caa020; }
.score { font-size: 12px; color: #666; }
.timestamps { font-size: 12px; color: #999; font-variant-numeric: tabular-nums; margin-left: auto; }

.card video {
  width: 100%;
  display: block;
  background: #000;
  max-height: 220px;
}

.trim-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 7px 13px;
  background: #131313;
}
.trim-btn {
  background: #222;
  border: 1px solid #333;
  color: #bbb;
  padding: 5px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  white-space: nowrap;
  flex-shrink: 0;
}
.trim-btn:hover { background: #2a2a2a; border-color: #505050; color: #eee; }
.trim-label {
  font-size: 11px;
  color: #777;
  font-variant-numeric: tabular-nums;
  flex: 1;
  text-align: center;
}

.decision-row { display: flex; gap: 8px; padding: 8px 13px; }
.btn-keep, .btn-remove {
  flex: 1;
  padding: 9px 0;
  border: 2px solid transparent;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 0.02em;
  transition: all 0.12s;
}
.btn-keep          { background: #182818; border-color: #285828; color: #5aba5a; }
.btn-keep:hover    { background: #1d331d; border-color: #3a7a3a; }
.btn-keep.active   { background: #1f4a1f; border-color: #4a9a4a; color: #80d880; }
.btn-remove        { background: #281818; border-color: #582828; color: #ba5a5a; }
.btn-remove:hover  { background: #331d1d; border-color: #7a3a3a; }
.btn-remove.active { background: #4a1f1f; border-color: #9a4a4a; color: #d88080; }

.comment-row { padding: 0 13px 10px; }
.comment-row input {
  width: 100%;
  background: #131313;
  border: 1px solid #252525;
  color: #bbb;
  padding: 6px 10px;
  border-radius: 4px;
  font-size: 12px;
  font-family: inherit;
}
.comment-row input:focus { outline: none; border-color: #3a3a3a; }
.comment-row input::placeholder { color: #484848; }

#footer {
  position: sticky;
  bottom: 0;
  background: #161616;
  border-top: 1px solid #222;
  padding: 11px 20px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  z-index: 100;
}
#tally { font-size: 13px; color: #777; }
#save-btn {
  background: #183a18;
  border: 2px solid #2a6a2a;
  color: #80d880;
  padding: 9px 36px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 700;
  transition: all 0.12s;
}
#save-btn:hover { background: #1f4d1f; border-color: #3a8a3a; color: #9ae89a; }

#saved-screen {
  display: none;
  position: fixed;
  inset: 0;
  background: #111;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 14px;
  color: #80d880;
  font-size: 20px;
}
#saved-screen p { color: #666; font-size: 14px; }
</style>
</head>
<body>

<div id="guide">
  <strong>Clip Curation Guide</strong> &nbsp;&middot;&nbsp;
  Play a clip &nbsp;&middot;&nbsp;
  Pause and stamp <strong>Set Start / End Here</strong> to trim &nbsp;&middot;&nbsp;
  <strong>Keep</strong> or <strong>Remove</strong> &nbsp;&middot;&nbsp;
  Add a note &nbsp;&middot;&nbsp;
  <strong>Save &amp; Close</strong> when done
</div>

<div id="page-header">
  <div>
    <h1 id="source-name"></h1>
    <div class="meta" id="tier-summary"></div>
  </div>
  <button id="approve-all" onclick="approveAll()">Approve All</button>
</div>

<div id="cards"></div>

<div id="footer">
  <span id="tally"></span>
  <button id="save-btn" onclick="saveAndClose()">Save &amp; Close</button>
</div>

<div id="saved-screen">
  <span>Saved.</span>
  <p>Return to your terminal to continue.</p>
</div>

<script>
var SERVER_PORT  = SERVER_PORT_PLACEHOLDER;
var CONFIG       = MOMENTS_DATA_PLACEHOLDER;
var SESSION_ID   = CONFIG.session_id;
var SOURCE_VIDEO = CONFIG.source_video;
var moments      = CONFIG.moments;

// Sort by score descending — must happen before decisions is built so positional idx is consistent
moments.sort(function(a, b) { return b.score - a.score; });

var decisions = moments.map(function(m) {
  return { id: m.id, action: 'keep', start: m.start, end: m.end,
           original_start: m.start, original_end: m.end, comment: '' };
});

function fmt(s) {
  var h  = Math.floor(s / 3600);
  var m  = Math.floor((s % 3600) / 60);
  var sc = Math.floor(s % 60);
  return pad(h) + ':' + pad(m) + ':' + pad(sc);
}
function pad(n) { return n < 10 ? '0' + n : String(n); }

function updateTally() {
  var kept      = decisions.filter(function(d) { return d.action === 'keep'; }).length;
  var removed   = decisions.filter(function(d) { return d.action === 'remove'; }).length;
  var commented = decisions.filter(function(d) { return d.comment.trim(); }).length;
  document.getElementById('tally').textContent =
    kept + ' kept \u00b7 ' + removed + ' removed \u00b7 ' + commented + ' with comments';
}

function setDecision(idx, action) {
  decisions[idx].action = action;
  var card      = document.getElementById('card-' + idx);
  var isStrong  = moments[idx].score > 70;
  var cls = 'card';
  if (!isStrong)        cls += ' maybe';
  if (action === 'remove') cls += ' removed';
  card.className = cls;
  card.querySelector('.btn-keep').classList.toggle('active', action === 'keep');
  card.querySelector('.btn-remove').classList.toggle('active', action === 'remove');
  updateTally();
}

function setStart(idx) {
  var video = document.getElementById('video-' + idx);
  var m     = moments[idx];
  var abs   = m.start + video.currentTime;
  abs = Math.max(m.context_start, Math.min(m.context_end, abs));
  decisions[idx].start = abs;
  document.getElementById('trim-label-' + idx).textContent =
    fmt(abs) + ' \u2192 ' + fmt(decisions[idx].end);
}

function setEnd(idx) {
  var video = document.getElementById('video-' + idx);
  var m     = moments[idx];
  var abs   = m.start + video.currentTime;
  abs = Math.max(m.context_start, Math.min(m.context_end, abs));
  decisions[idx].end = abs;
  document.getElementById('trim-label-' + idx).textContent =
    fmt(decisions[idx].start) + ' \u2192 ' + fmt(abs);
}

function buildCards() {
  var sourceName = (SOURCE_VIDEO || '').split(/[\/\\]/).pop();
  document.getElementById('source-name').textContent = sourceName || 'Unknown source';
  var strong = moments.filter(function(m) { return m.score > 70; }).length;
  var maybe  = moments.length - strong;
  document.getElementById('tier-summary').textContent =
    moments.length + ' clips \u00b7 ' + strong + ' strong \u00b7 ' + maybe + ' maybe';

  var container = document.getElementById('cards');

  moments.forEach(function(m, idx) {
    var isStrong = m.score > 70;
    var dur = (m.end - m.start).toFixed(1);

    /* Card */
    var card = document.createElement('div');
    card.id        = 'card-' + idx;
    card.className = 'card' + (!isStrong ? ' maybe' : '');

    /* Header row */
    var hdr = document.createElement('div');
    hdr.className = 'card-header';

    var clipNum = document.createElement('span');
    clipNum.className   = 'clip-num';
    clipNum.textContent = '#' + (idx + 1);

    var badge = document.createElement('span');
    badge.className   = 'badge ' + (isStrong ? 'strong' : 'maybe');
    badge.textContent = isStrong ? 'STRONG' : 'MAYBE';

    var score = document.createElement('span');
    score.className   = 'score';
    score.textContent = String(m.score);

    var ts = document.createElement('span');
    ts.className   = 'timestamps';
    ts.textContent = fmt(m.start) + ' \u2192 ' + fmt(m.end) + '   ' + dur + 's';

    hdr.appendChild(clipNum);
    hdr.appendChild(badge);
    hdr.appendChild(score);
    hdr.appendChild(ts);

    /* Video */
    var video = document.createElement('video');
    video.id      = 'video-' + idx;
    video.controls = true;
    video.preload  = 'metadata';
    var vsrc = document.createElement('source');
    vsrc.src  = 'preview_' + m.id + '.mp4';
    vsrc.type = 'video/mp4';
    video.appendChild(vsrc);

    /* Trim row */
    var trimRow = document.createElement('div');
    trimRow.className = 'trim-row';

    var btnStart = document.createElement('button');
    btnStart.className   = 'trim-btn';
    btnStart.textContent = '\u2702 Set Start Here';
    btnStart.setAttribute('data-idx', idx);
    btnStart.onclick = function() { setStart(parseInt(this.getAttribute('data-idx'), 10)); };

    var trimLabel = document.createElement('span');
    trimLabel.className   = 'trim-label';
    trimLabel.id          = 'trim-label-' + idx;
    trimLabel.textContent = fmt(m.start) + ' \u2192 ' + fmt(m.end);

    var btnEnd = document.createElement('button');
    btnEnd.className   = 'trim-btn';
    btnEnd.textContent = 'Set End Here \u2702';
    btnEnd.setAttribute('data-idx', idx);
    btnEnd.onclick = function() { setEnd(parseInt(this.getAttribute('data-idx'), 10)); };

    trimRow.appendChild(btnStart);
    trimRow.appendChild(trimLabel);
    trimRow.appendChild(btnEnd);

    /* Decision row */
    var decRow = document.createElement('div');
    decRow.className = 'decision-row';

    var btnKeep = document.createElement('button');
    btnKeep.className   = 'btn-keep active';
    btnKeep.textContent = '\u2713 Keep';
    btnKeep.setAttribute('data-idx', idx);
    btnKeep.onclick = function() { setDecision(parseInt(this.getAttribute('data-idx'), 10), 'keep'); };

    var btnRemove = document.createElement('button');
    btnRemove.className   = 'btn-remove';
    btnRemove.textContent = '\u2717 Remove';
    btnRemove.setAttribute('data-idx', idx);
    btnRemove.onclick = function() { setDecision(parseInt(this.getAttribute('data-idx'), 10), 'remove'); };

    decRow.appendChild(btnKeep);
    decRow.appendChild(btnRemove);

    /* Comment row */
    var commentRow = document.createElement('div');
    commentRow.className = 'comment-row';

    var input = document.createElement('input');
    input.type        = 'text';
    input.placeholder = 'Note for export agent...';
    input.setAttribute('data-idx', idx);
    input.addEventListener('input', function() {
      decisions[parseInt(this.getAttribute('data-idx'), 10)].comment = this.value;
      updateTally();
    });
    commentRow.appendChild(input);

    card.appendChild(hdr);
    card.appendChild(video);
    card.appendChild(trimRow);
    card.appendChild(decRow);
    card.appendChild(commentRow);
    container.appendChild(card);
  });
}

function approveAll() {
  moments.forEach(function(_, idx) { setDecision(idx, 'keep'); });
  saveAndClose();
}

function saveAndClose() {
  var boundaryAdjusted = decisions.filter(function(d) {
    return d.start !== d.original_start || d.end !== d.original_end;
  }).length;
  var payload = {
    version: 1,
    session_id: SESSION_ID,
    source_video: SOURCE_VIDEO,
    boundary_adjusted: boundaryAdjusted,
    moments: decisions
  };
  fetch('http://localhost:' + SERVER_PORT + '/save', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload, null, 2)
  }).then(function(r) {
    if (!r.ok) throw new Error('Server returned ' + r.status);
    var s = document.getElementById('saved-screen');
    s.style.display = 'flex';
    document.getElementById('footer').style.display = 'none';
  }).catch(function(err) {
    alert('Failed to save. Make sure the terminal is still running.\n\n' + err.message);
  });
}

buildCards();
updateTally();
</script>
</body>
</html>
```

- [ ] **Step 2: Verify the file was created**

```bash
ls -la "C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0/templates/"
```

Expected: `dashboard.html` is listed.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0"
git add templates/dashboard.html
git commit -m "feat: add dashboard v2 static template with video players and trim controls"
```

---

## Chunk 2: dashboard-builder.md — Step 1 (video extraction)

### Task 2: Replace MP3 core preview extraction with per-clip MP4 video extraction

**Files:**
- Modify: `agents/dashboard-builder.md` — Step 1 section

Replace the "Core preview" block (both single-track and multi-track MP3 variants) with a single MP4 extraction command. The "Extended preview" MP3 block is kept unchanged.

- [ ] **Step 1: Replace the Step 1 header and core preview block**

In `agents/dashboard-builder.md`, use the Edit tool to make two targeted replacements:

**Replacement A** — change the section heading and opening sentence:

Replace the exact line:
    ## Step 1: Extract Audio Previews

With:
    ## Step 1: Extract Previews

Then replace the exact line:
    For each moment, extract two audio clips from the clean voice tracks:

With:
    For each moment, extract one video clip and one extended audio clip.

**Replacement B** — replace the "Core preview" sub-section heading and its two ffmpeg command blocks.

The sub-section to replace starts at this exact heading:
    ### Core preview (the detected moment)

And ends just before this line (do not include or remove this line):
    ### Extended preview (context zone — for "hear more")

Replace everything between those two anchors with the following text (the new sub-section heading + ffmpeg command + notes):

    ### Video preview (the detected moment)

    ```bash
    ffmpeg -ss <moment.start> -t <moment.duration> -i "<source_video>" \
      -vf scale=-2:300 \
      -c:v libx264 -crf 28 -preset fast \
      -c:a aac -b:a 64k \
      -y "<tmp_dir>/preview_<id>.mp4"
    ```

    - Resolution: 300p height, auto width (`scale=-2:300`)
    - Codec: H.264/AAC, fast preset, CRF 28 — plays in all modern browsers
    - Audio is raw (undenoised) from source_video — acceptable for scrubbing
    - Output: `preview_<id>.mp4`

    Report progress after each clip: `"Extracting video preview N/M..."`

Also update the `### Extended preview` heading to:
    ### Extended audio preview (context zone — retained, not shown in v2 UI)

- [ ] **Step 2: Update the progress report line inside the extended preview block**

Still in `agents/dashboard-builder.md`, replace:
```
Report progress: `"Extracting audio preview N/M..."`
```
With:
```
Report progress after each clip: `"Extracting extended audio N/M..."`
```

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0"
git add agents/dashboard-builder.md
git commit -m "feat: dashboard-builder Step 1 — replace MP3 extraction with 300p MP4 per clip"
```

---

## Chunk 3: dashboard-builder.md — Step 2 (template injection)

### Task 3: Replace "Generate Dashboard HTML" with template read and injection

**Files:**
- Modify: `agents/dashboard-builder.md` — Step 2 section

Replace the entire `## Step 2: Generate Dashboard HTML` section (the large HTML spec block) with a short template-injection instruction.

- [ ] **Step 1: Replace the entire Step 2 section**

In `agents/dashboard-builder.md`, delete everything from the heading `## Step 2: Generate Dashboard HTML` up to (but not including) the heading `## Step 3: Start Local Server and Open Dashboard`. Replace with the following new section (copy verbatim):

---

    ## Step 2: Inject Data Into Dashboard Template

    Read `templates/dashboard.html` from this plugin's `templates/` folder.

    Resolve the template path relative to this agent file:

        import os, json
        plugin_root   = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
        template_path = os.path.join(plugin_root, 'templates', 'dashboard.html')

    If the file does not exist, abort immediately:

        Dashboard template not found at <template_path>. Re-install the gameplay-editor plugin.

    Do NOT fall back to generating HTML.

    Copy to the temp directory, then inject the config object:

        import shutil
        html_path = os.path.join(tmp_dir, 'dashboard.html')
        shutil.copy(template_path, html_path)

        config = {
            "session_id":   session_id,
            "source_video": source_video,
            "moments":      moments_list   # full list from moment-detector
        }

        with open(html_path, 'r', encoding='utf-8') as f:
            html = f.read()

        html = html.replace('MOMENTS_DATA_PLACEHOLDER', json.dumps(config, ensure_ascii=False))
        # SERVER_PORT_PLACEHOLDER is replaced in Step 3 after the port is known

        with open(html_path, 'w', encoding='utf-8') as f:
            f.write(html)

    **Minimum required fields per moment in `moments_list`:** `id`, `score`, `start`, `end`, `context_start`, `context_end`. All are present in the moment-detector output.

---

- [ ] **Step 2: Update the agent frontmatter description**

In `agents/dashboard-builder.md`, update the `description` field to:

```yaml
description: |
  Use this agent to build an HTML clip dashboard for curating gameplay moments.
  Extracts per-clip 300p video previews, loads a pre-built dashboard template, and polls for user decisions.
```

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0"
git add agents/dashboard-builder.md
git commit -m "feat: dashboard-builder Step 2 — replace HTML generation with template injection"
```

---

## Chunk 4: dashboard-builder.md — Step 3 (boundary_adjusted read-back)

### Task 4: Add boundary_adjusted to the polling summary

**Files:**
- Modify: `agents/dashboard-builder.md` — Step 3 polling block

- [ ] **Step 1: Update the polling summary block**

In `agents/dashboard-builder.md`, replace:

```python
        kept = sum(1 for m in data['moments'] if m['action'] == 'keep')
        removed = sum(1 for m in data['moments'] if m['action'] == 'remove')
        commented = sum(1 for m in data['moments'] if m.get('comment', ''))
        print(f"Decisions loaded: {kept} kept, {removed} removed, {commented} with comments")
```

With:

```python
        kept             = sum(1 for m in data['moments'] if m['action'] == 'keep')
        removed          = sum(1 for m in data['moments'] if m['action'] == 'remove')
        commented        = sum(1 for m in data['moments'] if m.get('comment', ''))
        boundary_adjusted = data.get('boundary_adjusted', 0)
        print(f"Decisions loaded: {kept} kept, {removed} removed, {commented} with comments, {boundary_adjusted} boundaries adjusted")
```

- [ ] **Step 2: Commit**

```bash
cd "C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0"
git add agents/dashboard-builder.md
git commit -m "feat: dashboard-builder Step 3 — add boundary_adjusted to polling summary"
```

---

## Chunk 5: SKILL.md update

### Task 5: Add templates/ folder note to SKILL.md

**Files:**
- Modify: `skills/gameplay-editor/SKILL.md`

- [ ] **Step 1: Add the templates/ line to the "How It Works" section**

In `skills/gameplay-editor/SKILL.md`, use the Edit tool to replace this exact text:

    - **Phase 4 (Export):** export-assembler agent — masters audio, assembles highlight reel + auto-generates shorts

With:

    - **Phase 4 (Export):** export-assembler agent — masters audio, assembles highlight reel + auto-generates shorts

    The dashboard template lives in `templates/dashboard.html` (plugin root). The dashboard-builder agent reads it and injects moment data — it never generates HTML from scratch.

- [ ] **Step 2: Commit**

```bash
cd "C:/Users/prucs/.claude/plugins/cache/prucsakos/gameplay-editor/3.0.0"
git add skills/gameplay-editor/SKILL.md
git commit -m "docs: note templates/ folder in SKILL.md"
```

---

## Verification Checklist

After all tasks complete:

- [ ] `templates/dashboard.html` exists at plugin root
- [ ] Opening `templates/dashboard.html` in a browser shows the guide banner, header, and footer — no JS console errors
- [ ] With mock data injected, clip cards render in score-descending order (highest score first)
- [ ] `decisions.json` payload includes `boundary_adjusted` at the top level (not nested inside `moments`)
- [ ] `agents/dashboard-builder.md` frontmatter description mentions "300p video previews"
- [ ] `agents/dashboard-builder.md` Step 1 references `preview_<id>.mp4`, not `.mp3`
- [ ] `agents/dashboard-builder.md` Step 2 reads template, no HTML generation block
- [ ] `agents/dashboard-builder.md` Step 3 prints `boundary_adjusted` count
- [ ] `skills/gameplay-editor/SKILL.md` mentions `templates/dashboard.html`
- [ ] `git log --oneline` shows 5 new commits from this work
