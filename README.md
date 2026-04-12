<p align="center">
  <img src="logo.png" alt="Video Health Spec" width="400">
</p>

# Video Health Spec <sup>v26.4</sup>

Training video gets passed between teams with no shared definition of "good." One side ships data they consider clean; the other re-evaluates it with their own criteria. Gaps surface late. This spec fixes that — a shared taxonomy of failure modes so both sides can agree on what to check before data changes hands.

YAML throughout. 7 profiles, 12 variants. Results describe a **packet** — a bag of aligned files (video, IMU, sidecars) reviewed together — which makes it a natural fit for WebDataset and other sharded distribution formats.

## What it does

A validator processes a **packet** (a bag of aligned files — video, IMU, sidecars — reviewed together), evaluates rules, and emits **findings**. Each finding has a typed path (`video-health/{profile}/{code}`), a severity, and structured details. An empty findings array means no issues were found.

```yaml
version: "26.4"
profile: egocentric
status: REJECT

collection:
  id: buildai-chores-2026q2
  uri: s3://buildai/ego/chores/

packet:
  id: chore_017
  uri: s3://buildai/ego/chores/shard-00042.tar#chore_017
  assets:
    - { uri: chore_017.mp4,  type: video }
    - { uri: chore_017.imu,  type: imu }
    - { uri: chore_017.yaml, type: annotation }

findings:
  - type: video-health/egocentric/privacy-face-visible
    severity: critical
    description: Identifiable face detected
    details:
      attachments: []
      verbatims: {}
      derived:
        face_ratio: 0.34
        threshold: 0.25
        confidence: 0.91

  - type: video-health/egocentric/safety-sharp-object
    severity: high
    description: Sharp object detected (knife)
    details:
      attachments: []
      verbatims: {}
      derived:
        sharp_object_ratio: 0.08
        threshold: 0.05
        knife_ratio: 0.08
        scissors_ratio: 0.00

  - type: video-health/egocentric/privacy-bathroom-visible
    severity: critical
    description: Bathroom fixtures visible
    details:
      attachments: []
      verbatims: {}
      derived:
        bathroom_ratio: 0.12
        threshold: 0.10

  - type: video-health/egocentric/training-no-hands-visible
    severity: high
    description: No hands visible
    details:
      attachments: []
      verbatims: {}
      derived:
        person_ratio: 0.32
        threshold: 0.50
```

## Findings

Each entry in `findings[]` is an issue. Empty array = no issues found.

| Field | What it is |
|---|---|
| `type` | `video-health/{profile}/{code}` — the code is a hyphenated phrase prefixed by category (`quality-*`, `privacy-*`, `safety-*`, etc.) |
| `severity` | `low` / `medium` / `high` / `critical` — `critical` means `REJECT`; anything else means `FLAG` |
| `description` | Human-readable explanation |
| `details.attachments` | File names (dumped frames, logs) |
| `details.verbatims` | Raw strings from the container or media (`observed_codec: hevc`) |
| `details.derived` | Computed values, thresholds, confidences (`blur_score: 3.2`, `threshold: 5.0`) |

## Profiles

Self-contained YAML rulesets with default thresholds. No inheritance — copy and edit.

| Profile | For |
|---|---|
| [base](rulesets/base.yaml) | Any training video |
| [egocentric](rulesets/egocentric.yaml) | Head-mounted, hand manipulation |
| [manipulation](rulesets/manipulation.yaml) | Robot demonstration + RLDS trajectory |
| [multiview](rulesets/multiview.yaml) | Multi-camera sync + ego-exo overlap |
| [pedestrian](rulesets/pedestrian.yaml) | Ego navigation — sidewalks, crosswalks, indoor wayfinding |
| [delivery](rulesets/delivery.yaml) | Delivery robots — sidewalk bots, last-mile autonomous vehicles |
| [aerial](rulesets/aerial.yaml) | Drone / aerial — mapping, inspection, overflight |

## Rules

A rule in a profile looks like this:

```yaml
- code: quality-blurry
  name: Video is blurry
  metric: blur_score
  type: float
  pass: ">=15"
  flag: "5–15"
  reject: "<5"
```

Compute `blur_score`. Above 15 — no issue emitted. 5 to 15 — flag. Below 5 — reject.

**Types:** `bool`, `float`, `ratio` (0–1), `int` (non-negative).

**Thresholds:** `">=N"`, `"<=N"`, `">N"`, `"<N"`, `"A–B"` (en-dash range), `true`/`false`, exact `"N"`.

## Variants

Variants override profile thresholds for specific deployments without forking the profile.

**Override variant** — tightens or loosens thresholds:

```yaml
name: high-precision
base_profile: manipulation

overrides:
  - code: quality-blurry
    pass: ">=20"
    flag: "10–20"
    reject: "<10"
```

