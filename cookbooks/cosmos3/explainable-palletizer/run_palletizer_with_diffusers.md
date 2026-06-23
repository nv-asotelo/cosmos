# Run Explainable Palletizer with Cosmos3 Diffusers

This markdown entry point validates the Cosmos3-Nano or Cosmos3-Super generation
path used by the explainable palletizer cookbook. It expects a small
Diffusers-compatible proxy with:

- `GET /health`
- `GET /v1/models`
- `POST /v1/infer`

The same request shape works for Nano and Super. The model is selected by the
server; verify it with `/v1/models` before running generation.

## 1. Choose an endpoint

Use the deployed Nano endpoint:

```bash
export COSMOS3_DIFFUSERS_BASE_URL="${COSMOS3_DIFFUSERS_BASE_URL:-http://127.0.0.1:8000}"
```

For a local Super deployment, point the variable at the Super service instead:

```bash
export COSMOS3_DIFFUSERS_BASE_URL="http://127.0.0.1:8000"
```

## 2. Smoke test health and model identity

```bash
curl -fsS "${COSMOS3_DIFFUSERS_BASE_URL}/health"
curl -fsS "${COSMOS3_DIFFUSERS_BASE_URL}/v1/models"
```

Expected Nano model response:

```json
{"object":"list","data":[{"id":"nvidia/Cosmos3-Nano","object":"model"}]}
```

For Super, the returned model ID should identify `nvidia/Cosmos3-Super`.

## 3. Generate a palletizing-scene artifact

```bash smoke-test
python3 - <<'PY'
import base64
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path

base_url = os.environ.get("COSMOS3_DIFFUSERS_BASE_URL", "http://127.0.0.1:8000").rstrip("/")
out_dir = Path(os.environ.get("COSMOS3_OUTPUT_DIR", "/tmp/cosmos3-palletizer-smoke"))
out_dir.mkdir(parents=True, exist_ok=True)

payload = {
    "prompt": (
        "A clean warehouse palletizing cell with a robot arm sorting mixed "
        "cardboard boxes, visible handling labels, a conveyor, and an "
        "operator review panel, technical demo style."
    ),
    "negative_prompt": "blurry, unsafe robot motion, broken gripper, low quality",
    "resolution": os.environ.get("COSMOS3_RESOLUTION", "256"),
    "num_output_frames": int(os.environ.get("COSMOS3_NUM_FRAMES", "1")),
    "fps": float(os.environ.get("COSMOS3_FPS", "1")),
    "steps": int(os.environ.get("COSMOS3_STEPS", "1")),
    "guidance_scale": float(os.environ.get("COSMOS3_GUIDANCE", "1.1")),
    "seed": int(os.environ.get("COSMOS3_SEED", "20260616")),
}

start = time.time()
req = urllib.request.Request(
    f"{base_url}/v1/infer",
    data=json.dumps(payload).encode("utf-8"),
    headers={"Content-Type": "application/json"},
    method="POST",
)

try:
    with urllib.request.urlopen(req, timeout=1800) as response:
        data = json.loads(response.read())
except urllib.error.HTTPError as exc:
    body = exc.read().decode("utf-8", errors="replace")
    raise SystemExit(f"generation failed with HTTP {exc.code}: {body[:2000]}")

if data.get("error"):
    raise SystemExit(f"generation failed: {data['error']}")

media = base64.b64decode(data["b64_video"], validate=True)
suffix = ".jpg" if media.startswith(b"\xff\xd8\xff") else ".mp4"
artifact = out_dir / f"palletizer-cosmos3-{payload['seed']}{suffix}"
artifact.write_bytes(media)

print(json.dumps({
    "backend": data.get("backend"),
    "seed": payload["seed"],
    "bytes": len(media),
    "artifact": str(artifact),
    "elapsed_sec": round(time.time() - start, 2),
}, indent=2))

if len(media) < 1024:
    raise SystemExit("decoded media artifact is unexpectedly small")
PY
```

If the Diffusers service was just started, the first generation request may
spend several minutes loading model weights before it returns. Keep the request
timeout high enough for that warm load; subsequent smoke requests should be much
faster on the same process.

## 4. Run the Doosan full-stack smoke

Choose the full-stack mode first:

```bash
# off = default Cosmos3-Nano Reasoner action smoke; verbose reasoning not required
# on  = Cosmos3-Nano Reasoner with visible reasoning required as validation evidence
export PALLETIZER_REASONING_MODE="${PALLETIZER_REASONING_MODE:-off}"
```

