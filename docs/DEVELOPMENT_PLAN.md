# AR VR Mobile Project Development Plan

## 1. Project Goal

Build an end-to-end prototype in which an 11-inch second-generation iPad Pro
scans one indoor room using ARKit and LiDAR, runs vision models locally, and
creates a metrically scaled, photorealistically textured 3D reconstruction on
the device. A Meta Quest application receives that reconstruction directly over
a local network and lets a user explore it in VR.

This is the first technical milestone for the larger asymmetric cross-device
escape-room project. Gameplay, live collaboration, AI generation, and remote
guidance are intentionally deferred until reconstruction is reliable. A cloud
service or Windows reconstruction worker must not be required for the final
demonstration.

## 2. Definition of Photorealistic Reconstruction

For this project, photorealistic reconstruction means that real camera colors
are projected onto the LiDAR-derived room mesh and baked into one or more UV
texture atlases. Walls, floors, furniture, and visible materials should be
recognizable from ordinary viewing distances in VR.

It does not mean cinematic digital-twin quality. The first version may contain
holes behind furniture, blurred regions, lighting differences, and simplified
geometry. A vertex-colored mesh is retained as a fallback and debugging output,
but it does not by itself satisfy the final textured-reconstruction milestone.

## 3. On-Device Intelligence Requirement

Running models locally on the iPad is a required research contribution, not an
optional optimization. The project will define model roles and interfaces now,
then select concrete models after profiling the A12Z device.

The first milestone contains two local inference stages:

1. A scan-time vision stage, executed at a controlled frequency, that improves
   capture decisions. Its eventual task may be semantic/dynamic-region masking,
   keyframe quality, coverage estimation, or another measured scan-time need.
2. A post-scan enhancement stage that improves reconstruction quality. Its
   eventual task may be depth completion/refinement, geometry confidence,
   texture selection, denoising, super-resolution, or inpainting.

The exact models are intentionally undecided. Each stage is accessed through a
versioned interface so candidates can be replaced without rewriting scanning or
export. At least one scan-time model and one post-scan model must run entirely
on the iPad in the final prototype and must produce measurable output used by
the reconstruction pipeline.

Core requirements are:

- Inference works with internet access disabled.
- RGB, depth, and room data never need to leave the local devices.
- Model files, preprocessing, outputs, and versions are recorded.
- Inference is asynchronous and cannot stall ARKit tracking.
- Failure or unsupported operations fall back to an ARKit-only reconstruction.
- Evaluation compares ARKit-only and local-model-enhanced outputs.
- Latency, memory, energy/thermal behavior, and quality improvement are measured.

### Candidate model roles

These are planned capability slots, not selected models. The research spike will
retain one scan-time role and one post-scan role based on device evidence.

| Role | Inputs | Outputs used by the app | Execution window | Non-model fallback |
|---|---|---|---|---|
| Capture quality/coverage | RGB, depth confidence, pose history | frame score or missing-view map | Throttled during scanning | motion, blur, and tracking heuristics |
| Semantic/dynamic masking | RGB and optional depth | per-pixel labels/confidence | During scan or finalization | ARKit classification and no mask |
| Depth refinement | RGB, LiDAR depth, confidence | enhanced depth and uncertainty | Post-scan, tiled | raw or smoothed ARKit depth |
| Texture enhancement | projected atlas and coverage mask | denoised, upscaled, or filled texture plus confidence | Post-scan, tiled | camera projection and neutral hole fill |

Pretrained and converted models are acceptable; training a new model is not a
first-milestone requirement. Model choice must remain separate from capture,
geometry, texturing, and export code.

## 4. Scope

### Included

- One indoor room per capture.
- ARKit LiDAR scene mesh acquisition through Unity AR Foundation.
- Synchronized RGB, scene depth, confidence, camera intrinsics, and poses.
- A model-agnostic on-device inference runtime.
- Scan-time and post-scan local vision-model stages.
- Scan-origin and floor calibration.
- On-device geometry cleanup, UV generation, projection, blending, and export.
- Direct local-network transfer from iPad to Quest.
- Quest rendering, collision, teleportation, and joystick locomotion.
- Baseline-versus-model evaluation of quality and device performance.

### Deferred

- Live mesh streaming while scanning.
- Multi-room capture and persistent buildings.
- Shared avatars and simultaneous AR/VR editing.
- Escape-room objects, puzzles, voice prompts, and AI generation.
- Gaussian splats or NeRF rendering on Quest.
- Public cloud hosting and user accounts.
- Cloud inference or a PC required during normal use.
- Selecting or training a final model before device profiling.

## 5. System Architecture

