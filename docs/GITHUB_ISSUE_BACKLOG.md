# GitHub Issue Backlog

This backlog is ordered by dependency, not just by feature area. Create the
milestones and labels first, then open the issues under the suggested milestone.

## Labels

`area:shared`, `area:ios`, `area:geometry`, `area:capture`, `area:on-device-ml`,
`area:texturing`, `area:transfer`, `area:quest`, `area:testing`, `area:docs`, `type:feature`,
`type:bug`, `type:spike`, `type:chore`, `priority:p0`, `priority:p1`,
`priority:p2`, `blocked`, and `needs-device-test`.

## Milestones

- M0 Foundation
- M1 LiDAR Geometry
- M2 Multimodal Capture and Local Models
- M3 On-Device Reconstruction
- M4 Quest Exploration
- M5 Evaluation and Demonstration

## M0 Foundation

### RB-001 Define repository conventions

Create the directory layout, Unity `.gitignore`, Git LFS policy, branch naming,
pull-request checks, and contribution notes. Ensure generated Unity directories,
builds, captures, credentials, and processed rooms are ignored.

Acceptance: a clean clone contains no generated or private capture data.

### RB-002 Create Unity project and platform build profiles

Configure Unity 6 LTS, URP, iPadOS and Android build profiles, assembly
definitions, and separate scanner/viewer scenes.

Acceptance: both profiles compile with shared code but platform-specific code is
isolated behind assemblies rather than scattered preprocessor directives.

### RB-003 Pin XR and model packages

Install compatible AR Foundation, ARKit XR Plugin, OpenXR, Unity OpenXR: Meta,
XR Interaction Toolkit, and glTFast versions. Record the tested versions.

Acceptance: package restore is reproducible on a second machine.

### RB-004 Build iPad smoke test

Generate the Xcode project, configure signing and camera/local-network privacy
strings, and deploy an empty AR scene to the physical iPad.

Acceptance: AR tracking starts and a device-test screenshot/log is attached.

### RB-005 Build Quest smoke test

Configure Android SDK/NDK, OpenXR and Meta project validation, then deploy a
minimal XR scene.

Acceptance: head and controller tracking work on the target Quest.

### RB-006 Add shared tests and continuous integration

Run edit-mode tests for contracts, matrix conversion, checksums, and validators
on every pull request. Device builds may remain manual initially.

Acceptance: a deliberately failing test blocks the pull request check.

## M1 LiDAR Geometry

### RB-010 Implement scanner state machine

Implement Ready, Scanning, Paused, Finalizing, Processing, ReadyToShare,
Complete, and Failed states with valid transitions and reset behavior. Depends
on RB-002 and RB-004.

### RB-011 Display live LiDAR mesh

Configure `ARMeshManager`, a mesh prefab, visualization material, and optional
colliders for debugging. Depends on RB-003 and RB-010.

### RB-012 Add tracking and scan-quality guidance

Display tracking state, elapsed time, mesh count, approximate bounds, and clear
warnings when tracking is poor. Depends on RB-010 and RB-011.

### RB-013 Select room origin, floor, forward direction, and VR spawn

Let the scanner confirm a floor point and forward direction. Persist the pose in
room coordinates. Depends on RB-011.

### RB-014 Freeze a consistent mesh snapshot

Stop accepting mesh changes, wait for pending mesh jobs, copy vertex/index data,
and record every chunk transform. Depends on RB-011 and RB-013.

### RB-015 Implement coordinate conversion fixtures

Create known cube, floor, and camera fixtures covering Unity, ARKit, the
on-device processing boundary, and glTF handedness. Depends on RB-006.

### RB-016 Clean and measure captured geometry

Remove degenerate triangles and tiny islands, repair normals, calculate bounds,
and report triangle counts without changing scale. Depends on RB-014 and RB-015.

### RB-017 Build desktop geometry viewer

Load a saved geometry fixture on Windows and display axes, origin, bounds, and
measuring tools. Depends on RB-015 and RB-016.

## M2 Multimodal Capture and Local Models

### RB-020 Capture one synchronized RGB-D frame

Acquire and dispose camera/depth resources correctly, then save aligned RGB,
scene depth, depth confidence, intrinsics, timestamp, resolution, orientation,
and camera-to-room transform. Depends on RB-004 and RB-015.

### RB-021 Verify camera projection math

Reproject known 3D points into saved color and depth images and test alignment,
rotation, and cropping on the physical iPad. Depends on RB-020.

### RB-022 Implement keyframe selection

Capture after configurable translation/rotation thresholds and avoid duplicate
views. Bound the conversion queue and storage use. Depends on RB-020.

### RB-023 Define local-model contracts

