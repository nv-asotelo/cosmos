# Serve A Review Frontend

Use this optional step after the headless runner has written
`raw_response.txt`, `reasoning_trace.txt`, `plan.json`, and
`request_summary.json`. It creates a small static browser view from local files
and serves it with Python's standard library.

This frontend is for review only. It does not call the model, execute robot
motion, or make safety decisions.

## Inputs

Set the same image source used for the headless run:

```bash
export PALLETIZER_OUTPUT_DIR="${PALLETIZER_OUTPUT_DIR:-/tmp/cosmos3-palletizer-headless}"
export PALLETIZER_IMAGE_DIR=/path/to/my/box-images
# or
export PALLETIZER_MANIFEST=/path/to/manifest.json
```

If the images are in a mounted volume, use the path visible from the process
running this script. For a Docker runtime, that means the container-visible path,
not the host path.

## Agent Instructions

An agent should:

1. Run `run_custom_images_with_reasoner.md` first.
2. Confirm `PALLETIZER_OUTPUT_DIR/plan.json` exists.
3. Start this review server in the background.
4. Report the URL and the PID file path.
5. Stop the server with `kill $(cat "$PALLETIZER_OUTPUT_DIR/review_server.pid")`
   when the user is done.

For local browser use, bind to `127.0.0.1`. On a remote GPU host, bind to
`0.0.0.0` only on a trusted network.

