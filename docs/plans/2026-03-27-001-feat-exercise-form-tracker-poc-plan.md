---
title: "feat: Exercise Form Tracker POC — Camera Pose Estimation"
type: feat
status: active
date: 2026-03-27
origin: docs/brainstorms/2026-03-27-exercise-form-tracker-poc-requirements.md
---

# feat: Exercise Form Tracker POC — Camera Pose Estimation

## Overview

Build a three-stage, browser-only HTML prototype that uses MediaPipe Pose to provide real-time joint angle tracking and color-coded form feedback for squat, deadlift/RDL, and overhead press. Deployed to GitHub Pages for same-day phone testing. Each stage is a standalone HTML file tested independently.

## Problem Frame

Validate whether phone-camera pose estimation (MediaPipe BlazePose) is accurate and fast enough to give useful real-time form feedback during strength training. The prototype must answer 5 questions: depth detection reliability, dumbbell occlusion handling, perceived latency, squat depth discrimination (good vs. shallow), and partial occlusion handling from a lateral angle.

Two engineers test this week in the Dominican Republic using iPhones and Android phones. No app store. No backend. No build toolchain.

## Requirements Trace

- R1. Camera + skeleton overlay — live video with 33-point BlazePose connections drawn on canvas
- R2. Exercise selector — buttons for Squat / Deadlift/RDL / Overhead Press; switches active angle set
- R3. Angle computation — joint angles for each exercise; left/right sides selected by visibility
- R4. Color-coded feedback — green/yellow/red per angle against hardcoded thresholds
- R5. Session logging — 5 fps in-memory capture, Web Share API JSON export
- R6. Confidence filtering — suppress angles when landmark visibility < 0.5
- R7. Staged build — three independently testable HTML files, each a superset of the prior stage

## Scope Boundaries

- No backend, no build toolchain, no Node.js — static HTML + CDN imports only
- No rep counting, no audio/haptic, no user accounts
- Exercises limited to squat, deadlift/RDL, overhead press
- Angle thresholds are source-level constants, not configurable in UI
- No automatic exercise detection
- No PWA/home-screen installation (avoids iOS camera permission persistence bug)

## Context & Research

### Relevant Code and Patterns

No existing codebase — greenfield static HTML prototype.

### External References

- `@mediapipe/tasks-vision@0.10.21` — last stable npm release as of March 2026 (subsequent versions dropped vision tasks); fixes `DrawingUtils` export bug from 0.10.19–0.10.20
- jsDelivr CDN: `https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.21/vision_bundle.mjs`
- WASM base path: `https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.21/wasm`
- Model: `pose_landmarker_lite` from Google GCS (~4 MB), GPU delegate uses WebGL2 (25–35 fps on iPhone 12 class hardware), CPU fallback (~10–15 fps)
- `PoseLandmarker.POSE_CONNECTIONS` — static constant encoding all 33-point skeleton edges; no manual connection list needed
- iOS Safari requires `autoplay muted playsinline` on `<video>`; hidden video causes black canvas on iOS 16 — keep video visible and layer canvas via `position: absolute`
- `requestVideoFrameCallback` available in Safari 15.4+ — fires exactly once per decoded frame; better than rAF for this pipeline
- iOS `download` attribute on `<a>` is silently ignored — must use `navigator.share({ files: [...] })` Web Share API for JSON export; files must be the only property in the share object
- `facingMode: { exact: 'environment' }` for iOS rear camera — do not use `deviceId` (unreliable on iOS)
- Wait for `loadedmetadata` event before reading `videoWidth`/`videoHeight` (iOS can return 0 in portrait before this fires)
- GitHub Pages WASM MIME: not an issue when WASM is fetched from jsDelivr CDN (not from GitHub Pages origin); include `.nojekyll` anyway

## Key Technical Decisions

- **Zero-build delivery**: Plain HTML `<script type="module">` with jsDelivr CDN import. No Vite, no bundler. Eliminates all toolchain friction for a POC that must ship in days. Each stage is a self-contained `.html` file.

