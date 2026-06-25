# Run Custom Images With Cosmos3 Reasoner

This appendix is for agents and advanced users who need to make the human
recipe in [README.md](README.md) run headlessly. It sends local box images to a
Cosmos3 Reasoner `/v1/chat/completions` endpoint and writes the saved artifacts
used by the optional review UI.

The runner validates reasoning and action formatting only. It does not execute
robot motion or certify that a placement is safe for a real cell.

## Environment

Set these variables before running the Python block.

| Variable | Default | Purpose |
| --- | --- | --- |
| `COSMOS3_REASONER_BASE_URL` | `http://127.0.0.1:8200/v1` | Base URL for the Cosmos3 Reasoner endpoint |
| `COSMOS3_REASONER_MODEL` | `nvidia/Cosmos3-Nano` | Served model ID returned by `/v1/models` |
| `COSMOS3_REASONER_API_KEY` | empty | Optional bearer token for secured endpoints |
| `PALLETIZER_IMAGE_DIR` | empty | Folder of `jpg`, `jpeg`, `png`, or `webp` files |
| `PALLETIZER_MANIFEST` | empty | JSON manifest with boxes, images, pallets, and optional metadata |
| `PALLETIZER_OUTPUT_DIR` | `/tmp/cosmos3-palletizer-headless` | Directory where artifacts are written |
| `PALLETIZER_REASONING_MODE` | `summary` | `summary` for compact evidence, `off` for action-first output |
| `MAX_COMPLETION_TOKENS` | `1536` | Response budget for the JSON object |
| `PALLETIZER_REQUEST_TIMEOUT` | `600` | Endpoint timeout in seconds |

Confirm the endpoint first:

```bash
export COSMOS3_REASONER_BASE_URL="${COSMOS3_REASONER_BASE_URL:-http://127.0.0.1:8200/v1}"
export COSMOS3_REASONER_MODEL="${COSMOS3_REASONER_MODEL:-nvidia/Cosmos3-Nano}"
curl -fsS "${COSMOS3_REASONER_BASE_URL}/models"
```

## Input Patterns

### Local Folder

Use a folder when each image filename can become the `box_id`.

```bash
export PALLETIZER_IMAGE_DIR=/data/palletizer/images
export PALLETIZER_OUTPUT_DIR=/tmp/cosmos3-palletizer-headless
```

### Manifest

Use a manifest when you need repeatable metadata. Relative image paths are
resolved from the manifest directory.

```json
{
  "pallets": [
    {
      "id": 1,
      "occupied_cells": 0,
      "valid_positions": [[0, 0, 0], [1, 0, 0]]
    },
    {
      "id": 2,
      "occupied_cells": 0,
      "valid_positions": [[0, 0, 0], [1, 0, 0]]
    }
  ],
  "boxes": [
    {
      "box_id": "box_0000",
      "image": "box_0000.png",
      "weight_kg": 2.1,
      "size_cm": [50, 50, 25],
      "notes": "wireless earbuds, fragile contents",
      "valid_positions": {
        "1": [[0, 0, 0], [1, 0, 0]],
        "2": [[0, 0, 0]]
      }
    }
  ]
}
```

```bash
export PALLETIZER_MANIFEST=/data/palletizer/manifest.json
export PALLETIZER_OUTPUT_DIR=/tmp/cosmos3-palletizer-headless
```

If both `PALLETIZER_MANIFEST` and `PALLETIZER_IMAGE_DIR` are set, the manifest
wins.

### Mounted Files

When an agent runs inside a container or remote workspace, mount input data and
outputs explicitly, then use the path visible from that process:

```bash
export PALLETIZER_MANIFEST=/mnt/palletizer/manifest.json
export PALLETIZER_OUTPUT_DIR=/mnt/palletizer/outputs
export COSMOS3_REASONER_BASE_URL=http://HOST_OR_IP:8200/v1
```

For read-only inputs, keep the image mount read-only and write artifacts to a
separate output mount.

### Public Datasets

Materialize public data locally first, review the dataset license, then point
the recipe at the local subset.

```bash
export PALLETIZER_DATA_ROOT=/tmp/palletizer-public-data
mkdir -p "${PALLETIZER_DATA_ROOT}/images"

# Example for a public archive. Substitute the dataset source you are
# allowed to use.
curl -L -o "${PALLETIZER_DATA_ROOT}/dataset.zip" "https://example.com/public-palletizer-images.zip"
python3 - <<'PY'
from pathlib import Path
from zipfile import ZipFile

root = Path("/tmp/palletizer-public-data")
with ZipFile(root / "dataset.zip") as archive:
    archive.extractall(root)
PY

find "${PALLETIZER_DATA_ROOT}" -type f \
  \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o -iname "*.webp" \) \
  | head
```