```text
iPad Scanner
  ARKit mesh chunks
  RGB + depth + confidence keyframes
  camera calibration + room origin
          |
          v
iPad Local Intelligence and Reconstruction
  scan-time model -> capture guidance/masks/confidence
  post-scan model -> depth/geometry/texture enhancement
  clean mesh -> unwrap UVs -> bake and blend textures
  export textured GLB + processing report
          |
          | direct local-network RoomPackage transfer
          v
Meta Quest Viewer
  async GLB import -> materials -> collision
  safe spawn -> teleport/joystick exploration
```

The Windows laptop is a development, test, and optional reference-processing
host. It is not part of the deployed workflow. The Mac is needed periodically
for Xcode signing, compilation, profiling, and deployment of the iPad app.
After both apps are installed, the core demonstration should require only the
iPad, Quest, and a local Wi-Fi network.

## 6. Technology Baseline

### Unity application

- Unity 6 LTS, pinned to one tested patch version.
- Universal Render Pipeline.
- AR Foundation and matching Apple ARKit XR Plugin versions.
- OpenXR, Unity OpenXR: Meta, and XR Interaction Toolkit.
- glTFast for runtime GLB export/import where its tested version is suitable.
- Unity Test Framework for shared coordinate and serialization tests.

### On-device model runtime

- A Unity-facing `ILocalVisionModel` contract independent of any model.
- A runtime-adapter spike comparing a native Core ML bridge and suitable Unity
  inference support; selection occurs after testing on the 2020 iPad Pro.
- Explicit preprocessing and output schemas for RGB, depth, confidence, masks,
  quality maps, and enhanced depth or color.
- A bounded inference queue, cancellation, progress, and thermal/memory metrics.
- Model manifests containing identifier, version, input/output shapes, precision,
  expected compute units, and checksum.

Core ML is the primary platform capability to evaluate because it can schedule
work across the Apple CPU, GPU, and Neural Engine. The plan does not assume that
every candidate operation is supported or fast enough on the A12Z.

### On-device reconstruction

- C# Jobs/Burst or a native iOS plugin for geometry processing.
- Metal compute where profiling shows a benefit.
- xatlas or an equivalent iOS-compatible library for UV generation.
- Visibility-aware camera projection and texture-atlas baking on the iPad.
- GLB output with base-color textures and meter-based scale.

An optional desktop reference implementation may use OpenCV or OpenMVS to
measure the quality/performance gap. It must not become a hidden requirement for
the iPad-to-Quest workflow.

### Direct transfer

- An embedded HTTP server or equivalent cross-platform LAN protocol on iPad.
- Bonjour/mDNS discovery where practical, with manual pairing fallback.
- Quest downloads a completed, checksummed RoomPackage directly from the iPad.
- The transfer works without internet access.

Large geometry and images should not be sent through a real-time multiplayer
service such as Photon. A future multiplayer layer should synchronize small
gameplay state while the room asset remains an HTTP download.

## 7. Data Contracts

### RoomCapture input bundle

```text
capture_<room-id>.zip
  manifest.json
  geometry/
    mesh_000.bin
    mesh_001.bin
  frames/
    frame_0001.jpg
    frame_0001.depth
    frame_0001.confidence
    frame_0001.json
  inference/
    model-manifest.json
    frame_0001.mask
  preview.jpg
```

Each mesh record contains positions, normals, indices, a stable mesh identifier,
and its transform relative to the selected room origin. Each frame record
contains the image path, timestamp, image resolution, focal length, principal
point, camera-to-room transform, orientation, tracking state, depth calibration,
and references to local-model outputs.

The manifest contains a schema version, units, coordinate convention, device
information, room bounds, scan origin, intended Quest spawn pose, and checksums.

### RoomPackage output bundle

```text
room_<room-id>.zip
  manifest.json
  room.glb
  preview.jpg
  processing-report.json
```

The processing report records vertex and triangle counts, image count, texture
coverage, atlas resolution, processing duration, warnings, model versions,
inference timings, peak memory, and thermal-state observations.

## 8. Component Breakdown

### A. iPad scanning shell

Create an AR session with explicit states: Ready, Scanning, Paused, Finalizing,
Processing, ReadyToShare, Complete, and Failed. Provide tracking-quality feedback, reset,
scan duration, and a visible mesh overlay. Keep the controls usable while the
user walks and holds the iPad with two hands.

Exit condition: the app can repeatedly start, stop, and reset a room scan on the
physical iPad without leaking camera images or retaining an old AR session.

### B. LiDAR geometry capture

