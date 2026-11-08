---
name: uav-manage-pipelines
description: >-
  Start, stop, inspect, and modify DL Streamer Pipeline Server pipelines in
  an already-deployed uav-vision-analytics stack — via the arm/disarm
  pipeline manager, the REST API directly, or by editing an element's
  properties / parameter schema / adding a pipeline in configs/config-*.json.
  USE FOR: "start/stop a pipeline by hand", "what pipelines are running",
  "change the detection threshold", "a pipeline fails to start / 400 from
  the REST API", "add a new parameter to a pipeline", "the captioning branch
  needs a different frame rate". DO NOT USE FOR: declaring which camera uses
  which pipeline (that's a configs/cameras-*.yaml edit — use
  `uav-manage-cameras`), swapping a model file (use `uav-change-detection-
  model` / `uav-change-vlm-model`), or scaffolding a brand-new stack
  (use `uav-vision-analytics`).
license: Apache-2.0
compatibility: >-
  Requires an already-deployed uav-vision-analytics stack checked out at a
  known path, reachable via its nginx reverse proxy (self-signed TLS,
  `curl -k`). Editing configs/config-*.json requires recreating the
  dlstreamer-pipeline-server container to pick up the change (its pipeline
  root is a tmpfs volume reset on recreation).
---

<!-- SPDX-FileCopyrightText: (C) 2026 Intel Corporation -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# Manage UAV Pipelines Skill

Two layers: the **pipeline manager** (`make start-rtsp`) drives every
pipeline automatically from `configs/cameras-*.yaml` as the vehicle
arms/disarms, and the **REST API** starts/stops/inspects a pipeline directly
(scripting, or a one-off run with custom parameters). The pipeline
*definitions* themselves (`configs/config-*.json`) are a fixed matrix of
GStreamer launch strings plus a parameter schema — this skill also covers
editing that file when an element's behaviour (not which camera uses it)
needs to change.

## Canonical Documentation (read before acting — do not rely on memorized snippets)

| Task | Read this file in the target repo first |
| --- | --- |
| Start/stop/check a pipeline (manager or REST API), list available pipelines | `docs/user-guide/how-to-customize/pipeline-management/manage-pipelines.md` |
| Change an element's property, add/change a parameter, pipeline naming, add a pipeline | `docs/user-guide/how-to-customize/pipeline-management/modify-pipelines.md` |
| Which pipeline a camera selects, camera file schema | `docs/user-guide/how-to-customize/camera-management/configure-cameras.md` |
| Inspect what a running pipeline detected | `docs/user-guide/how-to-guides/view-detection-logs.md` |

Read the live copy in the target repo for exact REST paths, `curl` syntax,
and the pipeline JSON shape — these are generated from the actual shipped
`config-*.json` and can differ by app version; do not hardcode a remembered
payload.

## How to Use This Skill

1. Confirm `{{STACK_DIR}}`, deployment mode, and `HOST_IP` (for REST/RTSP URLs).
2. Ask, in one batched question, what's needed:
   - Operate: start/stop/check a pipeline by hand, or via the manager —
     which camera(s)/pipeline name, with which parameters if starting.
   - Modify: which element/property, or which new parameter, in which
     pipeline name(s) — and whether this should apply to one device variant
     or the whole matrix (cpu/gpu/npu × caption cpu/gpu).
3. Read `manage-pipelines.md` and/or `modify-pipelines.md` as applicable for
   the exact commands/JSON shape.
4. Act (REST call, or edit `configs/config-*.json`), then validate, apply,
   and verify per the guardrails/completion criteria below.

## Parameter Validation (enforce BEFORE acting)

| Context | Rule | Why |
| --- | --- | --- |
| Starting a pipeline | pipeline `name` exists in the mounted `config.json` (list first if unsure) | a typo'd name 404s |
| Stopping/checking a pipeline | use the integer `instance_id` from the POST response, never the pipeline `name` | `DELETE /pipelines/{instance_id}`, not `/pipelines/{name}/{id}` |
| Editing an element property | keep the element's device in step with the pipeline's name (`uav_object_detection_gpu` keeps `device=GPU` throughout) | a mismatched device silently fails or runs on the wrong hardware |
| Adding a parameter | give it a `type`, a `default`, and a `pattern`/`minimum`/`maximum` restricting caller input | an unconstrained parameter lets a start request inject arbitrary pipeline behaviour |
| Captioning-branch edits | device stays `cpu`/`gpu` only, frame-rate throttle (`videorate ... framerate=1/5`) stays unless the user explicitly wants a different cadence | `gvagenai` has no NPU plugin; removing the throttle overloads the VLM |

## Execution Guardrails

- The REST API is reached only through nginx at `https://<HOST_IP>/` with a
  self-signed certificate (`curl -k`) — `dlstreamer-pipeline-server`'s own
  port is not published to the host; never suggest a direct `:8081` URL.
- Never add, remove, or rename a pipeline just to support a new camera
  combination that's actually a `configs/cameras-*.yaml` concern — check
  with `uav-manage-cameras`'s scope first; only touch `config-*.json` when
  an element's *behaviour* must change for everyone using that pipeline.
- Anything in the GStreamer string that is not a `{parameters[x]}`
  placeholder is fixed and can only be changed by editing the file — do not
  try to override it from a start request.
- Validate a hand-edited GStreamer string with `gst-launch-1.0` (placeholders
  replaced with real values) before saving it — a bad string only fails at
  pipeline *start*, not at config load, so a mistake is otherwise invisible
  until someone arms the vehicle.
- After any `configs/config-*.json` edit, recreate the container
  (`make pymav-down && make pymav-up`, or the `uavsdk` equivalent) — its
  pipeline root is a tmpfs volume, and `restart` alone keeps the stale
  config; then `make start-rtsp` to restart the pipeline manager.
- Stopping a pipeline through the manager means disarming the vehicle; to
  stop everything the manager started while it keeps waiting for the
  vehicle, use the REST stop-all endpoint, not `docker stop` on the
  container (which also kills the manager).
- Report exact REST calls and file edits made; never run
  `make pymav-down`/`make uavsdk-down` without the user's confirmation, since
  it drops every currently-streaming camera.

## Completion Criteria

1. For a manual start: the REST call returns an `instance_id`, and
   `GET /pipelines/{instance_id}/status` reports `RUNNING`.
2. For a manual stop: `DELETE /pipelines/{instance_id}` succeeds and the
   instance no longer appears in a subsequent status list.
3. For an element/parameter edit: the container was recreated (not just
   restarted), `make start-rtsp` reconnects, and `make cameras-status` shows
   the affected camera(s) running the updated pipeline.
4. For a new parameter: a start request setting it is accepted (no `400`),
   and the resulting pipeline behaves as expected (verified via RTSP output,
   MQTT metadata, or the detection log as appropriate).
5. No pipeline name/path regression for cameras not in scope of the change.
