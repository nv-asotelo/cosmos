# Run Custom Palletizing Images with Cosmos3 Reasoner

Use this entry point when you do not want the Doosan demo frontend, Isaac Sim, or
the robot-control loop. It sends your own box images directly to a Cosmos3-Nano
Reasoner OpenAI-compatible endpoint and writes an operator-visible reasoning
trace plus a structured palletizing recommendation.

This path validates model reasoning and output formatting only. It does not
execute robot motion, compute collision-free trajectories, or certify that a
placement is robot-safe.

## 1. Start or select a Reasoner endpoint

Run a Cosmos3-Nano Reasoner server with vLLM or NIM, then point this client at
the `/v1` base URL:

```bash
export COSMOS3_REASONER_BASE_URL="${COSMOS3_REASONER_BASE_URL:-http://127.0.0.1:8200/v1}"
export COSMOS3_REASONER_MODEL="${COSMOS3_REASONER_MODEL:-nvidia/Cosmos3-Nano}"

curl -fsS "${COSMOS3_REASONER_BASE_URL}/models"
```

The model list should include `nvidia/Cosmos3-Nano` for the default Nano path.
Use the model ID returned by your server if it differs.

## 2. Prepare your images

For quick experiments, put `jpg`, `jpeg`, `png`, or `webp` files in one
directory. The filename stem becomes the `box_id`:

```bash
export PALLETIZER_IMAGE_DIR=/path/to/my/box-images
```

For repeatable tests, provide a manifest with image paths and optional box
metadata. Relative image paths are resolved from the manifest directory.

```json
{
  "pallets": [
    {"id": 1, "max_weight_kg": 500, "occupied_cells": 0},
    {"id": 2, "max_weight_kg": 500, "occupied_cells": 0}
  ],
  "boxes": [
    {
      "box_id": "box_0000",
      "image": "box_0000.png",
      "weight_kg": 2.1,
      "size_cm": [50, 50, 25],
      "notes": "wireless earbuds, fragile contents"
    },
    {
      "box_id": "box_0001",
      "image": "box_0001.png",
      "weight_kg": 9.2,
      "size_cm": [25, 25, 25],
      "notes": "canned drink case"
    }
  ]
}
```

Set:

```bash
export PALLETIZER_MANIFEST=/path/to/manifest.json
```

If both `PALLETIZER_MANIFEST` and `PALLETIZER_IMAGE_DIR` are set, the manifest
wins.

## 3. Run the headless client

The block below uses only the Python standard library, so it can run as a shell
smoke test or be copied into a notebook cell. It writes all outputs to
`PALLETIZER_OUTPUT_DIR`.

