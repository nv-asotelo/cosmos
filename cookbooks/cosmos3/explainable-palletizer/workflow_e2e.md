# Decision Walkthrough and Validation Criteria

This guide keeps the palletizing decision criteria from the original Doosan
Robotics explainable-palletizer demo, but adapts them to the public Cosmos3
headless recipe. The runnable path is:

1. Prepare box images and optional manifest metadata.
2. Send them to a standalone Cosmos3 Reasoner `/v1/chat/completions` endpoint.
3. Save the reasoning trace, structured per-box evidence, and action plan.
4. Review the output against the criteria below.

There is no external demo-stack requirement in this public recipe.

## Original Backend Note

The original merged cookbook used the upstream demo's `inference-server`: a
vLLM OpenAI-compatible server on port `8200` with
`INFERENCE_MODEL=nvidia/Cosmos-Reason2-8B` by default, `nvidia/Cosmos-Reason2-2B`
as the smaller option, and optional Cosmos Reason 2 LoRA adapters.

That backend is useful historical context, but it is not Cosmos3-compatible as
configured. This recipe keeps the same OpenAI-compatible API pattern and port,
then swaps the backend to a standalone Cosmos3 Reasoner server. A valid Cosmos3
run should show `/v1/models` returning the served Cosmos3 Reasoner model ID and
the saved request summary should record that model ID.

## Public Control Flow

| Stage | Responsibility |
| --- | --- |
| Image set or manifest | Provides box IDs, image files, optional weights, dimensions, notes, and pallet state |
| Headless client | Encodes images as data URIs, builds the palletizing prompt, calls `/v1/chat/completions`, and saves artifacts |
| Cosmos3 Reasoner endpoint | Inspects images, explains visible evidence, and emits one parseable JSON object |
| Review artifacts | `raw_response.txt`, `reasoning_trace.txt`, `plan.json`, and `request_summary.json` |

The headless client validates model reasoning and formatting only. It does not
execute robot motion, compute valid robot trajectories, or certify a real
workcell.

## Action Contract

The recipe asks for three action types:

| Action | Meaning |
| --- | --- |
| `PICK_AND_PLACE` | Recommend placing one box at a stated pallet position with speed and grip guidance |
| `CALL_A_HUMAN` | Escalate damaged, contaminated, unsealed, ambiguous, or otherwise unsafe boxes |
| `WAIT` | Defer when too little information is visible or metadata is insufficient for a recommendation |

The action JSON should include a concise operator-visible reason. If dimensions
or weights are missing, the model should say what is uncertain and choose
`WAIT` or `CALL_A_HUMAN` when a safe recommendation cannot be justified from
visible evidence.

## Scenario 1: Damaged Carton

**Expected action:** `CALL_A_HUMAN`

Three boxes arrive together. Two are visibly unsafe: one has an open top flap
with detached tape, and another is crushed and deformed. The intact box should
not cause the system to ignore the unsafe buffer state.

| Field | Value |
| --- | --- |
| Visible boxes | `box_0000`, `box_0001`, `box_0002` |
| Visible condition | `box_0000`: open flap, detached tape; `box_0001`: intact; `box_0002`: crushed, deformed |
| Expected rationale | Escalate visibly damaged or unsealed boxes before any placement recommendation |

Parsed action:

```json
{
  "action": "CALL_A_HUMAN",
  "box_id": null,
  "reason": "box_0000 has open flaps and detached tape; box_0002 is crushed and deformed"
}
```

<p align="center">
  <img src="./assets/scenario1.webp" alt="Damaged carton scenario" width="900">
</p>

## Scenario 2: Heavy Appliance Box

**Expected action:** `PICK_AND_PLACE` at a low `z` position with a firm grip.

Three intact heavy or sturdy boxes arrive together: a metal tool set, a
36-can case of canned beans, and a 25 kg set of rubber-coated weight plates.
Heavy, rigid boxes should seed the base layer before lighter or fragile goods.

| Field | Value |
| --- | --- |
| Visible boxes | `box_0001` (tool set), `box_0003` (canned beans), `box_0004` (weight plates) |
| Expected rationale | Heavy and sturdy boxes belong low in the stack for stability |
| Expected handling | Lower speed, firm grip, base-layer position |

Parsed action:

```json
{
  "action": "PICK_AND_PLACE",
  "box_id": "box_0001",
  "target_pallet": 1,
  "position": [0, 0, 0],
  "speed_pct": 80,
  "grip_strength": "firm",
  "reason": "Heavy sturdy box; place on the base layer to improve stack stability."
}
```

<p align="center">
  <img src="./assets/scenario2.webp" alt="Heavy appliance scenario" width="900">
</p>

## Scenario 3: Mixed-SKU Stacking

**Expected action:** place heavy or rigid items below fragile items.

Three intact boxes arrive: a 10-pack of SPAM cans, a 4-pack of glass kimchi
fermentation jars, and a multi-pack of honey butter chips. The next action
should preserve stack quality by placing heavier rigid goods at the lowest
available stable slot and reserving higher layers for fragile goods.

| Field | Value |
| --- | --- |
| Visible boxes | `box_0008` (SPAM cans), `box_0010` (glass jars), `box_0011` (chip multipack) |
| Expected rationale | Canned goods are rigid; glass and chips require gentler handling and should avoid crushing loads |
| Expected handling | Firm or standard grip for cans; gentle top-layer handling for fragile goods |

Parsed action:

```json
{
  "action": "PICK_AND_PLACE",
  "box_id": "box_0008",
  "target_pallet": 1,
  "position": [0, 0, 2],
  "speed_pct": 40,
  "grip_strength": "firm",
  "reason": "SPAM cans are heavy and rigid; fragile boxes should be deferred to higher layers."
}
```

<p align="center">
  <img src="./assets/scenario3.webp" alt="Mixed-SKU scenario" width="900">
</p>

## What To Check

- The model refers only to visible evidence and supplied manifest metadata.
- Damaged, open, crushed, contaminated, or ambiguous packaging is escalated.
- Heavy or rigid goods are prioritized below fragile goods.
- The reasoning trace and action reason agree with each other.
- `plan.json` contains exactly one parseable JSON object.
- The action recommendation does not claim robot safety certification.

## Reasoning And Token Budget

Verbose reasoning can make responses slow and can increase the chance that JSON
is truncated. Keep batches small, start with three to six images, and use a
moderate `MAX_COMPLETION_TOKENS` value such as `2048`. If the model repeatedly
produces useful reasoning but incomplete JSON, split the workflow into two
calls: one visual audit call and one action-only JSON extraction call.

## Limitations

- The public headless recipe is not a robot controller.
- It does not replace motion planning, pallet stability checks, or site safety
  systems.
- The original Cosmos Reason 2 inference container, model IDs, and LoRA adapters
  are not Cosmos3-compatible as-is.