Collect `ARMeshManager` mesh chunks and freeze a consistent snapshot when the
scan ends. Transform every chunk into room-origin coordinates. Remove invalid
triangles and very small isolated components while preserving major furniture.
Record floor height and room bounds. Do not merge everything into one collider;
retain spatial chunks for loading and collision performance.

Exit condition: a desktop debug viewer can load the geometry and measured wall
distances remain within the agreed tolerance.

### C. Synchronized RGB-D keyframe capture

Acquire CPU camera images asynchronously instead of on every frame. Save a new
keyframe only after meaningful motion, for example roughly 0.2-0.3 metres or
10-15 degrees, and only when tracking and image sharpness are acceptable.
Record intrinsics, scene depth, depth confidence, pose, and orientation for the
same frame. Downsample and compress only after a quality spike establishes
suitable settings.

Exit condition: every saved image can be reprojected from its recorded camera
pose onto a known test plane with aligned color and depth.

### D. Model-agnostic local inference layer

Define common input/output tensors and metadata without choosing a model. Add
adapters for candidate runtimes, asynchronous scheduling, cancellation, model
loading, version checks, resource measurements, and graceful fallback. Keep
Unity scene code unaware of Core ML or another runtime's concrete API.

Exit condition: a trivial test model runs on the physical iPad, returns a known
result, records its execution metrics, and can be disabled without changing the
capture pipeline.

### E. Scan-time local intelligence

Integrate one throttled local vision stage into scanning. Candidate roles are
evaluated later, but its output must alter a real capture decision such as frame
acceptance, dynamic-region masking, or coverage guidance. The scheduler should
skip work rather than build a queue when the device is busy.

Exit condition: inference runs during a five-minute scan without loss of AR
tracking and its contribution is visible in logs and captured data.

### F. Post-scan local enhancement

Run a second local model after scanning, when latency is less strict. Candidate
roles include depth completion, confidence refinement, denoising, texture
enhancement, super-resolution, and inpainting. Process tiles or chunks to keep
memory bounded and show progress/cancellation.

Exit condition: the enhanced output is consumed by geometry or texturing and an
ARKit-only result can be generated from the same capture for comparison.

### G. Capture finalization and validation

Stop mesh updates, finish outstanding image conversions, verify indices and
matrices, calculate checksums, and write the versioned RoomCapture bundle.
Reject incomplete captures before processing and provide actionable messages such as
missing floor origin, insufficient keyframes, or invalid tracking data.

Exit condition: both an on-device validator and desktop test tool accept valid
bundles and identify deliberately damaged bundles without crashing.

### H. On-device geometry processing

Normalize coordinate handedness once at the bundle boundary. Weld near-duplicate
vertices where safe, remove degenerates and tiny disconnected islands, repair
normals, and create a lower-detail collision representation. Decimation is
optional until profiling demonstrates a need; dimensional accuracy is more
important than a small early file.

Exit condition: processing completes on the iPad without OS termination and
does not change room scale or orientation.

### I. On-device photorealistic texturing

Run an early device spike to determine which UV, projection, and blending stages
should use C#, Burst, Metal, or a native library. The selected on-device pipeline
must perform:

1. UV chart generation and atlas packing.
2. Triangle visibility tests against each camera.
3. Candidate-image scoring using resolution, distance, view angle, blur, and
   exposure.
4. Per-face or per-texel image selection.
5. RGB projection into the atlas with occlusion checks.
6. Multi-image blending, exposure compensation, padding, and seam reduction.
7. A coverage mask and a neutral fallback for unobserved texels.
8. Export of a textured GLB plus processing metrics.

Begin with a 2048-pixel atlas and support 4096 after Quest profiling. Keep the
untextured mesh and vertex-color preview as diagnostic outputs.

Exit condition: the iPad produces the textured GLB itself; at least 85 percent of
the visible test-room surface is textured, major materials are recognizable,
and severe projection errors are absent.

### J. Direct iPad-to-Quest transfer

Expose completed room packages to the Quest over the local network. Implement
discovery or pairing, checksums, resumable/retry behavior, safe paths, and clear
connection states. No internet or laptop may be required.

Exit condition: interrupted downloads can be retried without corrupting the
iPad's completed room or Quest cache.

### K. Quest viewer

Provide room-code entry or discovery, download progress, asynchronous GLB load,
texture/material setup, safe spawn placement, collision, teleportation, snap
turning, and optional smooth locomotion. Exploration must use virtual locomotion;
the scanned room is not assumed to match the user's current physical boundary.

Exit condition: a user can load and explore three different processed rooms
without restarting the Quest application.

### L. Evaluation and reproducibility

