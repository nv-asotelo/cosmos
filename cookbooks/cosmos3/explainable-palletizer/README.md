# See How It Thinks: Mixed Palletizing with Explainable Visual Reasoning

> **Authors:** Kyungchan Son, Minsoo Song, Yujeong Jeong, and Yuri Rocha -- Doosan Robotics

| Model | Workload | Use case |
| --- | --- | --- |
| Cosmos3 Reasoner, with Cosmos3-Nano as the default profile and Cosmos3-Super as the higher-quality profile | End-to-end reasoning | Explainable mixed-SKU palletizing: visual inspection, handling policy, and bounded action recommendations |

## Overview

Warehouse palletizing often begins with a simple rule: pick the next case,
place it in the next open slot, repeat. That works when every carton is intact
and every item has the same size, weight, and handling requirement. It becomes
brittle when a conveyor includes mixed SKUs, damaged cartons, fragile goods,
ambiguous labels, or products that should not be stacked under heavy loads.

This recipe shows a public Cosmos 3 version of an explainable palletizing
workflow. It sends box images and optional pallet metadata to a Cosmos3
Reasoner endpoint, then saves operator-visible evidence, a structured plan, and
review artifacts. The output is designed for audit and downstream validation:
humans can see what the model noticed, while agents and applications can parse
bounded actions.

<p align="center">
  <img src="./assets/main_workflow.svg" alt="Cosmos3 explainable palletizer workflow" width="900">
</p>

## What You Will Run

The default path is headless. You run or select a Cosmos3 Reasoner endpoint,
point the recipe client at `/v1/chat/completions`, and provide images through a
local folder or manifest. The client writes all artifacts to a local output
directory.

| Component | Role |
| --- | --- |
| Images and manifest | Box crops plus optional weight, size, label, pallet, and handling metadata |
| Prompt/orchestration client | Builds the request, sends images, validates the response shape, and writes artifacts |
| Cosmos3 Reasoner endpoint | Reads the images and metadata, summarizes visible evidence, and emits bounded actions |
| Saved artifacts | `raw_response.txt`, `reasoning_trace.txt`, `plan.json`, `request_summary.json`, and the request payload |
| Optional review UI | Static local browser view over the saved artifacts; it does not call the model |

The control loop for one headless batch is:

1. Collect one or more visible box images.
2. Attach optional box metadata such as weight, dimensions, notes, and pallet
   occupancy.
3. Ask Cosmos3 Reasoner to inspect the images, classify handling risks, and
   choose from allowed action types.
4. Parse the returned JSON and reject incomplete or malformed output.
5. Save artifacts for human review or downstream orchestration.

## Why Explainable Reasoning?

Mixed-SKU palletizing is a long-tail manipulation problem. A rule-based system
can work for one box size and one load pattern, then fail when a supplier
changes packaging or a damaged carton reaches the robot cell.

This recipe focuses on the expensive exceptions:

- **Packaging damage:** open flaps, torn tape, crushed cardboard, contamination,
  or contents at risk of falling out should escalate to a human.
- **Fragility and handling:** glass, electronics, liquids, cans, and paper goods
  need different speed and grip settings.
- **Load quality:** heavy or rigid products should form stable lower layers;
  fragile products should avoid crushing loads.
- **Auditability:** operators need a compact reason for each place, wait, or
  escalation decision.

The recipe never exposes model-internal chain of thought. The optional
`reasoning_trace` artifact is an operator-visible summary of evidence and the
decision rationale. For lower latency, run action-first mode and keep only the
per-action `reason` fields.

## Before You Start

Choose the smallest run mode that answers your question.

| Mode | How to run it | Use when |
| --- | --- | --- |
| Action-first headless run | `PALLETIZER_REASONING_MODE=off` | You need parseable actions with the lowest latency and smallest output budget |
| Compact reasoning headless run | `PALLETIZER_REASONING_MODE=summary` | You want a short operator-visible reasoning summary plus parseable actions |
| Optional review UI | [serve_review_frontend.md](serve_review_frontend.md) | You want a browser view over saved images, evidence, and JSON after inference |
| Scenario validation | [workflow_e2e.md](workflow_e2e.md) | You want to compare model output against damaged-carton, heavy-box, and mixed-SKU expectations |

Use Cosmos3-Nano for the default profile. Use Cosmos3-Super when you want a
higher-quality run and have a serving host that meets the larger model's GPU
requirements.

## Prerequisites

### System Requirements

The Python client can run on a laptop or CPU-only machine when it can reach a
remote Cosmos3 Reasoner endpoint. GPU requirements apply to the machine serving
the model.

