# AThread

AThread (Ariadne's Thread) is an iOS capture tool built on ARKit. It produces printable registration markers and uses them to anchor photo and LiDAR captures, so separate capture sessions can be aligned and stitched into one 3D mesh.

The name refers to Ariadne's thread, which in the myth marks the way through the labyrinth. AThread's markers do the same job for a scene.

## Status

Design stage. No application code yet. See [`docs/design-proposal.md`](docs/design-proposal.md).

## Planned capabilities

- **Marker Studio:** generates AprilTag (36h11) registration markers in distance tiers (Near ≤3 m, Mid ≤10 m, Far ≤50 m, Custom) and prints them over Wi-Fi via AirPrint, with a measured-size check to correct printer scaling.
- **Capture:** detects markers live in ARKit, anchors them in the world frame, and records high-resolution stills, camera poses, intrinsics and the LiDAR mesh.
- **Garment markers:** a separate, dynamic marker set printed on clothing, used to track and mask the subject without affecting the static background solve.
- **Export:** writes a capture bundle in open formats for downstream multi-session alignment. That alignment is planned as a separate server-side step.

## Repository layout

```
docs/   Design documents and research notes
```

The Xcode project will be added under `app/` once the design is agreed.
