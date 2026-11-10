# AGENTS.md — Ariadne

Guidance for AI agents working in the `ariadne` repository.

## Repository identity

`ariadne` owns the source code and design documents for the Ariadne iOS capture tool: ARKit capture, AprilTag registration markers, AirPrint marker printing, and capture-bundle export.

It does **not** own cluster or deployment configuration. Server-side workloads (stitching, visualisation) are deployed through `hydra-host` and its ArgoCD catalog.

## Conventions

- Write design notes and proposals in Markdown under `docs/`. Use a professional, factual tone and cite sources.
- Research before implementing. Record measured values (e.g. device intrinsics, detection thresholds) in `docs/` when they replace assumptions.
- Marker ID ranges, tier sizes and the marker-set manifest format are defined in `docs/design-proposal.md`. Update that document when they change.

## Forbidden actions

- Do not commit secrets, signing certificates, provisioning profiles, API keys or storage credentials.
- Do not commit capture data (photos, meshes, world maps) to Git.