Both cookbook modes use Cosmos3 by default. The source repo's
`docker-compose.test.yml` includes `facebook/opt-125m`, but that service is a
plumbing stub. It is not multimodal and rejects the palletizer image request, so
do not use the unmodified test compose stack as the recipe default.

This recipe still uses `docker-compose.test.yml` for convenient
sim/app/frontend wiring, but `app-server` must be overridden to call an external
Cosmos3-Nano Reasoner `/v1` endpoint.

### Default: Cosmos3 action smoke

Use this mode for the cookbook's normal full-stack smoke. It validates that
Cosmos3-Nano sees the box images, returns one parsed action per iteration, and
the simulator accepts the robot command. The reasoning panel may contain a short
action rationale instead of a long visual trace.

```bash
export PALLETIZER_REASONING_MODE=off
git clone https://github.com/doosan-robotics/explainable-palletizer.git
cd explainable-palletizer
cp docker/.env.example docker/.env
test -f uv.lock || uv lock
```

Default acceptance:

- `/api/health` returns `{"status":"ok"}`.
- `/api/status` reports healthy services.
- `app-server` logs show a successful request to the external Cosmos3-Nano
  Reasoner endpoint.
- At least one action is parsed and executed or deliberately escalated with
  `CALL_A_HUMAN`.
- The frontend loads and shows camera frames, box images, actions, and execution
  status.

### Optional: visible reasoning trace

Set `PALLETIZER_REASONING_MODE=on` when the validation target is the visible
reasoning trace itself. This uses the same Cosmos3-Nano Reasoner endpoint and
the same robot checks, but the pass condition also requires non-empty reasoning
text in the UI or saved `response.txt` artifacts.

This mode is stricter because the reference app asks the model for free-form
reasoning and action JSON in the same completion. A larger token budget reduces
truncation, but prompt and max-token settings alone cannot guarantee complete
parseable JSON for every scene. Keep the default action smoke for CI or PR
gating until the app uses structured output or a separate action-only follow-up
call.

Reserve host port `8200` for Cosmos3-Nano Reasoner. The upstream test
`inference-server` container can still satisfy the compose dependency, but it
must bind a different host port so it does not collide with Cosmos3:

```bash
git clone https://github.com/doosan-robotics/explainable-palletizer.git
cd explainable-palletizer
cp docker/.env.example docker/.env
test -f uv.lock || uv lock

python3 - <<'PY'
from pathlib import Path
p = Path("docker/.env")
text = p.read_text()
text = text.replace("INFERENCE_PORT=8200", "INFERENCE_PORT=8320")
p.write_text(text)
PY
```

If another service already occupies the other defaults, also change `SIM_PORT`,
`APP_PORT`, or `FRONTEND_PORT` in `docker/.env` before starting compose.

Start a Cosmos3-Nano Reasoner OpenAI-compatible server before launching the app
loop. The two supported Nano paths are:

- vLLM: serve `nvidia/Cosmos3-Nano` with
  the Cosmos3-Nano Reasoner checkpoint. Add
  `--hf-overrides '{"architectures": ["Cosmos3ReasonerForConditionalGeneration"]}'`
  only if the checkpoint metadata does not already declare the Reasoner
  architecture.
- NIM: serve `nvidia/cosmos3-nano-reasoner` with `NIM_MODEL_SIZE=nano`.