- **Separate HTML files per stage** (not a single file with feature flags): `stage1.html`, `stage2.html`, `stage3.html`. Each file is fully independent — paste-in-browser, no server context required. Isolates regressions between stages during testing.

- **`pose_landmarker_lite` + GPU delegate**: Lite model (~4 MB) is the intended mobile model. GPU delegate uses WebGL2; library auto-falls-back to CPU when WebGL2 is unavailable. `outputSegmentationMasks: false` disables the most expensive optional feature.

- **Left/right side selection by visibility**: For each exercise, compute angles for both the left and right sides. Display the side with higher average landmark visibility for the joints involved. If both sides have adequate visibility (> 0.5), show both. This handles the lateral camera angle occlusion scenario.

- **2D angle math only**: Use normalized landmark `(x, y)` coordinates. Do not use `z` or `worldLandmarks`. The 2D projection is sufficient for all target angles and is simpler to verify visually against the skeleton overlay.

- **`requestVideoFrameCallback` for inference loop**: Fires at the camera's native frame rate (typically 30 fps on mobile) rather than the display's 60 Hz, cutting wasted inference calls in half compared to rAF. Graceful degradation: if `rVFC` is unavailable, fall back to rAF.

- **Web Share API for JSON export**: `navigator.share({ files: [...] })` is the only reliable export path on iOS Safari. Fall back to `URL.createObjectURL` + anchor `.click()` on desktop browsers. The files array must be the sole property in the share object (iOS requirement).

- **Canvas layering**: Visible `<video>` element with canvas overlay via `position: absolute`. Video is the live preview; canvas draws the skeleton. Single overlay canvas (not split video/overlay) is sufficient for this POC — the skeleton is only drawn when inference completes, not every display frame.

## Open Questions

### Resolved During Planning

- **Plain HTML vs Vite**: Plain HTML with CDN imports. WASM loads from jsDelivr CDN (not GitHub Pages origin), so the GitHub Pages WASM MIME issue is bypassed entirely.
- **Separate files vs feature flags**: Separate `stage1.html / stage2.html / stage3.html` (see decision above).
- **Landmark indices**: Confirmed from API docs (see Implementation Unit 2).
- **Angle threshold physiological validity**: Thresholds are directional defaults. A first live test will likely require tuning — document the constants clearly so values can be changed in seconds.

### Deferred to Implementation

- **Cold-start UX**: MediaPipe model initialization can take 3–8 seconds on first load (model fetch + WASM compile). Implementer should add a visible loading indicator and disable the camera start button until ready.
- **Portrait vs landscape layout**: CSS layout choices for phone-propped-on-floor vs hand-held portrait are left to implementer judgment; functional correctness is the priority.
- **Angle threshold tuning**: After the first live workout test, constants will likely need adjustment. Document each constant with the exercise name and target range for easy tuning.

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.*

**Data flow (each video frame):**

```
[Camera stream]
  → <video autoplay muted playsinline>
  → requestVideoFrameCallback fires
  → poseLandmarker.detectForVideo(video, timestamp)
  → PoseLandmarkerResult { landmarks: NormalizedLandmark[33][] }
  → visibility filter: skip joints where visibility < 0.5
  → angle calculations (2D, using x/y only)
  → color classification per angle (thresholds table)
  → ctx.clearRect + drawConnectors + drawLandmarks  (Stage 1+)
  → angle readout panel update (Stage 2+)
  → 5fps log append (Stage 3)
```

**Angle calculation pattern (vector angle at vertex B):**

```
function angleDeg(A, B, C):
  BA = { x: A.x - B.x, y: A.y - B.y }
  BC = { x: C.x - B.x, y: C.y - B.y }
  dot = BA.x*BC.x + BA.y*BC.y
  mag = |BA| * |BC|
  return arccos(clamp(dot/mag, -1, 1)) * (180/π)
```

**Back angle from vertical (deadlift spine tilt):**