```bash
export PALLETIZER_OUTPUT_DIR="${PALLETIZER_OUTPUT_DIR:-/tmp/cosmos3-palletizer-headless}"
export PALLETIZER_REVIEW_HOST="${PALLETIZER_REVIEW_HOST:-127.0.0.1}"
export PALLETIZER_REVIEW_PORT="${PALLETIZER_REVIEW_PORT:-3000}"
mkdir -p "${PALLETIZER_OUTPUT_DIR}"

cat >"${PALLETIZER_OUTPUT_DIR}/serve_review.py" <<'PY'
import html
import json
import os
import shutil
from functools import partial
from http.server import SimpleHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path

IMAGE_EXTS = {".jpg", ".jpeg", ".png", ".webp"}


def read_json(path: Path, default):
    if not path.exists():
        return default
    return json.loads(path.read_text(encoding="utf-8"))


def read_text(path: Path) -> str:
    return path.read_text(encoding="utf-8") if path.exists() else ""


def collect_images():
    manifest_path = os.environ.get("PALLETIZER_MANIFEST")
    image_dir = os.environ.get("PALLETIZER_IMAGE_DIR")

    if manifest_path:
        manifest = Path(manifest_path).expanduser().resolve()
        data = read_json(manifest, {})
        base = manifest.parent
        images = []
        for idx, item in enumerate(data.get("boxes", [])):
            image = item.get("image")
            if not image:
                continue
            path = (base / image).resolve()
            images.append(
                {
                    "box_id": item.get("box_id") or path.stem or f"box_{idx:04d}",
                    "path": path,
                    "metadata": item,
                }
            )
        return images

    if image_dir:
        root = Path(image_dir).expanduser().resolve()
        return [
            {"box_id": path.stem, "path": path, "metadata": {"image": path.name}}
            for path in sorted(root.iterdir())
            if path.suffix.lower() in IMAGE_EXTS
        ]

    return []


def esc(value) -> str:
    return html.escape("" if value is None else str(value))


out_dir = Path(os.environ.get("PALLETIZER_OUTPUT_DIR", "/tmp/cosmos3-palletizer-headless")).expanduser().resolve()
review_dir = out_dir / "review_frontend"
image_out = review_dir / "images"
image_out.mkdir(parents=True, exist_ok=True)

plan = read_json(out_dir / "plan.json", {})
summary = read_json(out_dir / "request_summary.json", {})
trace = read_text(out_dir / "reasoning_trace.txt")
raw = read_text(out_dir / "raw_response.txt")
images = collect_images()

image_map = {}
for item in images:
    src = item["path"]
    if not src.exists():
        continue
    safe_name = f"{item['box_id']}{src.suffix.lower()}"
    dst = image_out / safe_name
    shutil.copy2(src, dst)
    image_map[item["box_id"]] = f"images/{safe_name}"

box_rows = []
for box in plan.get("boxes", []):
    box_id = box.get("box_id", "")
    image_html = ""
    if box_id in image_map:
        image_html = f'<img src="{esc(image_map[box_id])}" alt="{esc(box_id)}">'
    box_rows.append(
        f"""
        <article class="card">
          {image_html}
          <h3>{esc(box_id)}</h3>
          <p><b>Evidence:</b> {esc(box.get("visible_evidence"))}</p>
          <p><b>Contents:</b> {esc(box.get("contents_guess"))}</p>
          <p><b>Pickable:</b> {esc(box.get("pickable"))}</p>
          <p><b>Risks:</b> {esc(", ".join(box.get("risks", [])))}</p>
          <p><b>Handling:</b> {esc(box.get("handling"))}</p>
        </article>
        """
    )

if not box_rows:
    for item in images:
        image_html = ""
        if item["box_id"] in image_map:
            image_html = f'<img src="{esc(image_map[item["box_id"]])}" alt="{esc(item["box_id"])}">'
        box_rows.append(
            f"""
            <article class="card">
              {image_html}
              <h3>{esc(item["box_id"])}</h3>
              <pre>{esc(json.dumps(item["metadata"], indent=2))}</pre>
            </article>
            """
        )

actions = plan.get("actions", [])
action_items = "\n".join(f"<li><pre>{esc(json.dumps(action, indent=2))}</pre></li>" for action in actions)

index = f"""<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Cosmos3 Palletizer Review</title>
  <style>
    body {{ margin: 0; background: #0b0c0f; color: #e8e8e8; font-family: Arial, sans-serif; }}
    header {{ padding: 20px 28px; border-bottom: 1px solid #24262b; }}
    main {{ padding: 24px 28px; display: grid; gap: 24px; }}
    h1, h2, h3 {{ margin: 0 0 12px; }}
    .meta, .panel, .card {{ background: #14161a; border: 1px solid #2a2d33; border-radius: 8px; padding: 16px; }}
    .grid {{ display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 16px; }}
    img {{ width: 100%; max-height: 220px; object-fit: contain; background: #f3f4f6; border-radius: 6px; }}
    pre {{ white-space: pre-wrap; overflow-wrap: anywhere; color: #d6d6d6; }}
    b {{ color: #ffffff; }}
  </style>
</head>
<body>
  <header>
    <h1>Cosmos3 Palletizer Review</h1>
    <div class="meta">
      <b>Model:</b> {esc(summary.get("model"))}
      <br><b>Elapsed:</b> {esc(summary.get("elapsed_sec"))} sec
      <br><b>Output:</b> {esc(out_dir)}
    </div>
  </header>
  <main>
    <section>
      <h2>Box Evidence</h2>
      <div class="grid">
        {''.join(box_rows)}
      </div>
    </section>
    <section class="panel">
      <h2>Actions</h2>
      <ol>{action_items}</ol>
    </section>
    <section class="panel">
      <h2>Reasoning Trace</h2>
      <pre>{esc(trace)}</pre>
    </section>
    <section class="panel">
      <h2>Raw Response</h2>
      <pre>{esc(raw)}</pre>
    </section>
  </main>
</body>
</html>
"""

(review_dir / "index.html").write_text(index, encoding="utf-8")
host = os.environ.get("PALLETIZER_REVIEW_HOST", "127.0.0.1")
port = int(os.environ.get("PALLETIZER_REVIEW_PORT", "3000"))
handler = partial(SimpleHTTPRequestHandler, directory=str(review_dir))
print(f"Serving {review_dir} at http://{host}:{port}/", flush=True)
ThreadingHTTPServer((host, port), handler).serve_forever()
PY

python3 "${PALLETIZER_OUTPUT_DIR}/serve_review.py" \
  >"${PALLETIZER_OUTPUT_DIR}/review_server.log" 2>&1 &
echo $! >"${PALLETIZER_OUTPUT_DIR}/review_server.pid"
echo "PID: $(cat "${PALLETIZER_OUTPUT_DIR}/review_server.pid")"
echo "URL: http://${PALLETIZER_REVIEW_HOST}:${PALLETIZER_REVIEW_PORT}/"
```

Stop the server:

```bash
kill "$(cat "${PALLETIZER_OUTPUT_DIR}/review_server.pid")"
```