Define `ILocalVisionModel`, model manifest, typed input/output tensors,
preprocessing, cancellation, result confidence, metrics, and failure behavior.
Do not select a concrete model in this issue. Depends on RB-006 and RB-020.

### RB-024 Spike: select the iPad inference runtime strategy

Run a trivial known-output model through candidate runtime adapters on the A12Z
iPad. Compare native Core ML integration and suitable Unity inference support
for compatibility, latency, memory, package size, and implementation cost.

Acceptance: choose the runtime integration approach, not the final models, and
record an architecture decision. Depends on RB-003, RB-004, and RB-023.

### RB-025 Implement on-device model manager

Load a model by manifest, verify its checksum and compatibility, select allowed
compute units, warm it up, run asynchronously, record metrics, and unload it.
Depends on RB-024.

### RB-026 Implement bounded inference scheduler

Provide separate scan-time and post-scan queues, cancellation, back-pressure,
progress, thermal-state hooks, and fallback. Scan-time work must be skipped when
busy rather than accumulating stale frames. Depends on RB-025.

### RB-027 Spike: select local model roles and candidate families

Using captured fixtures, evaluate which scan-time and post-scan tasks offer the
best research value on the A12Z. Candidate tasks include semantic/dynamic masks,
quality/coverage estimation, depth refinement, denoising, super-resolution, and
inpainting. Do not combine model selection with runtime selection.

Acceptance: shortlist candidates, define metrics and thresholds, and create
separate implementation issues. Depends on RB-021, RB-025, and RB-026.

### RB-028 Integrate scan-time local inference

Run the selected lightweight stage at a controlled frequency and use its output
for a real capture decision such as keyframe acceptance, region masking, or
coverage guidance. Preserve an ARKit-only toggle. Depends on RB-022, RB-026,
and RB-027.

### RB-029 Integrate post-scan local enhancement

Run the selected enhancement after capture in bounded tiles/chunks. Persist its
output and consume it in geometry or texturing. Preserve cancellation and an
ARKit-only path. Depends on RB-026 and RB-027.

### RB-030 Define RoomCapture schema version 1

Document mesh, RGB-D frame, model manifest/output, coordinate, checksum, and
compatibility rules. Add valid and invalid fixtures. Depends on RB-015, RB-021,
and RB-023.

### RB-031 Serialize and validate RoomCapture on iPad

Finalize outstanding capture and inference jobs, validate references, and write
the bundle atomically with recovery from low storage or cancellation. Depends on
RB-014, RB-028 through RB-030.

### RB-032 Add cross-platform fixture validator

Validate schemas, paths, checksums, matrices, image references, mesh indices,
depth alignment, model outputs, and resource limits. Depends on RB-030.

## M3 On-Device Reconstruction

### RB-040 Establish ARKit-only reconstruction baseline

Produce geometry and a basic texture result without custom model output. This is
the fallback and experimental control for every later comparison. Depends on
RB-016, RB-021, and RB-031.

### RB-041 Implement local-model output fusion

Apply scan-time masks/confidence and post-scan enhancement outputs to geometry or
texture processing. Record exactly which output affects each stage. Depends on
RB-028, RB-029, and RB-040.

### RB-042 Process geometry on the iPad

Normalize coordinates, remove invalid topology and small islands, preserve
scale, repair normals, and create render/collision variants using bounded memory.
Depends on RB-016 and RB-041.

### RB-043 Spike: choose on-device UV and baking implementation

Benchmark C#/Burst, Metal, and native-library boundaries using the physical iPad.
Select components based on correctness, memory, thermal behavior, implementation
effort, and licenses. Depends on RB-040 and RB-042.

### RB-044 Generate UV atlas on the iPad

Generate non-overlapping UV charts with padding and configurable atlas size.
Emit utilization and distortion metrics. Depends on RB-043.

### RB-045 Implement on-device camera visibility tests

Project triangles into keyframes and reject back-facing, out-of-frame, or
occluded samples using captured depth or a z-buffer. Depends on RB-021 and RB-042.

### RB-046 Score and select source images

Score candidates by projected resolution, angle, distance, sharpness, tracking,
and exposure. Consume local-model confidence/masks when enabled and produce a
debug view showing the selected image per triangle. Depends on RB-041 and RB-045.

### RB-047 Bake base-color atlas on the iPad

Rasterize selected camera pixels into UV space, fill valid texels, and emit a
coverage mask. Depends on RB-044 and RB-046.

### RB-048 Add exposure balancing and seam reduction

Balance overlapping images, blend chart boundaries, pad atlas islands, and avoid
bleeding across unrelated charts. Depends on RB-047.

### RB-049 Apply local texture enhancement and fallback