Create a manifest when public filenames do not contain useful `box_id` values
or when you need weights, dimensions, labels, pallet state, or valid positions.

## Prompt Shape

The runner uses a bounded prompt so the response stays parseable:

- Return one JSON object and no Markdown.
- Keep `reasoning_trace` empty in action-first mode.
- Keep `reasoning_trace` under 180 words in compact reasoning mode.
- Keep each action `reason` under 30 words.
- Use only visible box IDs and supplied valid positions.
- Choose from `PICK_AND_PLACE`, `CALL_A_HUMAN`, and `WAIT`.

If JSON truncates or actions lag behind input images, reduce the batch size, set
`PALLETIZER_REASONING_MODE=off`, or increase `MAX_COMPLETION_TOKENS`.

## Run

```bash
export PALLETIZER_OUTPUT_DIR="${PALLETIZER_OUTPUT_DIR:-/tmp/cosmos3-palletizer-headless}"
export PALLETIZER_REASONING_MODE="${PALLETIZER_REASONING_MODE:-summary}"
export MAX_COMPLETION_TOKENS="${MAX_COMPLETION_TOKENS:-1536}"
export PALLETIZER_REQUEST_TIMEOUT="${PALLETIZER_REQUEST_TIMEOUT:-600}"

python3 - <<'PY'
import base64
import json
import mimetypes
import os
import time
import urllib.error
import urllib.request
from pathlib import Path

IMAGE_EXTS = {".jpg", ".jpeg", ".png", ".webp"}
ACTION_TYPES = {"PICK_AND_PLACE", "CALL_A_HUMAN", "WAIT"}


def request_json(url: str, payload: dict | None = None, timeout: int = 600) -> dict:
    data = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {"Content-Type": "application/json"}
    api_key = os.environ.get("COSMOS3_REASONER_API_KEY")
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"
    req = urllib.request.Request(
        url,
        data=data,
        headers=headers,
        method="GET" if payload is None else "POST",
    )
    try:
        with urllib.request.urlopen(req, timeout=timeout) as response:
            return json.loads(response.read())
    except urllib.error.HTTPError as exc:
        body = exc.read().decode("utf-8", errors="ignore")
        raise SystemExit(f"HTTP {exc.code} from {url}: {body[:2000]}") from exc


def image_data_url(path: Path) -> str:
    mime = mimetypes.guess_type(path.name)[0] or "image/png"
    encoded = base64.b64encode(path.read_bytes()).decode("ascii")
    return f"data:{mime};base64,{encoded}"


def load_inputs() -> tuple[list[dict], list[dict]]:
    manifest_path = os.environ.get("PALLETIZER_MANIFEST")
    image_dir = os.environ.get("PALLETIZER_IMAGE_DIR")

    if manifest_path:
        manifest = Path(manifest_path).expanduser().resolve()
        data = json.loads(manifest.read_text(encoding="utf-8"))
        base = manifest.parent
        boxes = []
        for idx, item in enumerate(data.get("boxes", [])):
            image = item.get("image")
            if not image:
                raise SystemExit(f"Manifest box {idx} is missing image")
            image_path = (base / image).resolve()
            if not image_path.exists():
                raise SystemExit(f"Missing image: {image_path}")
            box = dict(item)
            box.setdefault("box_id", image_path.stem or f"box_{idx:04d}")
            box["image_path"] = str(image_path)
            boxes.append(box)
        if not boxes:
            raise SystemExit(f"No boxes found in manifest: {manifest}")
        return boxes, data.get("pallets", [])

    if not image_dir:
        raise SystemExit("Set PALLETIZER_MANIFEST or PALLETIZER_IMAGE_DIR")

    root = Path(image_dir).expanduser().resolve()
    images = sorted(path for path in root.iterdir() if path.suffix.lower() in IMAGE_EXTS)
    if not images:
        raise SystemExit(f"No images found in {root}")
    boxes = [
        {
            "box_id": path.stem,
            "image": path.name,
            "image_path": str(path),
            "weight_kg": None,
            "size_cm": None,
            "notes": "",
        }
        for path in images
    ]
    return boxes, []


def public_box(box: dict) -> dict:
    clean = {key: value for key, value in box.items() if key != "image_path"}
    clean.setdefault("valid_positions", {"1": [[0, 0, 0]], "2": [[0, 0, 0]]})
    return clean


def build_prompt(boxes: list[dict], pallets: list[dict], reasoning_mode: str) -> str:
    public_boxes = [public_box(box) for box in boxes]
    if reasoning_mode == "off":
        trace_rule = (
            "Set reasoning_trace to an empty string. Keep evidence inside each "
            "box object and keep each action reason under 30 words."
        )
    else:
        trace_rule = (
            "Fill reasoning_trace with at most 180 words of visible evidence "
            "and decision rationale. Do not include hidden chain-of-thought."
        )

    return f"""
You are a Cosmos3 Reasoner assistant for a mixed-SKU palletizing review.
Inspect the attached box images and the metadata below.

Return exactly one JSON object and no Markdown.

Required JSON shape:
{{
  "reasoning_trace": "string",
  "boxes": [
    {{
      "box_id": "string",
      "visible_evidence": "string",
      "contents_guess": "string",
      "pickable": true,
      "risks": ["fragile"],
      "handling": "gentle|standard|firm"
    }}
  ],
  "actions": [
    {{
      "action": "PICK_AND_PLACE|CALL_A_HUMAN|WAIT",
      "box_id": "string for PICK_AND_PLACE",
      "box_ids": ["strings for CALL_A_HUMAN"],
      "target_pallet": 1,
      "position": [0, 0, 0],
      "speed_pct": 40,
      "grip_strength": "gentle|standard|firm",
      "reason": "short reason"
    }}
  ]
}}

Policy:
- Escalate open, torn, crushed, contaminated, or unsealed boxes with CALL_A_HUMAN.
- Place heavy or rigid goods on the lowest valid stable layer.
- Place fragile, glass, electronic, or crushable goods above heavier goods when possible.
- Use gentle grip and slow speed for fragile goods.
- Use firm grip for dense, canned, or rigid goods.
- Use only visible box IDs.
- Use only valid positions supplied in box metadata when present.
- If no valid placement exists, return WAIT or CALL_A_HUMAN.
- {trace_rule}

Pallet metadata:
{json.dumps(pallets, indent=2)}

Box metadata:
{json.dumps(public_boxes, indent=2)}
""".strip()


def message_text(message_content) -> str:
    if isinstance(message_content, str):
        return message_content
    if isinstance(message_content, list):
        parts = []
        for item in message_content:
            if isinstance(item, dict) and item.get("type") == "text":
                parts.append(str(item.get("text", "")))
        return "\n".join(parts)
    return str(message_content)


def first_json_object(text: str) -> dict:
    start = None
    depth = 0
    in_string = False
    escape = False
    for idx, ch in enumerate(text):
        if start is None:
            if ch == "{":
                start = idx
                depth = 1
            continue
        if in_string:
            if escape:
                escape = False
            elif ch == "\\":
                escape = True
            elif ch == '"':
                in_string = False
            continue
        if ch == '"':
            in_string = True
        elif ch == "{":
            depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0:
                return json.loads(text[start : idx + 1])
    raise ValueError("No complete JSON object found in model response")


def validate_plan(plan: dict, visible_box_ids: set[str]) -> None:
    if not isinstance(plan.get("boxes"), list):
        raise ValueError("plan.boxes must be a list")
    actions = plan.get("actions")
    if not isinstance(actions, list) or not actions:
        raise ValueError("plan.actions must be a non-empty list")
    for action in actions:
        action_type = action.get("action")
        if action_type not in ACTION_TYPES:
            raise ValueError(f"Unsupported action type: {action_type}")
        if action_type == "PICK_AND_PLACE":
            missing = [
                key
                for key in ("box_id", "target_pallet", "position", "speed_pct", "grip_strength", "reason")
                if key not in action
            ]
            if missing:
                raise ValueError(f"PICK_AND_PLACE missing fields: {missing}")
            if action["box_id"] not in visible_box_ids:
                raise ValueError(f"Action references non-visible box_id: {action['box_id']}")
        elif action_type == "CALL_A_HUMAN":
            box_ids = action.get("box_ids")
            if not isinstance(box_ids, list) or not box_ids:
                raise ValueError("CALL_A_HUMAN requires non-empty box_ids")
            unknown = [box_id for box_id in box_ids if box_id not in visible_box_ids]
            if unknown:
                raise ValueError(f"CALL_A_HUMAN references non-visible boxes: {unknown}")
        elif action_type == "WAIT" and not action.get("reason"):
            raise ValueError("WAIT requires reason")


base_url = os.environ.get("COSMOS3_REASONER_BASE_URL", "http://127.0.0.1:8200/v1").rstrip("/")
model = os.environ.get("COSMOS3_REASONER_MODEL", "nvidia/Cosmos3-Nano")
out_dir = Path(os.environ.get("PALLETIZER_OUTPUT_DIR", "/tmp/cosmos3-palletizer-headless")).expanduser().resolve()
reasoning_mode = os.environ.get("PALLETIZER_REASONING_MODE", "summary").strip().lower()
max_tokens = int(os.environ.get("MAX_COMPLETION_TOKENS", "1536"))
timeout = int(os.environ.get("PALLETIZER_REQUEST_TIMEOUT", "600"))

if reasoning_mode not in {"summary", "off"}:
    raise SystemExit("PALLETIZER_REASONING_MODE must be 'summary' or 'off'")

boxes, pallets = load_inputs()
prompt = build_prompt(boxes, pallets, reasoning_mode)
content = [{"type": "text", "text": prompt}]
for box in boxes:
    path = Path(box["image_path"])
    content.append({"type": "image_url", "image_url": {"url": image_data_url(path)}})

payload = {
    "model": model,
    "messages": [
        {
            "role": "system",
            "content": "You produce concise, parseable palletizing review JSON for Cosmos 3 recipes.",
        },
        {"role": "user", "content": content},
    ],
    "temperature": 0,
    "max_tokens": max_tokens,
}

out_dir.mkdir(parents=True, exist_ok=True)
start = time.time()
response = request_json(f"{base_url}/chat/completions", payload, timeout=timeout)
elapsed = round(time.time() - start, 3)
message = response["choices"][0]["message"]
raw_text = message_text(message.get("content", ""))
plan = first_json_object(raw_text)
plan.setdefault("reasoning_trace", "")
validate_plan(plan, {box["box_id"] for box in boxes})

(out_dir / "raw_response.txt").write_text(raw_text, encoding="utf-8")
(out_dir / "reasoning_trace.txt").write_text(str(plan.get("reasoning_trace", "")), encoding="utf-8")
(out_dir / "plan.json").write_text(json.dumps(plan, indent=2), encoding="utf-8")
(out_dir / "request_payload.json").write_text(json.dumps(payload, indent=2), encoding="utf-8")
(out_dir / "inputs.json").write_text(
    json.dumps({"boxes": [public_box(box) for box in boxes], "pallets": pallets}, indent=2),
    encoding="utf-8",
)
(out_dir / "request_summary.json").write_text(
    json.dumps(
        {
            "base_url": base_url,
            "model": model,
            "reasoning_mode": reasoning_mode,
            "max_completion_tokens": max_tokens,
            "elapsed_sec": elapsed,
            "box_ids": [box["box_id"] for box in boxes],
            "output_dir": str(out_dir),
        },
        indent=2,
    ),
    encoding="utf-8",
)

print(json.dumps({"output_dir": str(out_dir), "elapsed_sec": elapsed, "actions": plan["actions"]}, indent=2))
PY
```

## Notebook Usage

For a notebook, set the same environment variables in the first cell:

```python
import os

os.environ["COSMOS3_REASONER_BASE_URL"] = "http://127.0.0.1:8200/v1"
os.environ["COSMOS3_REASONER_MODEL"] = "nvidia/Cosmos3-Nano"
os.environ["PALLETIZER_MANIFEST"] = "/data/palletizer/manifest.json"
os.environ["PALLETIZER_OUTPUT_DIR"] = "/tmp/cosmos3-palletizer-headless"
os.environ["PALLETIZER_REASONING_MODE"] = "summary"
os.environ["MAX_COMPLETION_TOKENS"] = "1536"
```

Then paste the Python body from the `python3 - <<'PY'` block into a cell. After
it runs, load `plan.json` and render the selected images:

```python
from pathlib import Path
import json

out_dir = Path(os.environ["PALLETIZER_OUTPUT_DIR"])
plan = json.loads((out_dir / "plan.json").read_text())
plan["actions"]
```

## Agent Handoff Notes

When handing this recipe to another agent, include:

- endpoint base URL and served model ID,
- whether the endpoint is local or remote,
- input source path or manifest path,
- output directory,
- reasoning mode,
- any dataset license constraints,
- whether the optional review UI should be started after inference.
