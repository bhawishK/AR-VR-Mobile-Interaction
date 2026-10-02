# AR VR Mobile Project

AR VR Mobile Project is a cross-device spatial reconstruction prototype. An iPad Pro uses
ARKit, LiDAR, and locally running vision models to understand and reconstruct a
physical room. The iPad processes the capture into a metrically scaled,
photorealistically textured 3D model. A Meta Quest application receives the
model directly and lets a VR user explore it.

The repository contains one Unity project with separate iPadOS and Meta Quest
build profiles, shared data contracts, an on-device model runtime and
reconstruction pipeline, and direct local-network transfer. The two devices
require different platform builds, but they share the same repository and most
domain code.

## First Research Milestone

Given one indoor room, the system should:

1. Capture synchronized LiDAR geometry, depth, confidence, and RGB keyframes.
2. Run scan-time and post-scan vision inference locally on the iPad.
3. Produce a textured GLB reconstruction on the iPad at real-world scale.
4. Transfer the reconstruction directly over a local network.
5. Load it on Meta Quest with collision and safe VR locomotion.
6. Report reconstruction accuracy, texture coverage, processing time, file
   size, and Quest performance.

See [the development plan](docs/DEVELOPMENT_PLAN.md) and
[the GitHub issue backlog](docs/GITHUB_ISSUE_BACKLOG.md).

## Planned Repository Layout

```text
AR-VR-Mobile-Project/
  apps/unity/                 Unity project: iPad scanner and Quest viewer
  native/ios/                 Core ML and on-device processing bridge
  services/reference/         Optional desktop benchmark implementation
  contracts/                  Versioned RoomCapture and RoomPackage schemas
  datasets/                   Small, consented test captures only
  docs/                       Architecture, plans, experiments, and results
  tools/                      Validation and developer utilities
```

Large room captures, generated models, credentials, and Unity build outputs
must not be committed to Git.
