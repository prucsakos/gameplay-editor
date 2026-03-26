---
name: dashboard-builder
description: |
  Use this agent to build an HTML clip dashboard for curating gameplay moments.
  Extracts per-clip 300p video previews, loads a pre-built dashboard template, and polls for user decisions.
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
