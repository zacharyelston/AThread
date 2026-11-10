# AThread — ARKit Registration-Marker Capture App: Design Proposal

Date: 2026-11-10
Status: Draft for discussion

## 1. Summary

This document proposes an iOS capture app built on ARKit. It helps teams reconstruct and stitch 3D meshes of real scenes. The app would:

1. Generate printable registration markers at several distance ranges (about 3 m, 10 m and 50 m, plus a custom size) and print them over Wi-Fi through AirPrint.
2. Detect those markers live in the ARKit camera feed, estimate their 6-DoF pose, and anchor them in the ARKit world frame.
3. Record photos, camera poses, intrinsics, LiDAR mesh and marker observations as one capture bundle. A downstream process can use the bundle to align and merge separate capture sessions.
4. Support a second marker set printed on clothing. The subject can then be tracked, masked or registered on their own, without contaminating the static background solve.

AThread (Ariadne's Thread) is a standalone tool with its own repository. Any later server-side processing (stitching jobs, Rerun visualisation, blob storage) would be deployed onto the Hydra/AKS platform through `hydra-host`, which owns the ArgoCD catalog. The `apollo` project group already covers computer-vision and robotics workloads, so it is the most likely home for that processing.

## 2. Goals and non-goals

**Goals**

- Give each capture session repeatable, metric, uniquely identified tie points that persist across sessions and devices.
- Give metric scale beyond the roughly 5 m practical range of the iPhone/iPad LiDAR scanner.
- Make marker production self-service: choose a range, print from the phone, verify the scale, place the marker, capture.
- Export data in open formats that standard photogrammetry and SLAM tooling can consume.

**Non-goals (first release)**

- On-device full photogrammetric reconstruction of large scenes. The app captures and pre-aligns; heavy reconstruction runs server-side.
- Deformable-cloth reconstruction of the subject. Garment markers are used for subject tracking and masking, not for cloth simulation.
- Android support.

## 3. Marker technology choice

The request refers to "AR-QR codes". Three options were reviewed.

| Option | Strengths | Weaknesses for this use |
| --- | --- | --- |
| **QR code** (Apple Vision `VNDetectBarcodesRequest`) | Built into iOS; can carry a text payload (set ID, size, tier). | Built for data, not geometry. Even a version 1 QR code has 21 modules across, plus a quiet zone, so it needs about 2.6x the printed size of an AprilTag for the same detection range. Vision returns the code's bounds and payload, not a calibrated pose. |
| **ARKit reference images** (`ARReferenceImage`) | Native ARKit anchoring when the physical size is known. | Apple's guidance favours richly textured, non-repetitive, flat images. Square binary fiducials tend to fail Xcode's quality checks. Each marker has to be registered in advance as its own reference image. |
| **AprilTag** (family 36h11, C library, BSD licence) | Designed for pose estimation at low resolution. 587 unique IDs with strong error correction. Widely used in robotics and in VFX tracking (e.g. SynthEyes). No external dependencies, and can be called from Swift through a module map. | Needs a third-party library in the app. It carries no free-text payload, only an ID. |

**Recommendation:** Use **AprilTag 36h11** as the geometric marker. Print a **small QR code beside it** on each sheet that carries the marker metadata (set ID, tag ID, physical size, tier, print date). The QR code is for people and for close-range scanning during setup. The AprilTag does the registration work at distance. For the far tier only, family 16h5 can be considered: it needs about 25 % less printed size for the same range, but has only 30 IDs and a higher false-positive rate.

## 4. Marker sizing for 3 m, 10 m and 50 m

### 4.1 Model

The image size of a marker depends on the camera focal length in pixels (`fx`) and the distance (`d`):

```
pixels per metre at distance d  =  fx / d
required tag edge s            =  modules × pixels_per_module × d / fx
```

- AprilTag 36h11 has 8 modules across the measured edge (the outside of the black border). The printed footprint adds a one-module white border, so footprint = 1.25 × s.
- ARKit's tracking video stream is typically 1920×1440. The iPhone X has a reported `fx` of about 1665 px at that resolution. This document uses **fx ≈ 1500** as a conservative assumption.
- ARKit can also take full-resolution stills during a session (`captureHighResolutionFrame`, iOS 16+). For a 4032×3024 still, `fx` scales to roughly **3150**.
- Working assumptions, to be validated in field tests (section 9): **≈4 px/module** for reliable detection and **≈10 px/module** for good pose accuracy. No published rule of thumb for AprilTag pixel thresholds was found, and vendor documentation recommends testing with the actual camera.

The app should not rely on these constants. It should read `ARFrame.camera.intrinsics` on the actual device and compute sizes live (see 4.3).

### 4.2 Calculated tag edge (black border, excluding white margin)

| Stream | Quality target | 3 m | 10 m | 50 m |
| --- | --- | ---: | ---: | ---: |
| Video, fx ≈ 1500 | Detect (4 px/module) | 6.4 cm | 21 cm | 1.07 m |
| Video, fx ≈ 1500 | Pose (10 px/module) | 16 cm | 53 cm | 2.67 m |
| High-res still, fx ≈ 3150 | Detect (4 px/module) | 3.0 cm | 10 cm | 0.51 m |
| High-res still, fx ≈ 3150 | Pose (10 px/module) | 7.6 cm | 25 cm | 1.27 m |

For comparison, a version 1 QR code at pose quality in the video stream would need 42 cm at 3 m and 1.4 m at 10 m.

### 4.3 Proposed print tiers

Each tier is sized so the marker is **detected live in the video stream** at its rated distance and reaches **pose quality in high-resolution stills**.

| Tier | Rated range | Tag edge | Printed footprint | How to print |
| --- | --- | ---: | ---: | --- |
| **Near** | ≤ 3 m | 16 cm | 20 cm | One US Letter or A4 sheet, AirPrint |
| **Mid** | ≤ 10 m | 25 cm | 32 cm | 2×2 tiled Letter/A4 with crop and alignment marks, AirPrint; or one A2 sheet |
| **Far** | ≤ 50 m | 1.3 m | 1.6 m | Export a vector PDF for a large-format plotter or print shop (matte vinyl banner) |
| **Custom** | User-entered | Calculated | Calculated | Calculated from the device's live intrinsics and the selected quality target |

Notes:

- **Oblique viewing** shrinks the marker's projected width by cos θ. At 60° off-axis the effective size halves, so the app should warn when observations are more oblique than about 45°.
- **Motion blur and low light** reduce the effective pixels per module. Far-tier captures should use stills, not video frames.
- A 50 m marker cannot practically be produced on an office Wi-Fi printer. Tiling it would take around 50 sheets. The app should offer large-format PDF export for this tier rather than AirPrint.

## 5. Printing over Wi-Fi

- Generate each sheet as a **vector PDF** at exact physical dimensions. Print through `UIPrintInteractionController` (AirPrint).
- AirPrint and the printer driver may scale content to fit the printable area. Apple's own guidance notes that printer margins and paper sizes vary. To make the result verifiable:
  - Each sheet carries a **100 mm reference ruler** and the nominal tag size in text.
  - After printing, the app asks the operator to measure the ruler (or the tag edge) and enter the result. It stores the **measured physical size** in the marker-set manifest, which the stitching solver uses as the authoritative value.
- Tiled sheets (Mid tier) include alignment marks and a tile index so they can be assembled squarely on a rigid backing (foam board or aluminium composite).
- Print on matte stock to avoid specular glare. Rigid mounting keeps the marker planar, which both the pose estimate and ARKit's own guidance assume.

## 6. Garment markers (subject-worn registration grids)

Markers on clothing are useful, with one important constraint: **they move with the subject**. They cannot serve as static tie points for the background mesh unless the subject holds still for the whole capture. The design treats them as a separate, dynamic marker set.

**Intended uses**

1. **Subject tracking and masking.** Detected garment markers locate the subject in each frame. The background reconstruction can then mask the subject out instead of fusing a moving body into the static mesh.
2. **Subject registration.** If the subject holds still, or poses on a reference mat, the garment markers register a separate subject mesh to the scene.
3. **Near-field scale reference.** A known-size marker close to the camera gives scale in tight spaces.

**Design guidance**

- Use a **separate ID range** for garment markers (section 7) so the solver never treats them as static scene points.
- Place small tags (about 4–8 cm, Near tier scaled down for 1–2 m working distance) on the flattest regions: chest, upper back, upper arms and thighs. Avoid joints and folds.
- Use dye-sublimation on polyester or a printed heat-transfer patch. Verify the printed size after transfer, because fabric printing can distort scale.
- A grid of tags (an "AprilGrid") holds up better than a single large tag: some tags still detect when folds or occlusion hide others, and each tag contributes four corners.
- Prior work supports the approach. Garment-printed patterns are an established method for cloth and body capture (Utah two-level cloth pattern; GaMoCap; Neophytou et al.). Chic-Marker (ACM SCF 2024) shows square fiducials embedded in a houndstooth pattern by sublimation printing, which is relevant if the garments need to look less technical. DeepFormableTag is a candidate for later work on markers that tolerate deformation.

## 7. Marker-set management

A **marker-set manifest** (JSON) is generated with every print job and travels with every capture:

```json
{
  "set_id": "site-a-2026-11",
  "family": "tag36h11",
  "markers": [
    { "id": 12, "role": "scene",   "tier": "mid",  "nominal_edge_m": 0.25, "measured_edge_m": 0.2485 },
    { "id": 410, "role": "garment", "tier": "near", "nominal_edge_m": 0.06, "measured_edge_m": 0.0596 }
  ]
}
```

Proposed ID allocation within 36h11 (587 IDs):

| Range | Role |
| --- | --- |
| 0–399 | Static scene markers (all tiers) |
| 400–549 | Garment / dynamic markers |
| 550–586 | Scale bars, calibration boards, reserved |

## 8. App architecture

```
┌──────────────────── AThread iOS app ───────────────────┐
│ Marker Studio   Generate tags, size by tier or custom, │
│                 PDF/AirPrint, measured-size entry      │
│ Capture         ARWorldTrackingConfiguration           │
│                  + sceneReconstruction (LiDAR devices) │
│                  + captureHighResolutionFrame stills   │
│ Detector        AprilTag C library on capturedImage    │
│                 → corners + ID → PnP with intrinsics   │
│                 → ARAnchor("tag-<id>") in world frame  │
│ Session store   ARWorldMap, anchors, manifest          │
│ Export          Capture bundle → Azure Blob (optional) │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────── Hydra / AKS (apollo group, later phase) ───────┐
│ Stitch job: align sessions on shared scene tag IDs     │
│ (rigid/similarity fit), refine with ICP or bundle      │
│ adjustment, mask the subject using garment tags        │
│ Visualise in Rerun                                     │
└────────────────────────────────────────────────────────┘
```

**Key implementation points**

- **Device baseline:** iPhone 12 Pro or later, or iPad Pro (2020 or later), for LiDAR scene reconstruction (`ARMeshAnchor`). Non-LiDAR devices still work for photos and markers, with no on-device mesh.
- **iOS baseline:** iOS 17 or later (high-resolution frame capture needs iOS 16).
- **Detection loop:** Run AprilTag detection on a downscaled grayscale copy of `ARFrame.capturedImage` at a reduced rate (e.g. 5–10 Hz) for live feedback. Run full-resolution detection on high-resolution stills.
- **Pose fusion:** Each observation gives a camera-relative tag pose. The app transforms it into ARKit world space using the frame's camera transform and averages repeated observations per tag. It shows a per-tag confidence indicator (observation count, viewing angle, pixels per module).
- **Operator guidance:** A live overlay shows which tags are detected, their estimated pixels per module against the 4/10 thresholds, and the oblique-angle warning, so the operator knows when a marker is too small or too far.

**Capture bundle contents (per session)**

| Item | Format |
| --- | --- |
| Photos (high-resolution stills) | HEIC or JPEG, with EXIF |
| Per-photo camera pose and intrinsics | JSON (ARKit convention, documented) |
| Tag observations (ID, pixel corners, camera-relative pose) | JSON |
| Fused tag anchors in the session world frame | JSON |
| LiDAR mesh | OBJ or USDZ (from `ARMeshAnchor`) |
| ARKit world map | `ARWorldMap` archive (for on-device relocalisation) |
| Marker-set manifest | JSON (section 7) |

## 9. Validation plan

Before the tier sizes are fixed, run a short field test:

1. Print Near, Mid and a cut-down Far sample, using the 4.3 sizes and the 4.2 "detect" minimums.
2. On at least two device models, log detection rate, pixels per module and pose repeatability at 3, 5, 10, 25 and 50 m, frontal and at 30° and 60°, indoors and outdoors.
3. Read `camera.intrinsics` on each device to replace the `fx` assumptions.
4. Adjust the tier sizes and the 4/10 px/module thresholds from the measured data, then update this document.

## 10. Phased delivery

| Phase | Scope |
| --- | --- |
| 0 — Prototype | Marker Studio (Near tier, AirPrint, measured-size entry), live AprilTag detection overlay, field test from section 9 |
| 1 — Capture | High-resolution stills, LiDAR mesh, tag anchors, capture-bundle export, Mid and Far tiers with tiling and large-format PDF |
| 2 — Multi-session | Server-side alignment of sessions on shared tag IDs in the `apollo` group, Rerun visualisation |
| 3 — Subject | Garment marker set, subject masking, optional subject-mesh registration |

## 11. Open questions

1. Which device models will the team use in the field? This sets the LiDAR availability and the measured `fx` values.
2. What is the furthest real working distance? If scenes rarely exceed 25 m, the Far tier can shrink, and so can the print cost.
3. Should the capture bundles land in Azure Blob under an existing storage account, or in a new one owned by the `apollo` group?
4. Is there a preferred downstream reconstruction tool (e.g. Apple Object Capture, COLMAP or an internal pipeline)? This determines the export conventions.
5. For garments, is the priority subject masking (simpler) or subject-mesh registration (needs the subject to hold still)?
6. Long-term GitHub owner for the `AThread` repository (currently a personal account), and who maintains the app.

## 12. References

- Apple — `ARConfiguration.recommendedVideoFormatForHighResolutionFrameCapturing`: <https://developer.apple.com/documentation/arkit/arconfiguration/recommendedvideoformatforhighresolutionframecapturing>
- Apple WWDC22 session 10128, "Discover ARKit 6" (high-resolution frame capture): <https://nonstrict.eu/wwdcindex/wwdc2022/10128/>
- Apple — Detecting images in an AR experience: <https://developer.apple.com/documentation/arkit/detecting-images-in-an-ar-experience>
- Apple — `ARReferenceImage.physicalSize`: <https://developer.apple.com/documentation/arkit/arreferenceimage/physicalsize>
- Apple — `ARCamera.intrinsics`: <https://developer.apple.com/documentation/arkit/arcamera/intrinsics>
- Apple — Visualizing and interacting with a reconstructed scene: <https://developer.apple.com/documentation/arkit/visualizing-and-interacting-with-a-reconstructed-scene>
- Apple — `VNDetectBarcodesRequest`: <https://developer.apple.com/documentation/vision/vndetectbarcodesrequest>
- Apple — `UIPrintInteractionController`: <https://developer.apple.com/documentation/uikit/uiprintinteractioncontroller>
- Apple WWDC14 session 718, "Adopting AirPrint": <https://devstreaming-cdn.apple.com/videos/wwdc/2014/718xxctf8ley20j/718/718_adopting_airprint.pdf>
- University of Michigan APRIL Lab — AprilTag: <https://april.eecs.umich.edu/software/apriltag>
- FTC Docs — AprilTag introduction (36h11 module layout): <https://ftc-docs.firstinspires.org/apriltag-intro>
- Basler rc_cube — Tag detection (range depends on camera and family): <https://docs.baslerweb.com/stereovisard/rc_cube/en/tagdetect>
- Boris FX SynthEyes — Choosing an AprilTag: <https://borisfx.com/documentation/syntheyes/SynthEyesUM_files/choosing_an_apriltag/>
- OpenCV — Calibration with ArUco and ChArUco: <https://docs.opencv.org/5.0/tutorials/objdetect/aruco_calibration/aruco_calibration.html>
- MathWorks — Calibration patterns (AprilGrid, ChArUco): <https://www.mathworks.com/help/vision/ug/calibration-patterns.html>
- Liang et al., "Chic-Marker: Fashionably Fusing Fiducial Markers into Apparel and Accessories", ACM SCF 2024: <https://research.tue.nl/nl/publications/chic-marker-fashionably-fusing-fiducial-markers-into-apparel-and->
- Yaldiz et al., "DeepFormableTag": <https://arxiv.org/abs/2206.08026>
- Utah — Two-level cloth capture pattern: <https://graphics.cs.utah.edu/research/projects/two-level-cloth-capture-pattern/two-level-cloth-capture-pattern.pdf>
- Setti and Del Bue, GaMoCap (2015): <https://iris.univr.it/handle/11562/981295>
- Neophytou et al. (2015), dense garment capture: <https://proc.3dbody.tech/abstracts/2015/120neophytou.html>
- Stack Overflow — RealityKit/ARKit focal length (iPhone X intrinsics at 1920×1440): <https://stackoverflow.com/questions/62623726>
