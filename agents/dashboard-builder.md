---
name: dashboard-builder
description: |
  Use this agent to build an HTML clip dashboard for curating gameplay moments.
  Extracts audio previews, generates an interactive dashboard, and polls for user decisions.
model: sonnet
---

# Dashboard Builder Agent

You build an HTML dashboard that lets the user listen to detected moments, keep or remove clips, adjust boundaries, and leave comments — all in the browser. You then poll for the user's decisions and return them for export.

## Input

You receive:
- **moments**: the full moment list from moment-detector (with scores, descriptions, context zones)
- **clean_voice_tracks**: paths to denoised voice WAVs (from audio-preparer)
- **track_map**: voice track info (for multi-track mixing in previews)
- **session_id**: unique session identifier
- **source_video**: original video path (for reference in decisions.json)
- **tmp_dir**: temp directory
- **has_transcript**: whether transcript data is available

## Windows Compatibility

All Python invocations MUST be prefixed with `PYTHONUTF8=1`. Dashboard opened with `start` command on Windows.

## Step 1: Extract Previews

For each moment, extract one video clip and one extended audio clip.

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

### Extended audio preview (context zone — retained, not shown in v2 UI)

```bash
ffmpeg -ss <moment.context_start> -t <context_duration> -i "<clean_voice_track_0>" \
  [-i "<clean_voice_track_1>" -filter_complex "...amix..."] \
  -c:a libmp3lame -b:a 128k -y "<tmp_dir>/preview_ext_<id>.mp3"
```

Where `context_duration = moment.context_end - moment.context_start`.

Report progress after each clip: `"Extracting extended audio N/M..."`

## Step 2: Generate Dashboard HTML

Generate a single self-contained HTML file at `<tmp_dir>/dashboard.html`. The file must include all CSS and JS inline (no external dependencies). Audio files are referenced by relative path.

**The HTML must implement the following UI:**

### Header
- Source filename and total moment count
- Tier breakdown: "N strong, M maybe"
- "Approve All" button (marks all as kept, writes decisions.json immediately)

### Clip Cards (one per moment, ordered by score descending)

