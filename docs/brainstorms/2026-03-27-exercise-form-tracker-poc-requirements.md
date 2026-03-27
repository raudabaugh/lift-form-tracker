---
date: 2026-03-27
topic: exercise-form-tracker-poc
---

# Exercise Form Tracker POC

## Problem Frame

Validate whether phone-camera pose estimation is accurate and fast enough to provide useful real-time form feedback during a strength training workout. The prototype must answer 5 specific questions about pose estimation quality (depth detection, dumbbell occlusion, latency, squat depth discrimination, and partial occlusion handling) before committing to a full product build.

Two engineers will test the prototype live in the Dominican Republic this week using 3 exercises from a Peloton strength class.

## Requirements

- R1. **Camera + skeleton overlay** — Access the phone camera and render a real-time MediaPipe Pose skeleton overlay (33 keypoints, connections drawn) on the video feed. Must work in iPhone Safari and Android Chrome via HTTPS.

- R2. **Exercise selector** — A simple UI control to select the active exercise (Squat, Deadlift/RDL, Overhead Press). Only the angles relevant to the selected exercise are computed and displayed.

- R3. **Angle computation** — Compute and display the following joint angles as numeric readouts in real time:
  - **Squat**: knee angle (thigh-to-shin), hip angle (torso-to-thigh)
  - **Deadlift / RDL**: hip angle (torso-to-thigh), back angle relative to vertical (spine tilt)
  - **Overhead Press**: elbow extension angle (upper arm-to-forearm), shoulder elevation angle

- R4. **Color-coded feedback** — Each angle readout is color-coded green / yellow / red against configurable thresholds. Default thresholds (tunable via constants in source):
  - Squat knee angle: green ≤ 95°, yellow 96–115°, red > 115°
  - Squat hip angle: green ≤ 100°, yellow 101–120°, red > 120°
  - Deadlift hip angle: green ≤ 90°, yellow 91–110°, red > 110°
  - Deadlift back angle (from vertical): green 30–60°, yellow 20–30°, red < 20° or > 70°
  - Overhead press elbow: green ≥ 160°, yellow 140–159°, red < 140°
  - Overhead press shoulder: green ≥ 150°, yellow 120–149°, red < 120°

- R5. **Session logging** — Capture angle values + timestamp at ~5 fps into an in-memory log. A "Download Log" button exports the session as a JSON file for post-workout review.

- R6. **Confidence filtering** — Suppress angle readouts and color feedback for any joint whose MediaPipe visibility score falls below a threshold (default: 0.5). Show a "joint hidden" indicator instead of a misleading number.

- R7. **Staged build** — The prototype is built in 3 independently testable stages:
  - Stage 1: Camera + skeleton overlay only (R1)
  - Stage 2: Angle readouts added (R1–R3, R6)
  - Stage 3: Color feedback + logging added (R1–R6)

## Success Criteria

- Stage 1 runs at ≥ 20 fps on a mid-range phone (iPhone 12 or equivalent) with no dropped camera frames
- Skeleton tracks the full body reliably from a phone propped at 3–4 ft away at floor level, portrait or landscape
- Squat knee angle measurably distinguishes a parallel squat from a quarter squat (≥ 20° delta)
- Color feedback updates within one rep cycle without visible lag
- A session log downloads successfully and contains timestamped angle data

## Scope Boundaries

- No backend, no server — 100% client-side
- No automatic exercise detection — user selects exercise manually
- No rep counting in this POC
- No user accounts, history, or persistence beyond the current session download
- No audio cues or haptic feedback
- No support for exercises beyond the 3 selected (squat, deadlift/RDL, overhead press)
- Angle thresholds are hardcoded constants, not user-configurable in the UI

## Key Decisions

- **MediaPipe Pose over MoveNet**: 33 keypoints vs 17 is necessary for back angle (spine landmarks), wrist tracking during press, and occlusion recovery. The ~15ms latency difference is acceptable.
- **`@mediapipe/tasks-vision` (Tasks API)**: Newer, better-maintained than the legacy `@mediapipe/pose` package. WASM-based, no Node.js required.
- **GitHub Pages or Netlify for deployment**: Provides HTTPS without a backend, accessible from any phone without local network setup. Critical for testing in DR.
- **5 fps log sampling**: Enough temporal resolution for post-workout analysis without excessive memory use during a 10–20 min session.
- **Visibility threshold at 0.5**: Suppresses misleading angle values when joints are occluded (e.g., the hidden arm during lateral deadlifts).

## Dependencies / Assumptions

- MediaPipe Tasks Vision JS bundle is loaded from CDN or bundled — no build toolchain required for Stage 1
- Phone is positioned at ~3–4 ft, floor level, facing the user — no dynamic camera placement detection
- Adequate lighting — no low-light compensation in scope
- `getUserMedia` works on the target browsers when served over HTTPS

## Outstanding Questions

### Resolve Before Planning
_(none — requirements are sufficient to proceed)_

### Deferred to Planning
- [Affects R1][Needs research] What is the simplest zero-build delivery method? (Plain HTML + CDN import, or Vite for Stage 2+?) Evaluate based on Safari WASM loading constraints.
- [Affects R3][Technical] Which MediaPipe landmark indices map to each joint for the 3 exercises? Verify against the 33-point BlazePose landmark map.
- [Affects R4][Needs research] Are the default angle thresholds physiologically reasonable? May need adjustment after first live test.
- [Affects R7][Technical] Should each stage be a separate HTML file or a single file with feature flags? Separate files are simpler to test independently.

## Next Steps

→ `/ce:plan` for structured implementation planning