| Requirement | Minimum for this recipe |
| --- | --- |
| Cosmos3 Reasoner endpoint | OpenAI-style `/v1/models` and `/v1/chat/completions` routes |
| Default endpoint profile | Cosmos3-Nano Reasoner on port `8200` |
| Higher-quality profile | Cosmos3-Super Reasoner on a larger serving host |
| Client runtime | Python 3.10+; the supplied runner uses only the standard library |
| Inputs | `jpg`, `jpeg`, `png`, or `webp` box images; manifest optional |
| Disk | Enough space for input images and saved text/JSON artifacts |

For the model server, use the shared Cosmos 3 setup guidance in
[`../README.md`](../README.md). Treat Cosmos3-Nano as the smallest documented
profile for this recipe. Cosmos3-Super is usually served with multiple GPUs and
should use the launch profile documented by the shared setup guide.

### Endpoint Check

The recipe defaults to port `8200` so it can sit beside other local examples.
Use the served model ID returned by `/v1/models`.

```bash
export COSMOS3_REASONER_BASE_URL="${COSMOS3_REASONER_BASE_URL:-http://127.0.0.1:8200/v1}"
export COSMOS3_REASONER_MODEL="${COSMOS3_REASONER_MODEL:-nvidia/Cosmos3-Nano}"

curl -fsS "${COSMOS3_REASONER_BASE_URL}/models"
```

### Input Data

For a quick run, place images in a folder:

```bash
export PALLETIZER_IMAGE_DIR=/path/to/my/box-images
```

For repeatable runs, use a manifest. Relative image paths are resolved from the
manifest directory:

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

The runner accepts local folders, mounted folders, or public datasets that have
first been materialized locally. It reads local files, encodes them as data
URIs, and sends those data URIs to the endpoint.

## Quickstart

1. Change to the recipe directory:

```bash
cd cookbooks/cosmos3/explainable-palletizer
```

2. Set the endpoint, model, input, output, and reasoning mode:

```bash
export COSMOS3_REASONER_BASE_URL=http://127.0.0.1:8200/v1
export COSMOS3_REASONER_MODEL=nvidia/Cosmos3-Nano
export PALLETIZER_IMAGE_DIR=/path/to/my/box-images
export PALLETIZER_OUTPUT_DIR=/tmp/cosmos3-palletizer-headless

# Use "off" for the lowest-latency action-first path, or "summary" for a
# compact operator-visible reasoning summary.
export PALLETIZER_REASONING_MODE=summary
export MAX_COMPLETION_TOKENS=1536
```

3. Confirm the endpoint is reachable:

```bash
curl -fsS "${COSMOS3_REASONER_BASE_URL}/models"
```

4. Run the headless client in
[run_custom_images_with_reasoner.md](run_custom_images_with_reasoner.md).

5. Review the outputs:

```bash
ls -1 "${PALLETIZER_OUTPUT_DIR}"
python3 -m json.tool "${PALLETIZER_OUTPUT_DIR}/plan.json" >/dev/null
```

6. Optionally serve the review UI from saved artifacts using
[serve_review_frontend.md](serve_review_frontend.md).

## Output Contract

The recipe asks the model for one JSON object. `reasoning_trace` is either an
empty string in action-first mode or a compact operator-visible summary in
summary mode.