```
spineMid_shoulder = avg(pose[11], pose[12])
spineMid_hip      = avg(pose[23], pose[24])
spineVec = { x: spineMid_shoulder.x - spineMid_hip.x,
             y: spineMid_shoulder.y - spineMid_hip.y }
// In MediaPipe coords, y increases downward.
// Standing = spineVec points up = negative y.
// "Vertical up" reference = { x:0, y:-1 }
backAngle = arccos(dot(normalize(spineVec), {x:0,y:-1})) * (180/π)
```

**Angle-to-color classification:**

```
function classify(angle, rule):
  if rule.greenIncludes(angle) → GREEN
  if rule.yellowIncludes(angle) → YELLOW
  else → RED
```

**BlazePose landmark indices used across all three exercises:**

| Index | Joint |
|---|---|
| 11 | LEFT_SHOULDER |
| 12 | RIGHT_SHOULDER |
| 13 | LEFT_ELBOW |
| 14 | RIGHT_ELBOW |
| 15 | LEFT_WRIST |
| 16 | RIGHT_WRIST |
| 23 | LEFT_HIP |
| 24 | RIGHT_HIP |
| 25 | LEFT_KNEE |
| 26 | RIGHT_KNEE |
| 27 | LEFT_ANKLE |
| 28 | RIGHT_ANKLE |

**Per-exercise angle definitions:**

| Exercise | Angle name | Vertex | Point A | Point C |
|---|---|---|---|---|
| Squat | Knee angle | 25/26 (knee) | 23/24 (hip) | 27/28 (ankle) |
| Squat | Hip angle | 23/24 (hip) | 11/12 (shoulder) | 25/26 (knee) |
| Deadlift | Hip angle | 23/24 (hip) | 11/12 (shoulder) | 25/26 (knee) |
| Deadlift | Back angle | — (spine tilt, see formula above) | — | — |
| OHP | Elbow extension | 13/14 (elbow) | 11/12 (shoulder) | 15/16 (wrist) |
| OHP | Shoulder elevation | 11/12 (shoulder) | 23/24 (hip) | 13/14 (elbow) |

## Implementation Units

- [ ] **Unit 1: Stage 1 — Camera + Skeleton Overlay** (`stage1.html`)

**Goal:** Working camera feed with real-time BlazePose skeleton overlay. First testable checkpoint.

**Requirements:** R1, R7 (Stage 1)

**Dependencies:** None

**Files:**
- Create: `stage1.html`
- Create: `.nojekyll` (repo root, prevents Jekyll processing on GitHub Pages)
- Create: `index.html` (root landing page with links to all three stages + phone testing instructions)