Create a repeatable test protocol using measured reference distances and fixed
visual targets. Record capture time, processing time, scale error, texture
coverage, file size, load time, frame rate, memory, inference latency, thermal
state, and user-rated visual quality. Every quality test compares the ARKit-only
baseline with the local-model-enhanced result from the same capture.
Store only consented, non-sensitive captures and document deletion procedures.

Exit condition: another team member can reproduce the demonstration and its
measurements from the repository documentation.

## 9. Fourteen-Week Schedule

| Weeks | Milestone | Main deliverable |
|---|---|---|
| 1-2 | M0 Foundation | Repository, Unity profiles, schemas, CI, iPad and Quest smoke builds |
| 3-4 | M1 Multimodal capture | LiDAR mesh plus aligned RGB, depth, confidence, origin, and poses |
| 5 | M2 Local runtime | Model-neutral interfaces and test-model inference on the A12Z iPad |
| 6 | M3 Scan intelligence | Throttled local inference influences capture and records metrics |
| 7 | M4 Post-scan intelligence | Local enhancement output used by reconstruction with baseline fallback |
| 8 | M5 On-device geometry | Cleanup, coordinates, validation, and collision mesh on iPad |
| 9-10 | M6 On-device texturing | UV, projection, blending, enhancement, and textured GLB on iPad |
| 11 | M7 Direct transfer | iPad hosting/discovery, Quest download, retry and integrity handling |
| 12 | M8 Quest integration | Textured room loading, collision, locomotion, performance pass |
| 13 | M9 System validation | Baseline/model ablation, three rooms, offline end-to-end testing |
| 14 | M10 Demonstration | Stable builds, video, setup guide, limitations, and progress report |

Mac/iPad integration should occur at least once every week from Week 2 onward.
Work that does not require Apple hardware should use recorded bundles and the
desktop debug viewer on Windows.

## 10. Acceptance Criteria

- Scan one ordinary office or classroom in no more than five minutes.
- Produce a Quest-loadable result without manually copying model files.
- Run at least one scan-time and one post-scan model entirely on the iPad.
- Complete capture, inference, reconstruction, and GLB export with internet off.
- Demonstrate a measurable quality or capture-reliability improvement over the
  ARKit-only baseline for each retained local-model stage.
- Keep dimensional error below 5 percent over a measured two-metre distance,
  with 10 percent recorded as the maximum fallback threshold.
- Texture at least 85 percent of surfaces visible in the reference walkthrough.
- Keep the processed room package below 75 MB for the reference room.
- Complete post-scan processing on the iPad within an initial ten-minute budget;
  report per-stage time, peak memory, battery use, and thermal state.
- Load the room on Quest in less than 60 seconds over the test network.
- Maintain 72 frames per second during normal exploration of the reference room.
- Prevent invalid spawn locations and floor fall-through.
- Complete five consecutive end-to-end runs and reconstruct three distinct rooms.

## 11. Risk Controls

| Risk | Control |
|---|---|
| Blurry or sparse imagery | Motion-based keyframes, blur rejection, coverage guidance |
| Texture ghosting | Pose validation, visibility tests, view-angle scoring, seam blending |
| Lighting changes | Lock or normalize exposure where possible; compensate during baking |
| Mesh holes and noise | Scan guidance, cleanup, neutral texture fallback, explicit warnings |
| Heavy iPad CPU use | Throttled asynchronous image conversion and bounded capture queue |
| Model stalls AR tracking | Low-rate inference, bounded queue, cancellation, adaptive scheduling |
| Model unsupported on A12Z | Runtime capability test, smaller inputs/precision, alternate candidate, baseline fallback |
| iPad memory/thermal termination | Tiled post-processing, staged file I/O, profiling, reduced atlas/model settings |
| Large bundles | Downsample keyframes, JPEG compression, checksums, configurable limits |
| Quest memory/performance | Chunked geometry, atlas limits, async loading, LOD/collision mesh |
| Coordinate mismatch | One documented boundary conversion with automated fixture tests |
| Mac availability | Weekly device sessions and captured test bundles for Windows work |
| Network client isolation | Private router, hotspot, manual pairing, cached Quest packages |
| Third-party licensing | License review issue before adopting OpenMVS or other dependencies |

## 12. Working Method

- Every change begins with a GitHub issue and explicit acceptance criteria.
- Research uncertainty is tracked as a time-boxed `type:spike` issue.
- Pull requests should be small, reference their issue, and include device-test
  evidence when they change iPad or Quest behavior.
- `main` should remain demonstrable; incomplete work uses feature flags or
  separate scenes.
- Package versions and model-format versions are pinned after the first working
  end-to-end build.
- A model is adopted only after an iPad benchmark issue records quality, latency,
  peak memory, package size, compute-unit behavior, and thermal impact.