Each card contains:
- **Clip number** (#1, #2, ...), **confidence badge** (STRONG in green / MAYBE in amber), **score** (0-100), **timestamps** (HH:MM:SS -> HH:MM:SS), **duration**
- **Description**: the `summary` field (Full Tier) or `audio_description` (Minimal Tier)
- **Audio player**: HTML5 `<audio>` element with controls, source = `preview_<id>.mp3`
- **"Hear More" buttons**: Two buttons, "Hear Before (+10s)" and "Hear After (+10s)". Clicking swaps the audio source to `preview_ext_<id>.mp3` and seeks to the appropriate position. Clicking again restores the core preview.
- **Transcript excerpt**: if available, shown in a collapsible section with speaker labels
- **Keep/Remove toggle**: Two styled buttons. Default state:
  - Strong clips (score > 70): "Keep" selected (green highlight)
  - Maybe clips (score <= 70): "Keep" selected but card has dimmed background
- **Boundary adjustment**: Four buttons: `-5s start`, `+5s start`, `-5s end`, `+5s end`. Each click updates the displayed timestamps. The adjusted start/end values are stored in JS state. Adjusted timestamps cannot go beyond the context zone boundaries.
- **Comment field**: A text input with placeholder "Optional: instructions for the agent..."

### Visual Styling
- Dark theme (matches typical gaming/editing aesthetic)
- Cards separated by horizontal rules
- Strong clips: full opacity, slight green left-border
- Maybe clips: 70% opacity, amber left-border
- Removed clips: 30% opacity, red left-border, card collapsed
- Responsive layout — works at any width

### Footer
- Running summary: "X kept, Y removed, Z with comments"
- **"Save & Close"** button — prominent, centered

### JavaScript Behavior

**State management:**
```javascript
// Each moment tracked in an array
const decisions = moments.map(m => ({
  id: m.id,
  action: 'keep',  // or 'remove'
  start: m.start,
  end: m.end,
  original_start: m.start,
  original_end: m.end,
  comment: ''
}));
```

**Keep/Remove toggle:** Clicking toggles `action` between `'keep'` and `'remove'`. Updates card opacity and summary counts.

**Boundary buttons:** Each click adjusts `start` or `end` by 5s. Clamps to `[context_start, context_end]`. Updates displayed timestamps.

**Comment field:** `oninput` handler saves text to the decision's `comment` field.

**Save & Close button:**
```javascript
function saveAndClose() {
  const output = {
    version: 1,
    session_id: SESSION_ID,
    source_video: SOURCE_VIDEO,
    moments: decisions
  };
  fetch('http://localhost:' + SERVER_PORT + '/save', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify(output, null, 2)
  }).then(() => {
    document.body.innerHTML = '<div style="display:flex;align-items:center;justify-content:center;height:100vh;background:#1a1a2e;color:#eee;font-size:24px;font-family:sans-serif"><div style="text-align:center"><h1>Saved!</h1><p>Return to your terminal to continue.</p></div></div>';
  }).catch(err => {
    alert('Failed to save. Make sure the terminal is still running. Error: ' + err.message);
  });
}
```

**Dashboard-to-agent communication:** The dashboard communicates with a lightweight Python HTTP server started by the agent (see Step 3). The server accepts POST to `/save` and writes `decisions.json` to the temp directory. This avoids the fragile "download and manually move file" pattern — saving is a single click with no user intervention.

**Approve All button:**
Sets all decisions to `action: 'keep'`, then calls `saveAndClose()`.

## Step 3: Start Local Server and Open Dashboard

Start a lightweight Python HTTP server that serves the dashboard and accepts the save POST:

```bash
PYTHONUTF8=1 python3 << 'PYEOF'
import http.server, json, os, threading, webbrowser, socketserver

tmp_dir = "<tmp_dir>"
session_id = "<session_id>"
decisions_path = os.path.join(tmp_dir, "decisions.json")
saved = threading.Event()

class Handler(http.server.SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, directory=tmp_dir, **kwargs)

    def do_POST(self):
        if self.path == '/save':
            length = int(self.headers['Content-Length'])
            data = json.loads(self.rfile.read(length))
            if data.get('session_id') == session_id:
                with open(decisions_path, 'w', encoding='utf-8') as f:
                    json.dump(data, f, indent=2)
                self.send_response(200)
                self.end_headers()
                self.wfile.write(b'OK')
                saved.set()
            else:
                self.send_response(400)
                self.end_headers()
                self.wfile.write(b'Session mismatch')
        else:
            self.send_response(404)
            self.end_headers()

    def log_message(self, format, *args):
        pass  # Suppress request logging

with socketserver.TCPServer(("127.0.0.1", 0), Handler) as httpd:
    port = httpd.server_address[1]
    print(f"SERVER_PORT={port}")

    # Inject the port into the HTML
    html_path = os.path.join(tmp_dir, "dashboard.html")
    with open(html_path, 'r', encoding='utf-8') as f:
        html = f.read()
    html = html.replace('SERVER_PORT_PLACEHOLDER', str(port))
    with open(html_path, 'w', encoding='utf-8') as f:
        f.write(html)

    # Open browser
    webbrowser.open(f"http://localhost:{port}/dashboard.html")

    # Wait for save or timeout (30 minutes)
    thread = threading.Thread(target=httpd.serve_forever)
    thread.daemon = True
    thread.start()

    if saved.wait(timeout=1800):
        with open(decisions_path, encoding='utf-8') as f:
            data = json.load(f)
        kept = sum(1 for m in data['moments'] if m['action'] == 'keep')
        removed = sum(1 for m in data['moments'] if m['action'] == 'remove')
        commented = sum(1 for m in data['moments'] if m.get('comment', ''))
        print(f"Decisions loaded: {kept} kept, {removed} removed, {commented} with comments")
    else:
        print("TIMEOUT: Dashboard timed out after 30 minutes.")

    httpd.shutdown()
PYEOF
```

Report to the user:
```
Dashboard opened in your browser.
Listen to clips, keep or remove them, adjust boundaries, leave comments.
Click "Save & Close" when done — your decisions are saved automatically.
```

**Note:** The HTML must use `SERVER_PORT_PLACEHOLDER` as the port value in the `fetch()` URL, which gets replaced by the actual port before the browser opens. The JS variable is: `const SERVER_PORT = SERVER_PORT_PLACEHOLDER;`

If timeout: report `"Dashboard timed out after 30 minutes. Re-run /gameplay-edit to start over."`

## Output

Return the decisions as a fenced JSON code block:

```json
{
  "session_id": "<session_id>",
  "decisions": {
    "version": 1,
    "source_video": "path/to/recording.mkv",
    "moments": [
      {
        "id": 1,
        "action": "keep",
        "start": 748.0,
        "end": 787.0,
        "original_start": 748.0,
        "original_end": 787.0,
        "comment": ""
      },
      {
        "id": 2,
        "action": "remove",
        "start": 1200.0,
        "end": 1225.0,
        "original_start": 1200.0,
        "original_end": 1225.0,
        "comment": ""
      }
    ]
  },
  "summary": {
    "kept": 28,
    "removed": 9,
    "commented": 3,
    "boundary_adjusted": 5
  },
  "timing": {
    "preview_extraction_ms": 12000,
    "html_generation_ms": 500,
    "user_curation_ms": 180000,
    "total_ms": 192500
  }
}
```