**Approach:**
- Single HTML file with inline `<style>` and `<script type="module">`
- Imports: `PoseLandmarker`, `FilesetResolver`, `DrawingUtils` from jsDelivr `vision_bundle.mjs` at version 0.10.21 (pinned)
- WASM loaded via `FilesetResolver.forVisionTasks` pointing at jsDelivr `/wasm` path for the same pinned version
- Model: `pose_landmarker_lite` from Google GCS, `delegate: "GPU"`, `runningMode: "VIDEO"`, `numPoses: 1`, `outputSegmentationMasks: false`
- Camera: `getUserMedia({ video: { facingMode: { exact: 'environment' }, width: { ideal: 640 }, height: { ideal: 480 } }, audio: false })`. Falls back to `facingMode: 'environment'` (without `exact`) if `OverconstrainedError` is thrown (some devices don't have a labeled rear camera)
- Video element: `autoplay muted playsinline`, kept visible
- Canvas: `position: absolute` over video. Set `canvas.width / canvas.height` from `videoWidth / videoHeight` inside `loadedmetadata` handler
- Inference loop: prefer `video.requestVideoFrameCallback(loop)` for frame sync; fall back to `requestAnimationFrame(loop)` if `rVFC` is absent
- Each frame: call `poseLandmarker.detectForVideo(video, performance.now())`, then `ctx.clearRect`, `drawConnectors`, `drawLandmarks` using `PoseLandmarker.POSE_CONNECTIONS`
- Skip inference when `video.readyState < 2`
- Add a visible FPS counter (updated every ~30 frames) so the tester can immediately assess performance
- Show a loading overlay ("Initializing pose model…") during WASM + model fetch; remove it once `PoseLandmarker` is ready

**Technical design:** See High-Level Technical Design section.

**Patterns to follow:**
- Standard MediaPipe Tasks Vision initialization pattern from [ai.google.dev guide](https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker/web_js)
- Complete HTML skeleton from research (pinned version, `vision_bundle.mjs`, `autoplay muted playsinline`)

**Test scenarios:**
- Page loads on iPhone Safari (HTTPS) and shows "Initializing pose model…", then camera starts without a gesture (autoplay)
- Skeleton connections appear on a standing person at 3–4 ft camera distance
- FPS counter shows ≥ 20 fps on iPhone 12 or equivalent
- All 33 keypoints appear roughly correctly positioned on the body
- No console errors after initialization

**Verification:**
- Open `stage1.html` via GitHub Pages URL on both test phones; camera and skeleton render within 10 seconds of page load

---

- [ ] **Unit 2: Stage 2 — Angle Computation** (`stage2.html`)

**Goal:** Exercise selector plus real-time numeric joint angle readouts, with confidence filtering. Second testable checkpoint.

**Requirements:** R2, R3, R6, R7 (Stage 2)

**Dependencies:** Unit 1 complete and validated

**Files:**
- Create: `stage2.html` (full copy of stage1.html with angle additions)

**Approach:**
- Add three exercise selector buttons (Squat / Deadlift/RDL / Overhead Press) at the top of the page; active exercise is stored in a variable; buttons update it on click
- Implement `angleDeg(A, B, C)` using 2D vector math (see High-Level Technical Design). `A`, `B`, `C` are `NormalizedLandmark` objects with `.x` and `.y` properties
- Implement `backAngleDeg(pose)` using the spine-midpoint-to-vertical formula (see High-Level Technical Design)
- For each exercise, define an `ANGLES` config object listing the angles to compute and their landmark triplets (or special-case function)
- **Visibility filtering**: for any angle where any of its three constituent landmarks has `visibility < 0.5`, display "hidden" instead of a number
- **Side selection**: for paired (left/right) landmarks, compute both sides. Show the side with higher average visibility. If both sides have visibility > 0.5, show both
- Render angle readouts in a fixed overlay panel (e.g., bottom of screen). Format: `Knee: 88°` in a large readable font — tester needs to see this while exercising
- Update readouts inside the inference callback (not on a separate timer)

**Technical design:**

```
// Angle config structure (directional sketch, not code to reproduce):
const EXERCISES = {
  squat: [
    { label: "Knee", vertex: [25,26], pointA: [23,24], pointC: [27,28] },
    { label: "Hip",  vertex: [23,24], pointA: [11,12], pointC: [25,26] }
  ],
  deadlift: [
    { label: "Hip",  vertex: [23,24], pointA: [11,12], pointC: [25,26] },
    { label: "Back", special: "backAngle" }
  ],
  ohp: [
    { label: "Elbow",    vertex: [13,14], pointA: [11,12], pointC: [15,16] },
    { label: "Shoulder", vertex: [11,12], pointA: [23,24], pointC: [13,14] }
  ]
}
```

**Patterns to follow:**
- Same MediaPipe init as `stage1.html` (copy verbatim)
- `angleDeg` is a pure function — isolate it from DOM manipulation for easy debugging

**Test scenarios:**
- Squat selector active: Knee and Hip angles display numeric values while squatting
- Deadlift selector active: Hip and Back angles display; Back angle increases as user hinge-bends forward
- OHP selector active: Elbow angle approaches 180° at lockout
- Covering a landmark (arm behind body): readout shows "hidden" for affected angles
- Holding dumbbells: readouts still display (key validation question)
- Squat depth discrimination: parallel squat (thighs parallel to floor) vs quarter squat shows ≥ 20° difference on Knee angle

**Verification:**
- All six angle types show plausible values (within ±15° of expected for a known pose)
- "hidden" state appears when a hand or knee is covered

---

- [ ] **Unit 3: Stage 3 — Color Feedback + Session Logging** (`stage3.html`)

**Goal:** Color-coded angle feedback and downloadable session log. Third testable checkpoint.

**Requirements:** R4, R5, R7 (Stage 3), carries forward R1-R3, R6

**Dependencies:** Unit 2 complete and validated

**Files:**
- Create: `stage3.html` (full copy of stage2.html with color + logging additions)

**Approach:**

**Color feedback:**
- Define a `THRESHOLDS` constants table at the top of the script (tunable by editing the file):
  ```
  THRESHOLDS = {
    squat:    { knee: { green: [0,95], yellow: [96,115] },
                hip:  { green: [0,100], yellow: [101,120] } },
    deadlift: { hip:  { green: [0,90], yellow: [91,110] },
                back: { green: [30,60], yellow: [20,70] } },   // red outside 20–70
    ohp:      { elbow:    { green: [160,180], yellow: [140,159] },
                shoulder: { green: [150,180], yellow: [120,149] } }
  }
  ```
- `classify(angle, threshold)` → `"green"` | `"yellow"` | `"red"` | `"hidden"`
- Apply color as CSS text color on each readout (green `#22c55e`, yellow `#eab308`, red `#ef4444`)
- Optionally: render a colored dot next to each readout label

**Session logging:**
- Maintain an in-memory array: `log = []`
- Log entry shape: `{ ts: Date.now(), exercise: "squat", angles: { knee: 88, hip: 102, ... } }`
- Append at ~5 fps: use a `lastLogTime` variable; skip the append when `performance.now() - lastLogTime < 200`
- "Download Log" button fixed in corner of screen
- On click:
  1. `const blob = new Blob([JSON.stringify(log, null, 2)], { type: 'application/json' })`
  2. Try `navigator.canShare({ files: [new File([blob], 'form-log.json')] })` → if true, `navigator.share({ files: [...] })` (iOS path)
  3. Fall back to `URL.createObjectURL` + anchor click (desktop Chrome/Firefox path)
- Add a "Entries: N" counter next to the download button so testers can confirm logging is running

**Patterns to follow:**
- Same MediaPipe init + angle logic as `stage2.html` (copy verbatim)
- Threshold constants at the top of `<script>` with clear comments for each exercise

**Test scenarios:**
- Squat to parallel: Knee angle readout turns green; stopping at quarter depth turns red
- Deadlift with flat back: Back angle in green range; standing upright turns red (angle too small)
- OHP at lockout: Elbow angle turns green
- Download button on iPhone: iOS share sheet opens with `form-log.json` as a file
- Download button on Android/desktop: file downloads directly
- Log entries accumulate at ~5 fps; 10-minute session produces ~3000 entries

**Verification:**
- Color state changes visually within one rep cycle
- Downloaded JSON is valid and contains timestamped angle data with correct exercise label

---

- [ ] **Unit 4: GitHub Pages Deployment**

**Goal:** Both test phones can access all three stages via a public HTTPS URL.

**Requirements:** Deployment prerequisite for all R1–R6 testing

**Dependencies:** Units 1–3

**Files:**
- Create: `.nojekyll` (empty file at repo root — prevents Jekyll from processing HTML files and blocking WASM loading)
- Create: `index.html` (landing page with links to stage1.html / stage2.html / stage3.html; brief phone setup instructions)
- Verify: `stage1.html`, `stage2.html`, `stage3.html` all present at repo root

**Approach:**
- Push to a GitHub repo (public or private with Pages enabled)
- Enable GitHub Pages from Settings → Pages → Source: `main` branch, root `/`
- Pages URL will be `https://[username].github.io/[repo]/`
- Both testers open the URL in Safari (iPhone) and Chrome (Android); no app installation needed
- Do NOT use PWA `apple-mobile-web-app-capable` meta tag — this causes camera permission not to persist across launches on iOS

**Test scenarios:**
- Both phones open URL in browser; camera permission prompt appears once; subsequent refreshes do not re-prompt (Safari behavior — permission persists in browser sessions)
- Page loads over LTE/WiFi; model downloads in < 30 seconds on typical mobile connection (~4 MB model + ~10 MB WASM)

**Verification:**
- `https://[username].github.io/[repo]/stage1.html` renders camera feed on both test phones

## System-Wide Impact

- **No external surfaces**: purely client-side; no API calls except CDN resource fetches (jsDelivr, Google GCS for model file)
- **CDN dependency**: if jsDelivr or Google GCS is unavailable, the app fails to load. Acceptable risk for a same-week POC
- **Main thread blocking**: `detectForVideo` is synchronous and runs on the main thread. On GPU delegate, this is typically < 35ms per frame; on CPU fallback it can reach 100ms. If the main thread is blocked for extended periods, the UI will freeze. No mitigation in scope for this POC
- **Memory**: in-memory log at 5 fps × 20 min = ~6000 entries × ~200 bytes each ≈ 1.2 MB. Acceptable; no cleanup needed
- **No state persistence**: all session data is lost on page close unless downloaded. This is intentional per scope

## Risks & Dependencies

- **GPU cold-start latency (high likelihood)**: First model load can take 5–10 seconds including WASM compile + model fetch + GPU context init. This is expected behavior. Mitigation: loading overlay so testers know to wait.
- **iOS `facingMode: { exact: 'environment' }` may reject on some older iPhones (low likelihood)**: If `OverconstrainedError` is thrown, retry with `facingMode: 'environment'` (without `exact`). Implementer should add this fallback.
- **Lite model tracking loss during fast lateral movement (medium likelihood)**: The lite model trades accuracy for speed. Rapid lateral motion during exercises may cause brief landmark dropout. Mitigation: confidence filtering (R6) prevents misleading readouts; tester should note this in their validation assessment.
- **jsDelivr version pinning**: `@0.10.21` is pinned explicitly. If jsDelivr CDN has an outage or the package is deprecated, app fails. Acceptable for a same-week POC; would address with local bundling in a production build.
- **Angle threshold calibration (expected)**: Default thresholds are physiologically reasonable estimates but will almost certainly need tuning after the first live test. Thresholds are single-point constants in each file — easy 5-minute fix.

## Sources & References

- **Origin document:** [docs/brainstorms/2026-03-27-exercise-form-tracker-poc-requirements.md](../brainstorms/2026-03-27-exercise-form-tracker-poc-requirements.md)
- MediaPipe Tasks Vision API: [ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker/web_js](https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker/web_js)
- DrawingUtils class: [ai.google.dev/edge/api/mediapipe/js/tasks-vision.drawingutils](https://ai.google.dev/edge/api/mediapipe/js/tasks-vision.drawingutils)
- PoseLandmarker class: [developers.google.com/mediapipe/api/solutions/js/tasks-vision.poselandmarker](https://developers.google.com/mediapipe/api/solutions/js/tasks-vision.poselandmarker)
- BlazePose landmark overview: [research.google/blog/on-device-real-time-body-pose-tracking-with-mediapipe-blazepose](https://research.google/blog/on-device-real-time-body-pose-tracking-with-mediapipe-blazepose/)
- getUserMedia mobile constraints: [blog.addpipe.com/getusermedia-video-constraints](https://blog.addpipe.com/getusermedia-video-constraints/)
- requestVideoFrameCallback: [MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback)
- Web Share API (iOS file export): [web.dev/patterns/files/share-files](https://web.dev/patterns/files/share-files)
- iOS hidden video black canvas bug: [WebKit Bug 215884](https://bugs.webkit.org/show_bug.cgi?id=215884)
- iOS camera permission in standalone: [WebKit Bug 215884](https://bugs.webkit.org/show_bug.cgi?id=215884)
- `DrawingUtils` export fix in 0.10.21: [GitHub Issue #5790](https://github.com/google-ai-edge/mediapipe/issues/5790)
