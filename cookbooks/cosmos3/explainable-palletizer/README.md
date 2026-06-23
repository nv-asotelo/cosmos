# See How It Thinks: Mixed Palletizing with Explainable Visual Reasoning

This public cookbook runs an explainable mixed-SKU palletizing workflow with a
standalone Cosmos3 Reasoner endpoint. It sends box images plus optional manifest
metadata to an OpenAI-compatible `/v1/chat/completions` server, then writes an
operator-visible reasoning trace and structured action recommendations.

This recipe does not use Diffusers and does not require any external demo stack.
The Doosan Robotics project is credited as the scenario lineage, but the
runnable path here is a portable, public headless client.

| Model | Workload | Use case |
| --- | --- | --- |
| [Cosmos3-Nano](https://huggingface.co/nvidia/Cosmos3-Nano) Reasoner, [vLLM](https://docs.vllm.ai/) or [Cosmos3 Reasoner NIM](https://catalog.ngc.nvidia.com/orgs/nim/teams/nvidia/containers/cosmos3-reasoner) | Reasoner | Inspect palletizing box images, explain handling risks, and emit bounded action recommendations |

**Scenario lineage:** adapted from the Doosan Robotics explainable palletizer
Cosmos Cookoff recipe by Kyungchan Son, Minsoo Song, Yujeong Jeong, and Yuri
Rocha.

## Backend Compatibility

Minsoo's original recipe used an `inference-server` container from the upstream
demo. That server is a vLLM OpenAI-compatible API on port `8200`, but its
default model and environment are for Cosmos Reason 2:

- `INFERENCE_MODEL=nvidia/Cosmos-Reason2-8B` by default.
- `nvidia/Cosmos-Reason2-2B` for smaller GPU runs.
- Optional `yurirocha15/Cosmos-Reason2-*-palletizer-lora` adapters.
- Prompt and output parsing tuned for the upstream Doosan app.

That original container is not a drop-in Cosmos3 backend. For this Cosmos3
cookbook, use a standalone Cosmos3 Reasoner endpoint and point the headless
client at its `/v1` base URL. Do not reuse the Cosmos Reason 2 model IDs, LoRA
adapters, or external app wiring as evidence that Cosmos3 is running.

## What You Will Run

- Start or select a Cosmos3-Nano Reasoner server on port `8200`.
- Send your own box images, weights, dimensions, pallet state, and handling notes
  without a demo frontend.
- Save `raw_response.txt`, `reasoning_trace.txt`, `plan.json`, and
  `request_summary.json` for review.
- Compare the model output against damaged-carton, heavy-box, and mixed-SKU
  validation criteria in [workflow_e2e.md](workflow_e2e.md).

## Prerequisites

- A Linux GPU host with enough VRAM for the selected Cosmos3 Reasoner endpoint.
- Hugging Face access for the Cosmos3 model, or access to the Cosmos3 Reasoner
  NIM container.
- A vLLM or NIM endpoint exposing OpenAI-compatible `/v1/models` and
  `/v1/chat/completions`.
- Local images in `jpg`, `jpeg`, `png`, or `webp` format.

## Start The Reasoner

Use the shared [vLLM Reasoner setup](../README.md#vllm) or
[NIM Reasoner setup](../README.md#nim). The recipe defaults to port `8200`:

```bash
export COSMOS3_REASONER_BASE_URL=http://127.0.0.1:8200/v1
export COSMOS3_REASONER_MODEL=nvidia/Cosmos3-Nano
curl -fsS "${COSMOS3_REASONER_BASE_URL}/models"
```

A vLLM launch looks like this when you already have the Cosmos3-Nano Reasoner
checkpoint available locally:

```bash
export COSMOS3_REASONER_MODEL_PATH="${COSMOS3_REASONER_MODEL_PATH:-/path/to/Cosmos3-Nano-Reasoner}"

vllm serve "${COSMOS3_REASONER_MODEL_PATH}" \
  --served-model-name nvidia/Cosmos3-Nano \
  --host 0.0.0.0 \
  --port 8200 \
  --max-model-len 8192 \
  --media-io-kwargs '{"video": {"num_frames": -1}}' \
  --reasoning-parser qwen3 \
  --gpu-memory-utilization 0.55
```

Use the exact model ID returned by `/v1/models` if your server advertises a
different served name.

## Prepare Images

For quick experiments, put image files in one directory. The filename stem
becomes the `box_id`:

```bash
export PALLETIZER_IMAGE_DIR=/path/to/my/box-images
```

For repeatable tests, provide a manifest with image paths and optional box
metadata. Relative paths are resolved from the manifest directory:

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
    }
  ]
}
```

```bash
export PALLETIZER_MANIFEST=/path/to/manifest.json
```

## Run The Headless Client

```bash
cd cookbooks/cosmos3/explainable-palletizer
export COSMOS3_REASONER_BASE_URL=http://127.0.0.1:8200/v1
export COSMOS3_REASONER_MODEL=nvidia/Cosmos3-Nano
export PALLETIZER_OUTPUT_DIR=/tmp/cosmos3-palletizer-headless
```

Then run [run_custom_images_with_reasoner.md](run_custom_images_with_reasoner.md).
The Python block in that file uses only the standard library, so it can run as a
shell smoke test or be copied into a notebook.

## Output Contract

The public client asks the model for one JSON object:

```json
{
  "reasoning_trace": "Concise operator-visible evidence and decision rationale.",
  "boxes": [
    {
      "box_id": "box_0000",
      "visible_evidence": "text and package condition visible in the image",
      "contents_guess": "what the label or appearance suggests",
      "pickable": true,
      "risks": ["fragile"],
      "handling": "gentle"
    }
  ],
  "actions": [
    {
      "action": "PICK_AND_PLACE",
      "box_id": "box_0000",
      "target_pallet": 1,
      "position": [0, 0, 0],
      "speed_pct": 80,
      "grip_strength": "gentle",
      "reason": "brief operator-visible reason"
    }
  ]
}
```

Allowed actions are `PICK_AND_PLACE`, `CALL_A_HUMAN`, and `WAIT`. Treat the
result as an advisory planning artifact, not as a robot command. A production
robot system still needs a separate motion planner, collision checker, site
rules, weight limits, and human-approved exception handling.

## Architecture

<p align="center">
  <img src="./assets/main_workflow.svg" alt="Public headless explainable palletizer workflow" width="900">
</p>

## Validation

Use [workflow_e2e.md](workflow_e2e.md) to check whether output is useful:

- Box labels and visible package condition are reflected in the evidence.
- Unsafe boxes produce `CALL_A_HUMAN`.
- Heavy or rigid boxes are placed before fragile items.
- The JSON object is complete and parseable.
- The saved artifacts include the model ID, input box IDs, and elapsed time.

## Limitations

- This is a proof-of-concept reasoning recipe, not a production robot safety
  system.
- The headless client does not execute robot motion or validate collision-free
  trajectories.
- Prompt shape and token count reduce, but cannot eliminate, the risk of
  incomplete JSON from free-form reasoning responses. Use structured output or a
  follow-up action-only call for stricter production extraction.