```json
{
  "reasoning_trace": "Concise visible evidence and decision rationale.",
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

Allowed actions:

| Action | Required fields | Meaning |
| --- | --- | --- |
| `PICK_AND_PLACE` | `box_id`, `target_pallet`, `position`, `speed_pct`, `grip_strength`, `reason` | Pick one visible box and place it at a valid pallet position |
| `CALL_A_HUMAN` | `box_ids`, `reason` | Escalate damaged, contaminated, unsealed, or otherwise unsafe boxes |
| `WAIT` | `reason` | Wait when the available images or metadata are insufficient for a safe recommendation |

Field constraints:

| Field | Allowed values |
| --- | --- |
| `box_id` | One of the visible box IDs |
| `target_pallet` | `1` or `2` unless your manifest defines a different pallet set |
| `position` | A grid coordinate from the valid positions supplied in the prompt, or `[0, 0, 0]` when no placement grid is supplied |
| `speed_pct` | `40`, `80`, or `100` |
| `grip_strength` | `gentle`, `standard`, or `firm` |
| `reason` | Short human-readable rationale, ideally under 30 words |

Treat the output as an advisory planning artifact, not a robot command. A
production system still needs independent motion planning, collision checking,
weight validation, site policies, and human-approved exception handling.

## Pipeline Components

| File | Audience | Purpose |
| --- | --- | --- |
| [README.md](README.md) | Humans | Primary recipe: what to run, why it matters, requirements, quickstart, contract, scenarios, and resources |
| [run_custom_images_with_reasoner.md](run_custom_images_with_reasoner.md) | Agents and advanced users | Headless runner, environment variables, local/mounted/public input handling, and notebook usage |
| [serve_review_frontend.md](serve_review_frontend.md) | Agents and advanced users | Optional static review UI from saved artifacts |
| [workflow_e2e.md](workflow_e2e.md) | Humans and agents | Scenario appendix and validation checklist |
| [assets/main_workflow.svg](assets/main_workflow.svg) | Humans | Public architecture diagram |

## Walkthrough Scenarios

Use the scenarios below to sanity-check model behavior. The exact wording may
vary by model profile and prompt budget, but the action type and handling policy
should stay consistent.

### Scenario 1: Damaged Carton To Human Review

**Setup:** Three boxes arrive together. Two are visibly unsafe: one has an open
top flap with detached tape, and another is crushed and deformed. The third is
intact, but unsafe boxes in the same visible set should be escalated first.

**Expected result:** `CALL_A_HUMAN` with the unsafe `box_id` values and a reason
that cites open flaps, detached tape, crushing, deformation, or similar visible
evidence.

```json
{
  "actions": [
    {
      "action": "CALL_A_HUMAN",
      "box_ids": ["box_0000", "box_0002"],
      "reason": "box_0000 has open flaps and detached tape; box_0002 is crushed and deformed."
    }
  ]
}
```

<p align="center">
  <img src="./assets/scenario1.webp" alt="Scenario 1: damaged carton triggers human review" width="900">
</p>

### Scenario 2: Heavy Appliance Box To A Low Stable Slot

**Setup:** Three intact heavy or rigid boxes arrive together. Both pallets are
available. The model should seed a stable base rather than placing heavy goods
on a high layer.

**Expected result:** `PICK_AND_PLACE` for one heavy or rigid box with a low `z`
coordinate, `firm` grip, and a moderate or slow speed.

```json
{
  "actions": [
    {
      "action": "PICK_AND_PLACE",
      "box_id": "box_0001",
      "target_pallet": 1,
      "position": [0, 0, 0],
      "speed_pct": 80,
      "grip_strength": "firm",
      "reason": "Heavy rigid box starts the base layer for stack stability."
    }
  ]
}
```

<p align="center">
  <img src="./assets/scenario2.webp" alt="Scenario 2: heavy box placed at a low layer" width="900">
</p>

### Scenario 3: Mixed-SKU Stacking By Content

**Setup:** A batch includes rigid canned goods, glass containers, and light
snack packaging. The pallet is partially built, so the model must respect the
available cells and avoid crushing delicate items.

**Expected result:** heavy or rigid products are placed before fragile goods;
fragile or crushable items use `gentle` handling and higher layers.

```json
{
  "actions": [
    {
      "action": "PICK_AND_PLACE",
      "box_id": "box_0008",
      "target_pallet": 1,
      "position": [0, 0, 2],
      "speed_pct": 40,
      "grip_strength": "firm",
      "reason": "Canned goods are rigid and can occupy the lowest available stable layer."
    }
  ]
}
```

<p align="center">
  <img src="./assets/scenario3.webp" alt="Scenario 3: mixed-SKU stacking with delicate items deferred upward" width="900">
</p>

## Results

A useful run demonstrates:

- **Visual exception handling:** damaged, unsealed, or unsafe boxes route to
  human review.
- **Content-aware handling:** fragile, heavy, and sturdy products receive
  different speed and grip choices.
- **Structured actions:** the model emits machine-parseable actions checked
  against the prompt contract.
- **Auditable artifacts:** saved JSON and text files make the decision easy to
  review without rerunning inference.

## Next Steps

- Swap the sample SKU set for cartons, labels, weights, and dimensions from
  your own domain.
- Add a manifest generator for your warehouse or benchmark dataset.
- Run action-first mode for latency-sensitive checks, then rerun compact
  reasoning mode on selected cases that need audit detail.
- Connect `plan.json` to your own simulator, planner, or review queue after
  adding independent safety validation.
- Use Cosmos3-Super for higher-quality batch review when serving resources
  allow it.

## Resources

- [Run custom images headlessly](run_custom_images_with_reasoner.md)
- [Serve the optional review UI](serve_review_frontend.md)
- [Scenario and validation appendix](workflow_e2e.md)
- [Shared Cosmos 3 setup](../README.md)