Follow the shared [vLLM Reasoner](../README.md#vllm) or
[NIM Reasoner](../README.md#nim) setup, then point the Doosan app server at the
Reasoner `/v1` endpoint. A successful Cosmos3 full-stack smoke is not just
healthy containers: the logs must show a successful Cosmos3 completion request
and the app must parse an action. If `PALLETIZER_REASONING_MODE=on`, also
require non-empty visible reasoning text.

The validated agent path is to keep the Reasoner outside the Doosan compose
stack and point `app-server` at it. This avoids accidentally treating the
`facebook/opt-125m` health stub as a Cosmos3 pass.

```bash
# Run on the GPU host from the environment where vLLM and the Reasoner weights
# are installed. COSMOS3_REASONER_MODEL_PATH may be a local checkpoint path or
# a Hugging Face model path after license acceptance.
export COSMOS3_REASONER_MODEL_PATH="${COSMOS3_REASONER_MODEL_PATH:-/path/to/Cosmos3-Nano-Reasoner}"
export COSMOS3_REASONER_PORT="${COSMOS3_REASONER_PORT:-8200}"

vllm serve "${COSMOS3_REASONER_MODEL_PATH}" \
  --served-model-name nvidia/Cosmos3-Nano \
  --host 0.0.0.0 \
  --port "${COSMOS3_REASONER_PORT}" \
  --max-model-len 8192 \
  --media-io-kwargs '{"video": {"num_frames": -1}}' \
  --reasoning-parser qwen3 \
  --gpu-memory-utilization 0.55
```

Confirm the endpoint before starting the app loop:

```bash
curl -fsS "http://127.0.0.1:${COSMOS3_REASONER_PORT}/v1/models"
```

Create an app-server override that targets the host Reasoner. Keep completion
limits high enough for visual reasoning plus JSON action output. `512` tokens
can truncate three-box outputs inside `<answer>`; `1024` is the default smoke
setting for Cosmos3-Nano Reasoner:

```bash
cat >/tmp/cosmos3-palletizer-reasoner.override.yml <<'YAML'
services:
  app-server:
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      SIM_SERVER_URL: http://sim-server:8100
      INFERENCE_SERVER_URL: "http://host.docker.internal:${COSMOS3_REASONER_PORT:-8200}/v1"
      INFERENCE_MODEL: nvidia/Cosmos3-Nano
      MAX_COMPLETION_TOKENS: "1024"
      REQUEST_TIMEOUT: "240"
      STEP_LOG_DIR: /tmp/palletizer-step-logs
      APP_HOST: 0.0.0.0
      APP_PORT: "8000"
YAML
```

Start or recreate the UI-facing services with the override:

```bash
cd docker
docker compose -f docker-compose.yml \
  -f docker-compose.test.yml \
  -f /tmp/cosmos3-palletizer-reasoner.override.yml \
  up -d --force-recreate app-server frontend

curl -fsS http://127.0.0.1:8000/api/health
curl -fsS http://127.0.0.1:8000/api/status
curl -fsS http://127.0.0.1:3000/ >/dev/null
```

Start the loop from the UI or with the API:

```bash
curl -fsS -X POST http://127.0.0.1:8000/api/control/start
```

The stack is using Cosmos3 only after app-server logs show a request to the
Reasoner endpoint and a parsed action:

```bash
docker compose -f docker-compose.yml \
  -f docker-compose.test.yml \
  -f /tmp/cosmos3-palletizer-reasoner.override.yml \
  logs --tail=160 app-server
```

Look for lines like:

```text
POST http://host.docker.internal:8200/v1/chat/completions "HTTP/1.1 200 OK"
LLM raw response (first 500 chars): <think>
Executing: action=PICK_AND_PLACE box_id=box_0001 pallet=1 position=(0, 0, 1)
```

The `/api/status` response can still display a service named
`inference-server` because that is the app's generic health label. The default
smoke passes only when the logs show the external Cosmos3-Nano Reasoner URL and
the UI or step artifacts contain a parsed action. In
`PALLETIZER_REASONING_MODE=on`, additionally require a non-empty reasoning trace
in the UI or `response.txt`.

The app also writes per-step handoff artifacts inside the container:

```bash
docker compose -f docker-compose.yml \
  -f docker-compose.test.yml \
  -f /tmp/cosmos3-palletizer-reasoner.override.yml \
  exec app-server find /tmp/palletizer-step-logs -maxdepth 2 -type f | sort
```

Each successful Cosmos3 step should include `scenario.txt`, `response.txt`,
`action.json`, and the box images that were sent to Cosmos3.

The current public Doosan Dockerfile expects `uv.lock`; the `uv lock` guard
generates it for fresh clones where upstream has not checked in the lockfile.
When the launcher reports all services healthy, open the configured frontend
port and verify that the simulated camera, action panel, and execution status
are visible. For `PALLETIZER_REASONING_MODE=on`, also verify the reasoning panel.

### Plumbing-only upstream fallback

Use the unmodified upstream test stack only when you need to prove that Docker,
Isaac Sim, the FastAPI server, and the frontend can boot. This is not the
cookbook default and is not a Cosmos3 pass:

```bash
export PALLETIZER_PLUMBING_ONLY=1
cd docker
docker compose -f docker-compose.yml -f docker-compose.test.yml up -d
curl -fsS http://127.0.0.1:8000/api/health
curl -fsS http://127.0.0.1:8000/api/status
curl -fsS http://127.0.0.1:3000/ >/dev/null
```

In this fallback, `app-server` points at `facebook/opt-125m`. That model is not
multimodal, so completion requests for box images fail with a 400 error such as
`facebook/opt-125m is not a multimodal model`. Empty reasoning and action panels
are expected and should not be reported as recipe validation.

### Full-stack troubleshooting

If Docker reports `unknown or invalid runtime name: nvidia`, configure the NVIDIA
runtime before rerunning the stack:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

On a single-GPU host, Isaac Sim warmup can race with vLLM memory profiling when
all services start concurrently. Start the services sequentially if `make
docker-test` exits during inference-server profiling:

```bash
cd docker
set -a
. ./.env
set +a

docker compose -f docker-compose.yml -f docker-compose.test.yml up -d sim-server
until curl -fsS "http://127.0.0.1:${SIM_PORT}/sim/health"; do sleep 5; done
sleep 20

docker compose -f docker-compose.yml -f docker-compose.test.yml up -d inference-server
until curl -fsS "http://127.0.0.1:${INFERENCE_PORT}/health"; do sleep 5; done

docker compose -f docker-compose.yml -f docker-compose.test.yml up -d app-server frontend
until curl -fsS "http://127.0.0.1:${APP_PORT}/api/health"; do sleep 5; done
curl -fsS "http://127.0.0.1:${FRONTEND_PORT}/" >/dev/null
```

If you deliberately run `docker-compose.test.yml`, lower
`MAX_COMPLETION_TOKENS` to fit the tiny model only when testing request plumbing.
Do not treat that as a Cosmos3 reasoning pass; the useful palletizer demo should
run against Cosmos3-Nano Reasoner.

If the UI stays on "No reasoning yet" and the action list does not advance:

- Confirm `INFERENCE_SERVER_URL` inside `app-server` points to the external
  Cosmos3 `/v1` endpoint, not the test compose `inference-server`.
- Confirm `curl http://127.0.0.1:8200/v1/models` returns
  `nvidia/Cosmos3-Nano`.
- Inspect app logs for `POST ... /v1/chat/completions`. If there is no POST,
  the app never reached the Reasoner.
- If the UI shows reasoning but no new action, inspect
  `/tmp/palletizer-step-logs/step_*/response.txt`. A response that ends inside
  `<answer>` or midway through JSON was truncated before the parser could build
  an action; increase `MAX_COMPLETION_TOKENS` to `1024` or higher and recreate
  `app-server`.
- Inspect `STEP_LOG_DIR`; empty or missing box images usually means the app did
  not receive usable simulator image IDs.
- If `/sim/buffer/status` returns IDs like `box_0` but
  `/sim/boxes/images_for_ids` only returns images for zero-padded IDs like
  `box_0000`, use a source revision that normalizes those IDs or patch the
  local Doosan checkout for the smoke run.

If the reasoning panel describes several boxes but the action list shows only
one pick, first remember that this is normal control-loop shape: each iteration
asks the model to inspect the front buffer window and emit exactly one bounded
action. Treat it as a simulator/setup failure only when the app logs show the
action was handed to the sim and then failed, for example:

```text
Executing: action=PICK_AND_PLACE box_id=box_0000 ...
POST http://sim-server:8100/sim/robot/pick_place "HTTP/1.1 500 Internal Server Error"
pick_and_place RESPONSE 500: {"detail":"'NoneType' object is not subscriptable"}
```

In that failure mode, `/sim/buffer/status` can remain stuck with a partially
popped buffer such as `{"occupied":2,"slots":[0,1,1],"in_transit":false}`.
The UI may still show the old `box_0000` image because the app already captured
the prompt images, but the physical sim slot has been popped. Reset the
control loop before rerunning so the app stack and simulator buffer agree:

```bash
curl -fsS -X POST http://127.0.0.1:8000/api/control/reset
curl -fsS -X POST http://127.0.0.1:8000/api/control/start
curl -fsS http://127.0.0.1:8100/sim/buffer/status
```

If the same `pick_place` 500 reproduces after a clean reset, record the robot
loop as blocked by the Isaac Sim/cuRobo setup rather than by Cosmos3. Cosmos3
is validated for this path only up to parsed action generation unless the sim
logs also contain a successful `pick_and_place SUCCESS` or equivalent pallet
state update.

If `sim-server` exits with `AttributeError: module 'warp.types' has no attribute
'array'`, the image resolved a newer `warp-lang` than the current cuRobo/Isaac
Sim path expects. Pin `warp-lang==1.12.0` in the sim image, rebuild, and rerun
the smoke; do not count the robot-loop path as passed until all four services
are healthy.