```bash
export PALLETIZER_OUTPUT_DIR="${PALLETIZER_OUTPUT_DIR:-/tmp/cosmos3-palletizer-headless}"
export MAX_COMPLETION_TOKENS="${MAX_COMPLETION_TOKENS:-2048}"

python3 - <<'PY'
import base64
import json
import mimetypes
import os
import re
import time
import urllib.error
import urllib.request
from pathlib import Path

IMAGE_EXTS = {".jpg", ".jpeg", ".png", ".webp"}


def request_json(url: str, payload: dict | None = None, timeout: int = 600) -> dict:
    data = None if payload is None else json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(
        url,
        data=data,
        headers={"Content-Type": "application/json"},
        method="GET" if payload is None else "POST",
    )
    try:
        with urllib.request.urlopen(req, timeout=timeout) as response:
            return json.loads(response.read())
    except urllib.error.HTTPError as exc:
        body = exc.read().decode("utf-8", errors="replace")
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
            image_path = (base / item["image"]).resolve()
            box = dict(item)
            box.setdefault("box_id", image_path.stem or f"box_{idx:04d}")
            box["image_path"] = str(image_path)
            boxes.append(box)
        return boxes, data.get("pallets", [])

    if not image_dir:
        raise SystemExit("Set PALLETIZER_MANIFEST or PALLETIZER_IMAGE_DIR")

    root = Path(image_dir).expanduser().resolve()
    images = sorted(p for p in root.iterdir() if p.suffix.lower() in IMAGE_EXTS)
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


def parse_model_response(text: str) -> tuple[dict, str]:
    think_match = re.search(r"<think>(.*?)</think>", text, flags=re.DOTALL | re.IGNORECASE)
    answer_match = re.search(r"<answer>(.*?)</answer>", text, flags=re.DOTALL | re.IGNORECASE)
    answer_text = answer_match.group(1) if answer_match else text
    parsed = first_json_object(answer_text)
    trace = parsed.get("reasoning_trace") or (think_match.group(1).strip() if think_match else "")
    return parsed, trace


base_url = os.environ.get("COSMOS3_REASONER_BASE_URL", "http://127.0.0.1:8200/v1").rstrip("/")
model = os.environ.get("COSMOS3_REASONER_MODEL", "nvidia/Cosmos3-Nano")
out_dir = Path(os.environ.get("PALLETIZER_OUTPUT_DIR", "/tmp/cosmos3-palletizer-headless"))
max_tokens = int(os.environ.get("MAX_COMPLETION_TOKENS", "2048"))
temperature = float(os.environ.get("PALLETIZER_TEMPERATURE", "0.2"))
out_dir.mkdir(parents=True, exist_ok=True)

models = request_json(f"{base_url}/models", timeout=60)
model_ids = [item.get("id") for item in models.get("data", [])]
if model not in model_ids:
    print(json.dumps({"warning": "requested model not listed", "requested": model, "available": model_ids}, indent=2))

boxes, pallets = load_inputs()
if len(boxes) > int(os.environ.get("PALLETIZER_MAX_IMAGES", "6")):
    raise SystemExit("Too many images for one request. Set PALLETIZER_MAX_IMAGES or batch the directory.")

metadata = []
content = []
for box in boxes:
    path = Path(box["image_path"])
    if not path.exists():
        raise SystemExit(f"Missing image for {box['box_id']}: {path}")
    metadata.append({k: v for k, v in box.items() if k not in {"image_path"}})
    content.append({"type": "image_url", "image_url": {"url": image_data_url(path)}})

system_prompt = (
    "You are a careful warehouse palletizing reasoning assistant. "
    "Inspect only the provided images and metadata. Return operator-visible "
    "evidence, risk notes, and a structured recommendation. Do not claim robot "
    "safety certification; this is an advisory planning aid."
)

schema = {
    "reasoning_trace": "Concise operator-visible evidence and decision rationale.",
    "boxes": [
        {
            "box_id": "string",
            "visible_evidence": "text seen in image and visible package condition",
            "contents_guess": "what the label/appearance suggests, or unknown",
            "pickable": True,
            "risks": ["fragile", "heavy", "damaged", "unknown"],
            "handling": "gentle|standard|firm|human_review",
        }
    ],
    "actions": [
        {
            "action": "PICK_AND_PLACE|CALL_A_HUMAN|WAIT",
            "box_id": "string or null",
            "target_pallet": 1,
            "position": [0, 0, 0],
            "speed_pct": 80,
            "grip_strength": "gentle|standard|firm",
            "reason": "brief operator-visible reason",
        }
    ],
}

user_text = (
    "Analyze these palletizing box images as a headless audit. "
    "Use the metadata when present, but prioritize visible evidence from the images. "
    "Recommend safe handling and a palletizing action plan. "
    "For unknown dimensions or weights, explain the uncertainty and choose WAIT or CALL_A_HUMAN if needed.\n\n"
    f"Box metadata:\n{json.dumps(metadata, indent=2)}\n\n"
    f"Pallet state:\n{json.dumps(pallets or [{'id': 1, 'max_weight_kg': 500, 'occupied_cells': 0}], indent=2)}\n\n"
    "Return exactly one JSON object. You may wrap it in <answer>...</answer>. "
    "Use this schema:\n"
    f"{json.dumps(schema, indent=2)}"
)

messages = [
    {"role": "system", "content": [{"type": "text", "text": system_prompt}]},
    {"role": "user", "content": [{"type": "text", "text": user_text}, *content]},
]

payload = {
    "model": model,
    "messages": messages,
    "max_completion_tokens": max_tokens,
    "temperature": temperature,
    "top_p": float(os.environ.get("PALLETIZER_TOP_P", "0.9")),
}

start = time.time()
result = request_json(f"{base_url}/chat/completions", payload, timeout=int(os.environ.get("PALLETIZER_TIMEOUT", "600")))
raw = result["choices"][0]["message"].get("content", "")
parsed, trace = parse_model_response(raw)

(out_dir / "raw_response.txt").write_text(raw, encoding="utf-8")
(out_dir / "reasoning_trace.txt").write_text(trace, encoding="utf-8")
(out_dir / "plan.json").write_text(json.dumps(parsed, indent=2), encoding="utf-8")
(out_dir / "request_summary.json").write_text(
    json.dumps(
        {
            "model": model,
            "model_ids": model_ids,
            "box_ids": [box["box_id"] for box in boxes],
            "elapsed_sec": round(time.time() - start, 2),
            "output_dir": str(out_dir),
        },
        indent=2,
    ),
    encoding="utf-8",
)

print(json.dumps({
    "model": model,
    "boxes": [box["box_id"] for box in boxes],
    "actions": parsed.get("actions", []),
    "output_dir": str(out_dir),
    "elapsed_sec": round(time.time() - start, 2),
}, indent=2))
PY
```

## 4. Use it from a notebook

In a notebook, split the Python block into three cells:

1. Imports and helper functions.
2. Environment/config values plus `load_inputs()`.
3. Request, parse, and save outputs.

The key variables to change interactively are:

```python
base_url = "http://127.0.0.1:8200/v1"
model = "nvidia/Cosmos3-Nano"
manifest_path = "/path/to/manifest.json"
out_dir = "/tmp/cosmos3-palletizer-headless"
```

Display the result with:

```python
from pathlib import Path
import json

out_dir = Path("/tmp/cosmos3-palletizer-headless")
print((out_dir / "reasoning_trace.txt").read_text())
json.loads((out_dir / "plan.json").read_text())
```

## Output files

| File | Contents |
| --- | --- |
| `raw_response.txt` | Exact model response for debugging parser or truncation issues |
| `reasoning_trace.txt` | Operator-visible reasoning/rationale extracted from the JSON response or model text |
| `plan.json` | Structured per-box evidence and recommended actions |
| `request_summary.json` | Model ID, input box IDs, elapsed time, and output directory |

## Notes for real data

- Keep batches small. Three to six images per request is a practical starting
  point for Nano Reasoner.
- Provide dimensions and weights in the manifest when you want placement
  recommendations; otherwise the model should mark the uncertainty.
- Do not use this headless path as a robot command source without a separate
  motion planner, collision checker, weight limits, site rules, and human
  approval.
- For production-grade action extraction, use structured output or a follow-up
  action-only call instead of relying on free-form JSON inside a reasoning
  response.
