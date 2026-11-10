# AGENTS.md — AThread

Guidance for AI agents working in the `AThread` repository.

## Repository identity

`AThread` owns the source code and design documents for the AThread (Ariadne's Thread) iOS capture tool: ARKit capture, AprilTag registration markers, AirPrint marker printing, and capture-bundle export.

Server-side processing (multi-session stitching, visualisation) is planned for a later phase and is not yet part of this repository.

## Conventions

- Write design notes and proposals in Markdown under `docs/`. Use a professional, factual tone and cite sources.
- Research before implementing. Record measured values (e.g. device intrinsics, detection thresholds) in `docs/` when they replace assumptions.
- Marker ID ranges, tier sizes and the marker-set manifest format are defined in `docs/design-proposal.md`. Update that document when they change.

## Forbidden actions

- Do not commit secrets, signing certificates, provisioning profiles, API keys or storage credentials.
- Do not commit capture data (photos, meshes, world maps) to Git.