**Add-on variant** — appends new rules to any profile:

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

| Variant | Type | For |
|---|---|---|
| [home-chores](variants/home-chores.yaml) | override | Crowdsourced household recordings (Micro1, 1X style) |
| [home-cleaning](variants/home-cleaning.yaml) | override | Household cleaning — strict privacy for private homes |
| [factory](variants/factory.yaml) | override | Industrial settings, consented workers (Build AI style) |
| [kitchen](variants/kitchen.yaml) | override | Cooking / food prep — relaxed sharp-object, steam-tolerant |
| [high-precision](variants/high-precision.yaml) | override | Sub-mm assembly (USB insertion, snap-fit, soldering) |
| [auto-repair](variants/auto-repair.yaml) | override | Automotive service bays — tools expected, wide exposure range |
| [construction](variants/construction.yaml) | override | Building sites — dust, long sessions, coarse narration |
| [agriculture](variants/agriculture.yaml) | override | Outdoor farming — crop inspection, harvesting, long sessions |
| [yard-work](variants/yard-work.yaml) | override | Lawn care, gardening, landscaping — outdoor home tasks |
| [retail-stocking](variants/retail-stocking.yaml) | override | Store shelf stocking — visible text expected, strict customer privacy |
| [warehouse](variants/warehouse.yaml) | override | Warehouse picking/packing — industrial indoor, consented workers |
| [segmentation](variants/segmentation.yaml) | add-on | Scene boundary + caption quality |

## Codes

Every code carries its category as a prefix, so the taxonomy is in the name.

**Technical quality** — `quality-missing-moov` · `quality-corrupt-header` · `quality-truncated-file` · `quality-no-keyframes` · `quality-gop-too-long` · `quality-duplicate-frames` · `quality-timing-gaps` · `quality-low-bitrate` · `quality-high-bitrate` · `quality-file-too-small` · `quality-ffprobe-failed` · `quality-validator-error` · `quality-spec-mismatch` · `quality-variable-frame-rate` · `quality-irregular-gop` · `quality-container-codec-mismatch` · `quality-metadata-rotation` · `quality-edit-list` · `quality-multiple-video-tracks` · `quality-blurry` · `quality-bad-exposure` · `quality-lens-blocked` · `quality-duration-out-of-bounds` · `quality-invalid-intrinsics` · `quality-missing-imu` · `quality-imu-desync` · `quality-missing-depth`

**Privacy** — `privacy-bathroom-visible` · `privacy-reflective-surface` · `privacy-phone-visible` · `privacy-document-visible` · `privacy-face-visible` · `privacy-person-closeup` · `privacy-visible-text` · `privacy-screen-visible` · `privacy-speech-audio` · `privacy-property-overflight` · `privacy-license-plate-visible`

**Safety** — `safety-sharp-object`

**Training value** — `training-no-person-visible` · `training-no-hands-visible` · `training-static-video` · `training-black-frames` · `training-frozen-video`

**Metadata** — `metadata-mismatch`

**Multi-stream** — `multistream-sync-drift` · `multistream-fps-mismatch` · `multistream-frame-drops` · `multistream-stream-missing` · `multistream-duration-mismatch` · `multistream-calibration-missing`

**Ego-exo** — `egoexo-egomotion-missing` · `egoexo-sparse-narration` · `egoexo-insufficient-view-overlap`

**Task** — `task-maneuverability-untagged`

**Demonstration** — `demo-incomplete-task` · `demo-jerky-motion` · `demo-low-manipulation` · `demo-excessive-idle` · `demo-no-interaction` · `demo-bad-viewpoint`

**Trajectory (RLDS / OXE)** — `trajectory-invalid-episode` · `trajectory-invalid-action-vector` · `trajectory-missing-reward`

**Navigation** — `nav-gps-invalid` · `nav-gps-drift` · `nav-obstacle-not-annotated` · `nav-path-discontinuity` · `nav-surface-untagged`

**Delivery** — `delivery-gps-invalid` · `delivery-gps-drift` · `delivery-route-incomplete` · `delivery-weather-untagged` · `delivery-address-visible`

**Flight** — `flight-altitude-missing` · `flight-altitude-out-of-bounds` · `flight-gps-invalid` · `flight-stability-poor` · `flight-gimbal-drift`

**Segmentation** — `segmentation-scene-too-short` · `segmentation-scene-too-long` · `segmentation-generic-caption` · `segmentation-false-boundary` · `segmentation-missing-boundary` · `segmentation-low-coverage`

## Structure

```
rulesets/              7 profiles with default thresholds
variants/              12 deployment-specific overrides and add-ons
examples/              4 example results (pass, reject, multistream, webdataset)
result-schema.yaml     JSON Schema for result validation
DOCS.md                Full spec documentation
```

## License

Apache 2.0