Apply the post-scan model only where its confidence and benchmark justify it.
Mark enhanced/inpainted texels and fill remaining unobserved regions neutrally.
Depends on RB-029 and RB-047.

### RB-050 Export and validate textured GLB on the iPad

Export geometry, UVs, base-color textures, materials, bounds, and spawn metadata.
Reload the GLB and verify scale, texture references, normals, and checksums.
Write model/inference metrics into the report. Depends on RB-048 and RB-049.

### RB-051 Implement vertex-color diagnostic fallback

Project colors onto vertices to provide a fast preview when atlas generation
fails. Mark this output as diagnostic, not final photorealistic reconstruction.
Depends on RB-045.

## M4 Quest Exploration

### RB-060 Implement the iPad room-package server

Serve only completed, checksummed RoomPackages through an embedded local endpoint.
Add lifecycle control, safe paths, request limits, and clear connection state.
Depends on RB-050.

### RB-061 Implement local discovery and pairing

Evaluate Bonjour/mDNS discovery and provide a manual IP or short-code fallback.
The workflow must function with internet access disabled. Depends on RB-060.

### RB-062 Add Quest room selection and download

Discover or pair with the iPad, list completed rooms, show download status,
verify checksums, cache the result, and allow retry/deletion. Depends on RB-005
and RB-061.

### RB-063 Load GLB asynchronously on Quest

Instantiate the model without a long frame stall, apply supported materials,
and show meaningful errors for incompatible packages. Depends on RB-050 and RB-062.

### RB-064 Generate Quest collision representation

Use the iPad-produced collision mesh or cook chunked colliders incrementally.
Verify floors, walls, stairs, and missing geometry behavior. Depends on RB-042
and RB-063.

### RB-065 Implement safe spawn and locomotion

Place the rig at the recorded spawn, validate head clearance, add teleportation,
snap turn, and optional smooth locomotion. Do not assume the virtual room matches
the physical Guardian boundary. Depends on RB-013 and RB-064.

### RB-066 Profile and optimize Quest rendering

Measure CPU/GPU frame time, draw calls, memory, texture size, shader variants,
load time, and thermal behavior. Tune atlas and mesh settings while preserving
visual quality. Depends on RB-063 through RB-065.

## M5 Evaluation and Demonstration

### RB-070 Build reconstruction and local-model benchmark kit

Define three test rooms, measured distances, color/texture targets, scan route,
lighting conditions, ARKit-only controls, and a repeatable results sheet.

### RB-071 Benchmark local models on the A12Z iPad

For each retained model, record quality effect, inference latency, peak memory,
model/package size, compute-unit configuration, battery change, thermal state,
and scan-time tracking impact. Depends on RB-027 through RB-029 and RB-070.

### RB-072 Run geometry accuracy study

Measure scale, wall/floor alignment, completeness, repeatability, and failure
conditions across at least five runs. Compare baseline and model-enhanced output.
Depends on RB-042 and RB-070.

### RB-073 Run texture quality study

Measure surface coverage, projection errors, blur, seams, and processing time;
include baseline/model comparisons, consistent screenshots, and ratings.
Depends on RB-050 and RB-070.

### RB-074 Run Quest performance study

Measure package size, download/load time, memory, frame rate, and locomotion
failures for all test rooms. Depends on RB-066 and RB-070.

### RB-075 Verify private offline operation

Disable internet access and verify that capture, both inference stages,
reconstruction, export, direct transfer, and Quest exploration all complete.
Inspect network logs for unintended external requests. Depends on RB-060 through
RB-074.

### RB-076 Complete five end-to-end reliability runs

Scan, process, transfer, load, and explore without manual file intervention.
Record every failure and repeat after fixes. Depends on RB-071 through RB-075.

### RB-077 Produce final prototype documentation

Write setup, Mac/iPad deployment, Quest deployment, network setup, demo script,
architecture, local-model manifests, data/privacy notes, ablation results, and
known limitations. Depends on RB-076.

## Critical Path

```text
RB-002 -> RB-004 -> RB-011 -> RB-014 -> RB-016
                 -> RB-020 -> RB-021 -> RB-023 -> RB-024 -> RB-025
RB-025 -> RB-027 -> RB-028 + RB-029 -> RB-041
RB-016 + RB-041 -> RB-042 -> RB-044 -> RB-045 -> RB-047 -> RB-050
RB-050 -> RB-060 -> RB-062 -> RB-063 -> RB-064 -> RB-066 -> RB-076
```

The ARKit-only baseline reaches Quest before local-model enhancement is complete.
It remains a fallback and experimental control, while the final demonstration
must execute both required local inference stages on the iPad.

