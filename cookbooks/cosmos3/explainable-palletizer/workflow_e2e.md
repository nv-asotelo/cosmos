# Scenario And Validation Appendix

This appendix supports the primary recipe in [README.md](README.md). Use it to
validate the headless Cosmos3 Reasoner run against palletizing scenarios that
matter for mixed-SKU handling.

## Public Flow Under Test

1. Read a local image folder or manifest.
2. Build one request with box images, optional metadata, pallet state, and valid
   placement cells.
3. Send the request to the Cosmos3 Reasoner `/v1/chat/completions` endpoint.
4. Parse one JSON object containing `boxes`, `actions`, and optional
   `reasoning_trace`.
5. Save `raw_response.txt`, `reasoning_trace.txt`, `plan.json`,
   `request_summary.json`, and `request_payload.json`.

## Action Contract

The model must choose from three action types:

| Action | Required fields | Valid when |
| --- | --- | --- |
| `PICK_AND_PLACE` | `box_id`, `target_pallet`, `position`, `speed_pct`, `grip_strength`, `reason` | A visible box is intact and has a valid placement |
| `CALL_A_HUMAN` | `box_ids`, `reason` | One or more visible boxes are damaged, unsealed, contaminated, unstable, or ambiguous enough to need review |
| `WAIT` | `reason` | The current inputs are insufficient for a safe recommendation |

Use the same grid and handling constraints as the README:

- `speed_pct`: `40`, `80`, or `100`
- `grip_strength`: `gentle`, `standard`, or `firm`
- `target_pallet`: a pallet ID from the prompt
- `position`: one of the valid positions supplied in the prompt, or
  `[0, 0, 0]` when no grid is supplied

## Reasoning Modes

The recipe supports two prompt shapes:

| Mode | Environment | Expected output |
| --- | --- | --- |
| Action-first | `PALLETIZER_REASONING_MODE=off` | `reasoning_trace` is empty; each action still has a short `reason` |
| Compact reasoning | `PALLETIZER_REASONING_MODE=summary` | `reasoning_trace` summarizes visible evidence and decisions in a bounded paragraph |

Action-first mode is the fastest path. Compact reasoning mode is better when a
human needs to audit why a box was placed, delayed, or escalated. Avoid verbose
free-form traces in this recipe; they increase latency and can crowd out the
JSON object.

## Scenario 1: Damaged Carton To Human Review

**Setup:** Three boxes arrive together. Two are unsafe: one has an open top
flap with detached tape, and another is crushed and deformed. The third box is
intact.

**Expected behavior:**

- The model marks the damaged boxes as not pickable.
- The action is `CALL_A_HUMAN`.
- The action cites visible damage such as open flaps, detached tape, crushing,
  deformation, or contents at risk of falling out.
- It does not place an intact box while unsafe boxes remain in the same visible
  batch unless the surrounding application has already removed them.

**Example action:**

```json
{
  "action": "CALL_A_HUMAN",
  "box_ids": ["box_0000", "box_0002"],
  "reason": "box_0000 has open flaps and detached tape; box_0002 is crushed and deformed."
}
```

<p align="center">
  <img src="./assets/scenario1.webp" alt="Damaged cartons should be escalated" width="900">
</p>

## Scenario 2: Heavy Or Rigid Goods To Lower Layers

**Setup:** Three intact heavy or rigid boxes arrive together. The pallets have
open base-layer cells.

**Expected behavior:**

- The model identifies heavy, rigid, or canned goods as suitable lower-layer
  candidates.
- The action is `PICK_AND_PLACE`.
- The selected position has the lowest valid `z` coordinate for that box.
- The handling is usually `firm` grip and `40` or `80` speed.

**Example action:**

```json
{
  "action": "PICK_AND_PLACE",
  "box_id": "box_0001",
  "target_pallet": 1,
  "position": [0, 0, 0],
  "speed_pct": 80,
  "grip_strength": "firm",
  "reason": "Heavy rigid box starts the base layer for stack stability."
}
```

<p align="center">
  <img src="./assets/scenario2.webp" alt="Heavy goods should form lower layers" width="900">
</p>

## Scenario 3: Mixed-SKU Stacking By Content

**Setup:** A batch includes rigid canned goods, glass containers, and light
snack packaging. One pallet is partially filled.

**Expected behavior:**

- Heavy or rigid items are placed before fragile goods.
- Cans or dense products prefer low or lowest available stable layers.
- Glass, electronics, or crushable goods use `gentle` handling and higher
  layers when available.
- The model chooses from valid placement cells rather than inventing
  coordinates.

**Example action:**

```json
{
  "action": "PICK_AND_PLACE",
  "box_id": "box_0008",
  "target_pallet": 1,
  "position": [0, 0, 2],
  "speed_pct": 40,
  "grip_strength": "firm",
  "reason": "Canned goods are rigid and can occupy the lowest available stable layer."
}
```

<p align="center">
  <img src="./assets/scenario3.webp" alt="Mixed-SKU stacking should respect content and layer rules" width="900">
</p>

## Validation Checklist

Run this checklist after the headless client writes artifacts:

- `request_summary.json` records the endpoint base URL, served model ID,
  reasoning mode, elapsed time, and visible box IDs.
- `raw_response.txt` is non-empty.
- `plan.json` is valid JSON.
- `plan.json` has `boxes` and `actions` arrays.
- Every action uses one of `PICK_AND_PLACE`, `CALL_A_HUMAN`, or `WAIT`.
- Every `PICK_AND_PLACE` action names a visible box and includes
  `target_pallet`, `position`, `speed_pct`, `grip_strength`, and `reason`.
- Damaged or unsealed cartons are escalated.
- Heavy or rigid goods go to lower layers.
- Fragile or crushable goods are handled gently and placed above heavier goods
  when possible.
- Compact reasoning mode stays concise enough that the JSON object is complete.

## Useful Failure Signals

| Symptom | Likely cause | Next step |
| --- | --- | --- |
| `plan.json` is missing | The endpoint timed out or returned no JSON object | Check `raw_response.txt`, reduce batch size, or use action-first mode |
| JSON is truncated | Output budget is too small or trace is too verbose | Increase `MAX_COMPLETION_TOKENS` or set `PALLETIZER_REASONING_MODE=off` |
| Actions lag behind visible boxes | Request latency is higher than the image arrival rate | Batch fewer images, use action-first mode, or slow the upstream image feed |
| Damaged boxes are placed | Prompt omitted damage policy or images hide the damage | Add metadata notes, use clearer crops, and rerun the scenario |
| Fragile goods are placed low under heavy goods | Pallet state or valid positions are missing | Include valid cells and layer occupancy in the manifest |
