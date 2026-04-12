# Video Health Spec v26.4 — Overview

A machine-readable specification for training video data quality. Modeled after [SARIF](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) (Static Analysis Results Interchange Format), extended with standards from [Open X-Embodiment](https://robotics-transformer-x.github.io/), [Ego-Exo4D](https://ego-exo4d-data.org/), and [EPIC-KITCHENS](https://epic-kitchens.github.io/).

## Why this exists

Today, organizations acquiring training video rely on custom SDKs, bespoke quality guidelines, or post-acquisition testing to decide whether data meets their needs. Producers and consumers speak different languages — one side ships data they consider clean, the other re-evaluates it with their own criteria, and gaps surface late. The spec replaces this with a shared contract: a shared taxonomy of failure modes, severity levels, and result structure that both parties can reference before data changes hands. Validators differ in how they measure; the spec standardizes what they report.

## Design decisions

- Each finding has a hierarchical **`type`** path: `video-health/{profile}/{code}`. The `code` is a hyphenated phrase prefixed by its category (`quality-*`, `privacy-*`, `safety-*`, `training-*`, `metadata-*`, `multistream-*`, `egoexo-*`, `task-*`, `demo-*`, `trajectory-*`, `segmentation-*`), so the taxonomy is visible in the name itself.
- **`severity`** replaces `status`: `low` / `medium` / `high` / `critical`. `critical` drives the overall result to `REJECT`; `high` / `medium` / `low` drive it to `FLAG`.
- **`details`** replaces the flat `value` / `threshold` / `message` / `evidence` fields with three structured sub-objects: `attachments` (file names), `verbatims` (raw strings observed in the media), and `derived` (computed values, thresholds, confidences, evidence pointers).
- Each `quality-*` code names exactly one failure mode (`quality-missing-moov`, `quality-truncated-file`, `quality-low-bitrate`, `quality-spec-mismatch`, `quality-variable-frame-rate`, …) so a single finding pinpoints what went wrong.

## How it works

1. A **tool** (any validator) processes a **packet** — a bag of aligned files (video, IMU, sidecars) reviewed together
2. The tool evaluates **rules** — each rule measures a specific metric against a threshold
3. If a rule fails, the tool emits a **finding** — an issue with a typed path, severity, description, and structured details
4. All findings are collected into a **result** — a single YAML or JSON document
5. A downstream system reads the result and decides what to do

## Key concepts

### Rules

A rule defines *what to check* and *what thresholds to apply*:

```yaml
- code: quality-blurry        # stable identifier, hyphenated, prefixed by category
  name: Video is blurry       # human-readable name
  metric: blur_score          # what to measure
  type: float                 # data type (see Type System below)
  unit: laplacian_var         # optional — measurement unit
  pass: ">=15"                # threshold for pass
  flag: "5–15"                # threshold for flag
  reject: "<5"                # threshold for reject
  note: ...                   # optional — human context for reviewers
```

Rules say nothing about *how* to compute the metric. That's the tool's job.

### Type system

| Type | Range | Example metrics |
|---|---|---|
| `bool` | `true` / `false` | file_playable, intrinsics_valid, rlds_valid |
| `float` | Any real number | blur_score, brightness, motion_score, duration_sec |
| `ratio` | 0.0 – 1.0 | face_ratio, idle_ratio, overlap_ratio |
| `int` | Non-negative integer | missing_stream_count |

### Threshold syntax

Thresholds are strings with a defined grammar:

| Form | Meaning | Example |
|---|---|---|
| `">=N"` | Greater than or equal to N | `">=15"` |
| `"<=N"` | Less than or equal to N | `"<=120"` |
| `">N"` | Strictly greater than N | `">300"` |
| `"<N"` | Strictly less than N | `"<5"` |
| `"A–B"` | Inclusive range (en-dash, U+2013) | `"5–15"` |
| `"N"` | Exact value | `"0"` |
| `true` / `false` | Boolean (unquoted YAML) | `true` |

Ranges use en-dashes (`–`), not hyphens (`-`), to avoid ambiguity with negative numbers.

### Findings

A finding is one issue that was detected. An empty `findings[]` means no issues were found.

| Field | Required | Description |
|---|---|---|
| `type` | yes | `video-health/{profile}/{code}` — hierarchical issue identifier |
| `severity` | yes | `low`, `medium`, `high`, or `critical` |
| `description` | yes | Human-readable explanation |
| `details.attachments` | no | File names relevant to the issue (frames, logs) |
| `details.verbatims` | no | Raw strings observed in container or media (`observed_codec`, `missing_atom`, etc.) |
| `details.derived` | no | Computed values, thresholds, confidences, and evidence pointers |

Example:

```yaml
- type: video-health/manipulation/privacy-face-visible
  severity: critical
  description: Identifiable face visible in 30% of frames
  details:
    attachments: []
    verbatims: {}
    derived:
      face_ratio: 0.30
      threshold: 0.25
      confidence: 0.87
      evidence_frames: [6420, 6450]
```

### Profiles

A profile is a self-contained set of rules with **default thresholds**. No inheritance — copy and edit.

| Profile | Use case |
|---|---|
| **base** | Any training video |
| **egocentric** | Head-mounted, hand manipulation |
| **manipulation** | Robot demonstration + RLDS trajectory |
| **multiview** | Multi-camera sync + ego-exo overlap |
| **pedestrian** | Ego navigation — sidewalks, crosswalks, indoor wayfinding |
| **delivery** | Delivery robots — sidewalk bots, last-mile autonomous vehicles |
| **aerial** | Drone / aerial — mapping, inspection, overflight |

Profile thresholds are informed defaults. Override them with **variants** (see below).

### Variants

A variant customizes thresholds for a specific deployment without forking the profile.

**Two kinds:**

1. **Override variant** — changes thresholds on an existing profile:

```yaml
name: high-precision
base_profile: manipulation

overrides:
  - code: quality-blurry
    pass: ">=20"       # 2x stricter than default
    flag: "10–20"
    reject: "<10"
```

2. **Add-on variant** — defines new rules to append to any profile:

```yaml
name: segmentation
applies_to: any

rules:
  - code: segmentation-scene-too-short
    name: Scene too short
    metric: scene_min_sec
    type: float
    pass: ">=3"
    flag: "1–3"
    reject: "<1"
```

Results report which variant was applied:

```yaml
version: "26.4"
profile: manipulation
variant: high-precision
status: REJECT
```

**Included variants:**

| Variant | Type | Use case |
|---|---|---|
| [home-chores](variants/home-chores.yaml) | override | Crowdsourced household recordings (Micro1, 1X) |
| [home-cleaning](variants/home-cleaning.yaml) | override | Household cleaning — strict privacy for private homes |
| [factory](variants/factory.yaml) | override | Industrial settings with consented workers (Build AI) |
| [kitchen](variants/kitchen.yaml) | override | Cooking / food prep — sharp-object threshold relaxed, steam-tolerant |
| [high-precision](variants/high-precision.yaml) | override | Fine-maneuverability sub-mm assembly |
| [auto-repair](variants/auto-repair.yaml) | override | Automotive service — tools expected, wide exposure range |
| [construction](variants/construction.yaml) | override | Building sites — dust, long sessions, coarse narration |
| [agriculture](variants/agriculture.yaml) | override | Outdoor farming — crop inspection, harvesting, long sessions |
| [yard-work](variants/yard-work.yaml) | override | Lawn care, gardening, landscaping — outdoor home tasks |
| [retail-stocking](variants/retail-stocking.yaml) | override | Store shelf stocking — visible text expected, strict customer privacy |
| [warehouse](variants/warehouse.yaml) | override | Warehouse picking/packing — industrial indoor, consented workers |
| [segmentation](variants/segmentation.yaml) | add-on | Scene boundary and caption quality |

### Collections, packets, and assets

A result is organized around a three-level data model:

1. **Collection** — an optional dataset-level grouping. Typically an S3 bucket folder (or GCS prefix, or local path) that holds many packets. Carries only `id` and `uri`. Profile and variant do *not* live here — they describe the review, not the dataset, and stay at the top of the result document.
2. **Packet** — the review unit. A bag of aligned files that belong together and are reviewed as one thing (e.g. two videos from different angles and an IMU sidecar). Has `id`, `uri`, an `assets[]` array, and optional profile-specific annotation blocks (`task`, `trajectory`, `navigation`, `flight`).
3. **Assets** — the individual files inside a packet. Each carries only `uri` and `type`. All other per-asset properties (fps, duration, codec, IMU rate, viewpoint, lens model, …) live elsewhere — see "Asset annotations" below.

### Asset annotations: sidecar or embedded

`packet.assets[]` items deliberately carry only `uri` and `type`. Per-asset properties live in one of two places and the spec makes no attempt to enumerate them up-front:

1. **Sidecar file inside the packet.** A JSON or YAML file listing per-asset metadata, e.g. `chore_017.yaml` or a packet-wide `manifest.yaml`. The sidecar may itself be listed in `assets[]` with `type: annotation`.
2. **Embedded in the asset file itself.** MP4 atoms for fps / duration / codec, EXIF tags for camera intrinsics, RLDS schema fields for trajectory data, and so on.

When a rule needs an asset property, the validator reads it from whichever of the two is available — the spec does not mandate where. Computed values and thresholds flow into `findings[].details.derived` and top-level `metrics` as usual. This is what keeps the asset shape minimal: a new modality works by picking a `type` string. No schema bump required for lidar, event cameras, thermal, or anything else that shows up later.

### WebDataset and sharded collections

For producers that ship data as sharded tar files (e.g. Build AI's WebDataset shards), the packet URI uses the RFC 3986 fragment form to address a sample inside a tar:

- `packet.uri: s3://buildai/ego/chores/shard-00042.tar#chore_017` — the shard URI is everything before the `#`, the sample key is everything after. A consumer splits on `#` to recover both.
- `assets[].uri: s3://buildai/ego/chores/shard-00042.tar#chore_017.mp4` — each asset inside the same sample follows the same convention.

A collection of shards is addressed at its root:

- `collection.uri: s3://buildai/ego/chores/` — the prefix under which all shards live. Shards are a transport-level chunking of the collection. Per-shard aggregates (dashboards, quota gating) are a consumer concern; this spec only defines per-packet results, and consumers aggregate externally.

Folder-style packets also work — `packet.uri: take_042/` for a multiview session whose streams live as separate files under a shared directory.

### Result document

The complete output of one validation run:

| Field | Required | Description |
|---|---|---|
| `version` | yes | Schema version (`"26.4"`) |
| `profile` | yes | Which profile was applied (review-level) |
| `variant` | no | Which variant was applied on top (review-level) |
| `status` | yes | `PASS`, `FLAG`, or `REJECT` (worst severity wins) |
| `tool` | no | `{name, version, url}` — which validator produced this |
| `collection` | no | `{id, uri}` — optional dataset-level grouping |
| `packet` | yes | `{id, uri, assets[], task?, trajectory?, navigation?, flight?}` — the review unit |
| `findings` | yes | Array of issues. Empty array means no issues were found. |
| `metrics` | no | All measured values, keyed by metric name |
| `timing` | no | Duration of each validation phase in seconds |

**Packet sub-blocks.** `packet.task`, `packet.trajectory`, `packet.navigation`, and `packet.flight` are all optional profile-specific annotation blocks. They live inside `packet:` because they describe the review unit, not the review itself.

**Status derivation:** any finding with `severity: critical` → overall `REJECT`. Else any finding (severity `high` / `medium` / `low`) → overall `FLAG`. Else (empty `findings[]`) → `PASS`.

## Taxonomy (code prefixes)

Codes are grouped by category, which is encoded as a prefix on the code itself.

| Prefix | What |
|---|---|
| `quality-*` | File integrity, codec, container metadata, blur, exposure, blockage, duration, intrinsics, IMU, depth. Distinct sub-modes per failure: `quality-missing-moov`, `quality-corrupt-header`, `quality-metadata-rotation`, `quality-missing-imu`, `quality-imu-desync`, `quality-missing-depth`, `quality-blurry`, etc. |
| `privacy-*` | Bathroom, reflective surface, phone, face, person closeup, visible text, document, screen, speech, property overflight, license plate |
| `safety-*` | Unsafe sharp objects |
| `training-*` | Person visible, hands visible, static, black frames, frozen |
| `metadata-*` | Sidecar metadata mismatch |
| `multistream-*` | Sync drift, FPS, frame drops, duration, calibration |
| `egoexo-*` | Egomotion/IMU, dense narration, view overlap |
| `task-*` | Maneuverability classification (coarse vs fine) |
| `demo-*` | Task completion, motion, manipulation, idle, viewpoint |
| `trajectory-*` | RLDS episode structure, 7-DOF action vectors, reward tags |
| `nav-*` | GPS validity and drift, obstacle annotation, path continuity, surface tagging |
| `delivery-*` | GPS, route completion, weather tagging, address visibility |
| `flight-*` | Altitude, GPS, flight stability, gimbal drift |
| `segmentation-*` | Scene length, caption quality, boundary accuracy, coverage |

See the per-profile ruleset YAMLs for the exact list of codes and their thresholds.

## Trajectory & RLDS compliance

To train cross-embodiment models (RT-X, Octo), video data must map to reinforcement learning structures.

**Episode structure (`trajectory-invalid-episode`):** Each video is chunked into episodes containing steps. Each step must include: `[observation (image array), state (joint positions), action (deltas), reward, language_instruction]`.

**Action vector format (`trajectory-invalid-action-vector`):** Spatial manipulation maps to a canonical 7-dimensional vector: `(x, y, z, roll, pitch, yaw, gripper_state)` per the Open X-Embodiment standard.

**Reward annotation (`trajectory-missing-reward`):** Each episode's final frame must include a binary sparse reward value — `1.0` for task success, `0.0` for failure or incomplete.

## Ego-Exo synchronization

When merging egocentric and exocentric views, the spatial relationship is critical.

**Egomotion (`egoexo-egomotion-missing`):** Head-mounted cameras require SLAM or high-frequency IMU data to compute exact rig egomotion. Without it, models cannot separate head movement from hand movement — automatic reject in egocentric/manipulation profiles, flag in multiview (static exo cameras exempt).

**Dense narration (`egoexo-sparse-narration`):** Single-label descriptions ("making a sandwich") are insufficient. The specification requires timestamped micro-annotations ("opening jar at 12.3s", "spreading peanut butter at 15.1s"). Minimum 2 annotations per minute.

**View overlap (`egoexo-insufficient-view-overlap`):** Multi-view setups must provide sufficient visual overlap for 3D point cloud registration. If a critical interaction occurs outside the overlapping volume between ego and exo views, the sequence fails.

## Task granularity

**Maneuverability classification (`task-maneuverability-untagged`):** Videos must be tagged by physical precision level:

| Level | Examples | QA impact |
|---|---|---|
| Coarse | Pushing a box, pouring from a bottle | Standard blur thresholds |
| Fine | Threading a needle, inserting USB, snap-fit assembly | Stricter blur/focus — use `high-precision` variant |

## Segmentation quality

Six rules under the `segmentation-*` prefix validate how long recordings are split into scenes with captions. Defined in the [segmentation variant](variants/segmentation.yaml), applicable to any profile.

Covers: scene duration bounds, caption specificity, false/missed boundaries, and temporal coverage gaps.

## Privacy & ethics guardrails

Beyond visual de-identification (`privacy-face-visible`, `privacy-reflective-surface`), the spec enforces:

**Auditory sanitization (`privacy-speech-audio`):** Egocentric mics capture ambient conversations — a legal liability at scale. Unless audio is strictly necessary for the physical task (e.g. listening for a snap-fit), the audio track must be stripped or scrubbed of human speech.

**Algorithmic de-identification pipeline:** Before human QA review, vendors should run automated blurring over faces, license plates, screens, and reflective surfaces. The human reviewer's job is to catch what the algorithm missed. The `privacy-face-visible`, `privacy-screen-visible`, and `privacy-reflective-surface` rules validate the output of this pipeline.

## Intentional severity differences across profiles

Some rules deliberately vary in severity between profiles:

| Rule | base | egocentric | manipulation | multiview | Reason |
|---|---|---|---|---|---|
| `quality-invalid-intrinsics` | flag | reject | reject | reject | Consumer cameras have known fixed intrinsics; rigs need calibration |
| `egoexo-egomotion-missing` | — | reject | reject | flag | Static exocentric cameras don't need IMU data |
| `quality-blurry` | >=15 | >=10 | >=10 | >=15 | Fisheye lenses are inherently softer |

These are deliberate design choices, not bugs. Override them in a variant if your deployment needs different behavior.
