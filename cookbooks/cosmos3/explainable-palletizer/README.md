# See How It Thinks: Mixed Palletizing with Explainable Visual Reasoning

This cookbook shows how to validate an explainable mixed-SKU palletizing workflow
with Cosmos3-Nano or Cosmos3-Super. The live Cosmos3 path uses a Diffusers-backed
generation endpoint to create auditable palletizing-scene outputs, while the
Doosan Robotics reference stack provides the full Isaac Sim, cuRobo, FastAPI,
and React control-loop smoke test.

| Model | Workload | Use case |
| --- | --- | --- |
| [Cosmos3-Nano](https://huggingface.co/nvidia/Cosmos3-Nano), [Cosmos3-Super](https://huggingface.co/nvidia/Cosmos3-Super), [Isaac Sim](https://developer.nvidia.com/isaac-sim), [cuRobo](https://curobo.org/), [Diffusers](https://github.com/huggingface/diffusers) | End-to-end | Explainable mixed-SKU palletizing: visual scene generation, handling-policy validation, and simulated robot execution |

**Source project:** [doosan-robotics/explainable-palletizer](https://github.com/doosan-robotics/explainable-palletizer)

## What You Will Build

- Run a Cosmos3-Nano or Cosmos3-Super Diffusers smoke request for a palletizing
  cell and save the generated media artifact.
- Run your own box images through Cosmos3-Nano Reasoner without the demo
  frontend and save the reasoning trace plus structured recommendations.
- Validate the Doosan reference stack with real Isaac Sim, cuRobo, and a
  Cosmos3-Nano Reasoner endpoint that can populate the reasoning and action
  panels.
- Use the same prompt family to document why the system places, delays, or routes
  boxes for human inspection.
- Review the source project's damaged-carton, heavy-box, and mixed-SKU
  walkthroughs in [workflow_e2e.md](workflow_e2e.md).

## Prerequisites

- Follow the shared [Diffusers setup](../README.md#diffusers) if you are
  running Cosmos3 locally.
- For the Doosan reference stack, install Docker with Compose V2 and the NVIDIA
  Container Toolkit on a Linux GPU host.
- Install [`uv`](https://docs.astral.sh/uv/) for the Doosan full-stack smoke if
  the source checkout needs to generate `uv.lock`.
- For full Cosmos3 model runs, authenticate to Hugging Face and accept the
  relevant Cosmos3 model licenses.
- Use a host with enough VRAM for the selected model size.

| Profile | Model | Suggested hardware | Notes |
| --- | --- | --- | --- |
| Nano | `nvidia/Cosmos3-Nano` | Single NVIDIA GB10, RTX 4090-class, or larger GPU | Default for smoke tests and widest accessibility |
| Super | `nvidia/Cosmos3-Super` | Multi-GPU Hopper/Blackwell-class system | Higher quality; launch the serving backend with the Super checkpoint before running the same client request |

## Backends

| Backend | Entry point | GPU requirement |
| --- | --- | --- |
| Diffusers-compatible HTTP endpoint | [`run_palletizer_with_diffusers.md`](run_palletizer_with_diffusers.md) | Nano: single GPU; Super: multi-GPU |
| Headless Cosmos3-Nano Reasoner endpoint | [`run_custom_images_with_reasoner.md`](run_custom_images_with_reasoner.md) | Single GPU serving the Reasoner |
| Doosan reference stack | Cosmos3-Nano Reasoner endpoint plus the source repo app stack | NVIDIA GPU with Isaac Sim support |

These backends validate different things. The Diffusers endpoint validates
Cosmos3-Nano or Cosmos3-Super scene generation. The headless Reasoner path
validates model inspection and structured recommendations on your own images.
The Doosan stack validates the closed-loop UI, parser, pallet-state updates, and
simulated robot execution.
For the cookbook full-stack smoke, the Doosan `app-server` must point at a
Cosmos3-Nano Reasoner OpenAI-compatible `/v1` endpoint. The
`facebook/opt-125m` container in the upstream test compose file is only a
plumbing stub: it is neither multimodal nor a Cosmos3 validation backend.
Use the headless Reasoner endpoint when you want to bring your own box images
and collect model reasoning plus JSON output without running Isaac Sim or the
React frontend.

## Reasoning Modes

Use one of the two Cosmos3 modes for the full-stack recipe. The third row is a
plumbing-only fallback for upstream container checks:

| Mode | Inference backend | What it validates | Expected UI |
| --- | --- | --- | --- |
| `PALLETIZER_REASONING_MODE=off` | External Cosmos3-Nano Reasoner `/v1` endpoint | Default action smoke: parsed one-action-per-iteration decisions, step artifacts, and robot execution without requiring a verbose reasoning trace | Box images, action list, execution status; reasoning may be brief |
| `PALLETIZER_REASONING_MODE=on` | External Cosmos3-Nano Reasoner `/v1` endpoint | Optional visual-reasoning validation with the same action and robot-execution checks | Box images, reasoning text, action list, and execution status |
| `PALLETIZER_PLUMBING_ONLY=1` | Upstream `docker-compose.test.yml` with `facebook/opt-125m` | Container health, Isaac Sim startup, camera streams, box image flow, frontend/app wiring only | Box images can appear; reasoning and actions are not expected |

Both recipe modes use Cosmos3 by default. `PALLETIZER_REASONING_MODE=off` is the
recommended default because the pass condition is a parsed action and accepted
robot command, not a long visible reasoning transcript. `PALLETIZER_REASONING_MODE=on`
adds a stricter human-inspection requirement for the reasoning panel and saved
responses. `MAX_COMPLETION_TOKENS=1024` reduces truncation for Cosmos3-Nano, but
prompt shape alone cannot guarantee that a free-form reasoning response will
always include complete parseable JSON. A production-quality reasoning-on path
should use structured output or a separate action-only call after the visual
audit.

## Quick Start

Use the Diffusers entry point first. It is the fastest way to confirm that the
Cosmos3-Nano or Cosmos3-Super service is reachable and can produce a palletizing
scene artifact:

```bash
cd cookbooks/cosmos3/explainable-palletizer
export COSMOS3_DIFFUSERS_BASE_URL=http://127.0.0.1:8000
curl -fsS "${COSMOS3_DIFFUSERS_BASE_URL}/health"
curl -fsS "${COSMOS3_DIFFUSERS_BASE_URL}/v1/models"
```

Then run the Python smoke-test block in
[`run_palletizer_with_diffusers.md`](run_palletizer_with_diffusers.md) to write a
local media artifact and verify the response can be decoded. Use
[workflow_e2e.md](workflow_e2e.md) to compare the generated artifact against the
operator-review criteria from the reference palletizer scenarios.

To run on your own images without the demo frontend, start Cosmos3-Nano
Reasoner and call the headless client:

```bash
cd cookbooks/cosmos3/explainable-palletizer
export COSMOS3_REASONER_BASE_URL=http://127.0.0.1:8200/v1
export COSMOS3_REASONER_MODEL=nvidia/Cosmos3-Nano
export PALLETIZER_IMAGE_DIR=/path/to/my/box-images
export PALLETIZER_OUTPUT_DIR=/tmp/cosmos3-palletizer-headless
```

Then run
[`run_custom_images_with_reasoner.md`](run_custom_images_with_reasoner.md). It
writes `raw_response.txt`, `reasoning_trace.txt`, `plan.json`, and
`request_summary.json`, and the Python block can be copied directly into a
notebook. Use a `manifest.json` when you want to provide real weights,
dimensions, pallet state, or handling notes.

Then run the full-stack reference smoke on a GPU host. The default full-stack
smoke is Cosmos3-Nano with reasoning not required as a pass condition:

```bash
export PALLETIZER_REASONING_MODE=off
git clone https://github.com/doosan-robotics/explainable-palletizer.git
cd explainable-palletizer
cp docker/.env.example docker/.env
test -f uv.lock || uv lock

# Reserve host port 8200 for Cosmos3-Nano Reasoner. The upstream test
# inference stub can still satisfy the compose dependency, but it must not bind
# to the same host port.
python3 - <<'PY'
from pathlib import Path

p = Path("docker/.env")
text = p.read_text()
text = text.replace("INFERENCE_PORT=8200", "INFERENCE_PORT=8320")
p.write_text(text)
PY
```

Start an OpenAI-compatible Cosmos3-Nano Reasoner server with either vLLM or NIM:

- vLLM serves model ID `nvidia/Cosmos3-Nano` with the
  Cosmos3-Nano Reasoner checkpoint. Add
  `--hf-overrides '{"architectures": ["Cosmos3ReasonerForConditionalGeneration"]}'`
  only if the checkpoint metadata does not already declare the Reasoner
  architecture.
- NIM serves model ID `nvidia/cosmos3-nano-reasoner` with
  `NIM_MODEL_SIZE=nano`.

Follow the shared [vLLM Reasoner setup](../README.md#vllm) or
[NIM Reasoner setup](../README.md#nim), then configure the Doosan app server to
point at that `/v1` endpoint. Keep `MAX_COMPLETION_TOKENS` below the served
model's context budget.

This is the minimum Cosmos3-Nano Reasoner wiring that should be in place before
clicking Play in the UI. Set `PALLETIZER_REASONING_MODE=on` only when the pass
condition includes non-empty visible reasoning in addition to parsed actions:

```bash
# Run on the GPU host, outside the Doosan Docker network.
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

In another shell, confirm the model identity:

```bash
curl -fsS "http://127.0.0.1:${COSMOS3_REASONER_PORT}/v1/models"
```

Then point the Doosan app container at that host service. On Linux Docker,
`host.docker.internal` requires the `host-gateway` mapping:

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

cd docker
docker compose -f docker-compose.yml \
  -f docker-compose.test.yml \
  -f /tmp/cosmos3-palletizer-reasoner.override.yml \
  up -d --force-recreate app-server frontend
```

The app is inferencing with Cosmos3 only after the app logs show requests to the
Reasoner endpoint and parsed actions, for example:

```text
POST http://host.docker.internal:8200/v1/chat/completions "HTTP/1.1 200 OK"
Executing: action=PICK_AND_PLACE box_id=box_0001 pallet=1 position=(0, 0, 1)
```

Do not use the `inference-server` service name as proof of Cosmos3. With this
override, `/api/status` may still label the external Reasoner health check as
`inference-server`, but the validating evidence is the actual
`host.docker.internal:8200/v1/chat/completions` request to Cosmos3-Nano.

The upstream `docker-compose.test.yml` can still be run without the override for
a plumbing-only smoke by setting `PALLETIZER_PLUMBING_ONLY=1`, but do not make
that the cookbook default and do not count it as a Cosmos3 pass. In that mode
the app points at `facebook/opt-125m`, which rejects the multimodal request.

The operator UI can show a reasoning trace for multiple boxes while the action
panel advances by one pick. That is expected: each control-loop iteration asks
Cosmos3 to inspect the front buffer window and emit one bounded action. If the
action list stalls after that, inspect the app and sim logs. A repeated
`/sim/robot/pick_place` 500, especially with a partially popped buffer in
`/sim/buffer/status`, is an Isaac Sim/cuRobo execution blocker rather than a
Cosmos3 model-selection problem. Reset the control loop before rerunning so the
app's captured box stack and the simulator buffer are in sync.

The current upstream Dockerfile expects `uv.lock`; generate it once with
`uv lock` if the source checkout does not already include it. Change `SIM_PORT`,
`INFERENCE_PORT`, `APP_PORT`, and `FRONTEND_PORT` in `docker/.env` if another
local service already uses the defaults. See
[`run_palletizer_with_diffusers.md`](run_palletizer_with_diffusers.md#full-stack-troubleshooting)
for NVIDIA Docker runtime setup, single-GPU sequential startup, and the cuRobo
`warp-lang` compatibility note.

## Architecture

<p align="center">
  <img src="./assets/main_workflow.svg" alt="Explainable palletizer workflow" width="900">
</p>

The Doosan reference stack launches four services:

| Service | Default port | Role |
| --- | --- | --- |
| `sim-server` | 8100 | Runs Isaac Sim headlessly, creates conveyor-box images, and executes cuRobo-planned pick/place trajectories |
| `inference-server` | 8200 | Serves the model endpoint used by the application server |
| `app-server` | 8000 | Builds prompts, parses structured actions, maintains pallet state, and streams events |
| `frontend` | 3000 | Shows camera frames, reasoning, parsed actions, and execution status |

For the Cosmos3 Diffusers path, the client only needs an HTTP endpoint exposing:

- `GET /health`
- `GET /v1/models`
- `POST /v1/infer`

The request is model-size agnostic. To move from Nano to Super, start the serving
backend with `nvidia/Cosmos3-Super` and confirm `/v1/models` reports the Super
checkpoint before reusing the same prompt and payload shape.

## Results / Expected Output

A successful Diffusers smoke test writes one generated image or video artifact
and prints metadata similar to:

```text
model: nvidia/Cosmos3-Nano
backend: diffusers
decoded media bytes: non-zero
seed: fixed integer
```

A successful full-stack Doosan Cosmos3 smoke test prints healthy endpoints for:

- `sim-server`
- a Cosmos3-Nano Reasoner OpenAI-compatible endpoint
- `app-server`
- `frontend`

and exposes the UI at the configured frontend port with non-empty actions and
execution status. In `PALLETIZER_REASONING_MODE=on`, also require a non-empty
reasoning trace in the UI or saved response artifacts. Verify at least one
`STEP_LOG_DIR` entry contains `scenario.txt`, `response.txt`, `action.json`, and
the box image inputs that were sent to the Reasoner. If the stack is running
with `facebook/opt-125m`, the UI can show camera frames and boxes, but it is not
a Cosmos3 pass: the model is not multimodal and can also reject the app's default
token budget.

If the UI shows no actions and no reasoning while the services are healthy,
check these in order:

```bash
curl -fsS http://127.0.0.1:8200/v1/models
curl -fsS http://127.0.0.1:8000/api/status
docker compose -f docker-compose.yml \
  -f docker-compose.test.yml \
  -f /tmp/cosmos3-palletizer-reasoner.override.yml \
  logs --tail=120 app-server
```

Look for a `POST ... /v1/chat/completions` line, then either an `Executing:`
action log or a parser warning followed by a continuation request. If no
Reasoner request appears, the app server is still pointed at the wrong
inference URL. If reasoning appears in the UI or in
`/tmp/palletizer-step-logs/step_*/response.txt` but no new action appears, the
model likely returned a response that the reference parser could not parse.
Check whether the response is truncated inside `<answer>` or JSON. In that case,
raise `MAX_COMPLETION_TOKENS` to at least `1024` and restart `app-server`; a
`512` token budget can be too small for the visual inspection plus action JSON
on three-box prompts. If the prompt has no box images, compare
`/sim/buffer/status` against `/sim/boxes/images_for_ids`; some source revisions
report buffer IDs such as `box_0` while image APIs use zero-padded IDs such as
`box_0000`.

The companion walkthrough includes expected actions and screenshots for:

- damaged cartons routed to `CALL_A_HUMAN`,
- heavy boxes placed on low pallet layers with firm grip,
- mixed-SKU stacks that keep rigid goods below fragile items.

## Dataset

| Name | Source | License | Size |
| --- | --- | --- | --- |
| Synthetic palletizing scenes and box assets | [doosan-robotics/explainable-palletizer](https://github.com/doosan-robotics/explainable-palletizer) | See upstream repository | Small source assets plus Docker/model caches |
| Cosmos3 models | [NVIDIA Cosmos3 collection](https://huggingface.co/collections/nvidia/cosmos3) | NVIDIA Open Model License | Varies by model |

## Safety and Limitations

- This is a simulated proof of concept, not a production robot safety system.
- Cosmos3 generation can help validate scene prompts and expected outputs, but
  real palletizing deployments still need independent safety controls, guarded
  robot execution, and site-specific validation.
- The upstream Doosan project currently keeps its full closed-loop stack in the
  public reference repository; this cookbook does not vendor that source code.
- If the full-stack smoke test fails on a driver, CUDA, or container-runtime
  mismatch, fix the host runtime before treating the robot-loop path as passed.
- If the Diffusers endpoint was just restarted, the first Nano/Super request can
  spend several minutes loading weights before returning a generated artifact.

## Resources

- [doosan-robotics/explainable-palletizer](https://github.com/doosan-robotics/explainable-palletizer)
- [Cosmos3-Nano](https://huggingface.co/nvidia/Cosmos3-Nano)
- [Cosmos3-Super](https://huggingface.co/nvidia/Cosmos3-Super)
- [Cosmos3 Diffusers setup](../README.md#diffusers)
- [Isaac Sim](https://developer.nvidia.com/isaac-sim)
- [cuRobo](https://curobo.org/)
