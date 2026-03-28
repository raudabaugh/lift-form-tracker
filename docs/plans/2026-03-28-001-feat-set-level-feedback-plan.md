---
title: "feat: Set-Level Feedback, Rep Tracking, and Threshold Configuration"
type: feat
status: completed
date: 2026-03-28
origin: docs/brainstorms/2026-03-28-001-set-level-feedback-requirements.md
---

# feat: Set-Level Feedback, Rep Tracking, and Threshold Configuration

## Overview

Evolve the form tracker from a real-time color display into a set-level coaching tool. Adds automatic rep detection with manual correction, an end-of-set summary screen, a trainer-configurable threshold editor, and a complete logging overhaul that captures all angle data regardless of selected exercise or landmark visibility.

Implemented as a new `stage4.html` that builds on the patterns established in `stage3.html`.

## Problem Frame

POC testing showed the technology works but the product model is wrong. Real-time color feedback is noise during a rep. The valuable moment is between sets. Logging only captures the selected exercise's filtered angles, hiding the data needed to calibrate thresholds. Thresholds are hardcoded and require a code edit to adjust per-client.

(see origin: docs/brainstorms/2026-03-28-001-set-level-feedback-requirements.md)

## Requirements Trace

- R1. Automatic rep detection — primary angle state machine with live counter
- R2. Rep count correction — `+` / `−` buttons and reset action
- R3. End Set trigger — manual button + optional auto-detect after 3s neutral
- R4. Between-set summary — full-screen takeover with stats and quality breakdown
- R5. Threshold editor — per-angle inputs with localStorage persistence
- R6. Complete logging — all 6 angles per frame, raw values, visibility scores, set/rep context
- R7. Set structure in log — typed events (set_start, rep_detected, set_end, threshold_changed)

## Scope Boundaries

- No backend — localStorage + in-memory log only
- No automatic exercise detection — manual selection only
- No video recording or playback
- One threshold profile per device
- No audio/haptic cues
- `stage3.html` is preserved unchanged; `stage4.html` is the new build

## Context & Research

### Relevant Code and Patterns

- `stage3.html` — complete source to fork. All patterns (MediaPipe init, camera startup, rVFC loop, DrawingUtils, exercise config, THRESHOLDS, classify(), computeAngle(), maybeLog(), downloadLog(), Web Share API export) are established here and carried forward unchanged unless specified otherwise.
- `EXERCISES` config object (stage3.html:289) — defines angle definitions per exercise; add a `repAngle` field to designate the primary angle for rep detection per exercise.
- `THRESHOLDS` constant (stage3.html:229) — threshold structure to make mutable and localStorage-backed.
- `onFrame()` (stage3.html:618) — inference loop where rep state machine runs.
- `maybeLog()` (stage3.html:330) — to be replaced with complete logging.

### Institutional Learnings

None in `docs/solutions/` — this is a greenfield POC project.

### External References

- iOS Web Share API file size: no hard documented limit, but files under 2 MB succeed reliably. A 10-min session at 5fps with full angle data is ~1.2 MB — within range. No compression needed.
- localStorage: synchronous, no async needed, survives page refresh, scoped per origin. Key: `form-tracker-thresholds`.

## Key Technical Decisions

- **`stage4.html` as new file, not in-place edit of stage3.html**: Keeps the original three stages intact as reference points. Stage 4 is a superset of Stage 3 plus the new features.

- **Rep detection trigger = upper bound of yellow range per primary angle**: A rep requires reaching at least yellow-quality depth. Red attempts are not counted. This couples the rep counter to the quality threshold, which is intentional — if the threshold is wrong, both the color feedback and the rep count will be wrong in the same direction, surfacing calibration needs quickly.
  - Squat primary angle: knee. Trigger threshold: `THRESHOLDS.squat.knee.yellow[1]` (default 115°, lower = deeper)
  - Deadlift primary angle: hip. Trigger threshold: `THRESHOLDS.deadlift.hip.yellow[1]` (default 110°)
  - OHP primary angle: elbow. Trigger threshold: `THRESHOLDS.ohp.elbow.yellow[0]` (default 140°, higher = more extended)

- **Two-state rep machine with 2-frame debounce**: `inRep: false → true` when primary angle crosses trigger threshold and stays there for 2+ consecutive frames. `inRep: true → false` (rep counted) when angle crosses back through the trigger threshold toward neutral. Peak angle is tracked during the `inRep=true` phase for quality classification.

- **Auto end-of-set = 3s at neutral, cancellable**: "Neutral" defined per exercise as: squat knee > 155°, deadlift hip > 155°, OHP elbow < 120°. If the primary angle has been in the neutral range continuously for 3 seconds AND at least 1 rep has been counted in the current set, show a 3-2-1 countdown overlay. Tapping anywhere cancels it.

- **Threshold editor exposes one key number per angle** (not all four bounds): for "lower is better" angles, expose the green upper bound (target depth). For "higher is better" angles, expose the green lower bound (target lockout). For the symmetric back angle, expose both bounds. Yellow range auto-adjusts as ±20° from the green boundary. This keeps the editor to 7-8 inputs instead of 24.

- **Log entry schema uses a flat array with a `type` field**: Both sample data and structured events (`set_start`, `rep_detected`, `set_end`, `threshold_changed`) live in the same `sessionLog` array, distinguished by `type`. Samples have `type: "sample"`; events have explicit types. This keeps the export format simple and preserves chronological ordering in one pass.

- **Complete logging computes all 6 angles independently of `computeAngle()`**: A separate `computeAllAnglesForLog(pose)` function runs every logged frame and returns raw values (never suppressed) for all 6 angle definitions, with the visibility score of each constituent landmark alongside the value.

## Open Questions

### Resolved During Planning

- **Debounce window**: 2 consecutive frames (~60ms at 30fps) — prevents single-frame threshold crossings from counting as reps without introducing perceptible delay.
- **Auto end-of-set duration**: 3 seconds at neutral position — long enough to distinguish end-of-set rest from brief pauses; short enough to not annoy if the user finishes a set and waits.
- **Threshold editor UI**: Number inputs with ±5° step buttons. Mobile-appropriate touch targets. Sliders rejected: too imprecise at small increments on mobile.
- **Log file size**: ~1.2 MB for a 10-min session — safe for Web Share API on iOS.

### Deferred to Implementation

- **Exact neutral threshold values** (knee > 155°, etc.) — these are planning estimates. Implementer should verify against observed angle values during standing between sets and adjust if needed.
- **Auto end-of-set angle stability criterion** — the plan uses "primary angle in neutral range continuously." If the user sways slightly, brief exits from the neutral band might restart the 3s timer. Implementer may need to add a small tolerance (e.g., allow up to 0.5s excursions before restarting the timer).
- **Summary stat computation** — min/avg/max should exclude frames where the primary angle was null (geometry failure). Implementer decides whether to filter nulls or show "—" when insufficient data.

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.*

**Session state machine:**

```
STATES: idle | active_set | in_rep | end_set_countdown | summary

idle
  → [Start Set button] → active_set
    emit: set_start event

active_set
  → [primary angle crosses rep threshold, 2+ frames] → in_rep
  → [End Set button] → summary (skip countdown)
  → [3s at neutral, ≥1 rep counted] → end_set_countdown

in_rep
  → [primary angle returns to neutral side of threshold] → active_set
    emit: rep_detected event (peak angle, quality classification)
    increment rep counter

end_set_countdown (3-2-1 overlay, cancellable)
  → [tap anywhere / cancel] → active_set
  → [countdown reaches 0] → summary

summary (full-screen takeover)
  emit: set_end event (rep count, stats)
  → [Next Set button] → active_set (increment set number, reset rep counter)
```

**Per-frame processing (onFrame):**

```
draw video frame to canvas
run poseLandmarker.detectForVideo()

if pose detected:
  if state is active_set or in_rep:
    run rep state machine (primary angle only)
    update neutral timer (for auto end-of-set)
  draw skeleton overlay
  update angle readout panel (for active exercise only)

  if logging interval elapsed:
    computeAllAnglesForLog(pose) → append sample to sessionLog
```

**Log entry shapes:**

```
sample:     { type: "sample", ts, exercise, setNum, repNum,
              angles: { squat_knee: {value, vis_A, vis_V, vis_C}, ... all 6 ... } }

set_start:  { type: "set_start", ts, setNum, exercise, thresholds: {...} }
rep_detect: { type: "rep_detected", ts, setNum, repNum, peakAngle, quality }
set_end:    { type: "set_end", ts, setNum, repCount, stats: {min, avg, max per angle} }
thresh_chg: { type: "threshold_changed", ts, angle, field, oldVal, newVal }
```

## Implementation Units

---

- [x] **Unit 1: stage4.html scaffold + complete logging overhaul**

**Goal:** Fork stage3.html into stage4.html, replace the logging system with complete angle capture (all 6 angles, no visibility suppression), and add set/rep context to every sample.

**Requirements:** R6, R7

**Dependencies:** None (starting point for all other units)

**Files:**
- Create: `stage4.html` (fork of `stage3.html`)
- Modify: `index.html` (add Stage 4 link)

**Approach:**
- Copy stage3.html verbatim as stage4.html. All existing functionality (camera, skeleton, angle display, exercise selector, threshold color coding, download) must continue to work before adding anything new.
- Add session state variables: `setNumber = 0`, `repNumber = 0` (current rep within set, increments during active set).
- Add `computeAllAnglesForLog(pose)` — iterates over all exercises and all angle definitions, computes each angle using the same `angleDeg` / `backAngleDeg` math, but always returns the value (no visibility suppression). Returns an object keyed by `"${exercise}_${label.toLowerCase()}"` (e.g., `"squat_knee"`), each value being `{ value: number|null, visA: number, visV: number, visC: number }`. Back angle uses midpoint visibility (avg of 4 landmarks).
- Replace `maybeLog(exercise, computed)` with a new function that calls `computeAllAnglesForLog(pose)` and appends a `type: "sample"` entry with `setNum` and `repNum` alongside the full angles object.
- Update `downloadLog` to omit the "no data" guard message change but otherwise works the same.
- Add `pushEvent(event)` helper that appends any typed event to `sessionLog` — used by later units.
- Update `index.html` to add a Stage 4 card linking to `stage4.html`.

**Patterns to follow:**
- `maybeLog` in stage3.html:330 — same 5fps throttle pattern, replace the body
- `computeAngle` in stage3.html:405 — same angle math, generalized across all exercises
- `EXERCISES` config structure — iterate `Object.entries(EXERCISES)` to get all angle defs

**Test scenarios:**
- Download the log after 30 seconds of movement; verify JSON contains `type: "sample"` entries with all 6 angle keys (squat_knee, squat_hip, deadlift_hip, deadlift_back, ohp_elbow, ohp_shld)
- Deliberately occlude one knee; verify the corresponding angle value is still present in the log (not null unless geometry failed) with a low visibility score
- Switch exercises mid-session; verify log entries continue to capture all 6 angles regardless of which exercise is active

**Verification:**
- Exported JSON has no sample entries missing any of the 6 angle keys
- No angle value is `undefined` — only `null` when the vector math fails, never when visibility is low
- `setNum` and `repNum` are present on every sample (both are `0` until set tracking is wired in Unit 2)

---

- [x] **Unit 2: Rep state machine + live counter UI**

**Goal:** Detect reps automatically using the primary angle for the active exercise. Display a live counter. Provide `+` / `−` correction buttons and a reset action.

**Requirements:** R1, R2

**Dependencies:** Unit 1 (stage4.html scaffold)

**Files:**
- Modify: `stage4.html`

**Approach:**

**State variables to add:**
- `inRep: boolean` — whether the primary angle is currently past the rep threshold
- `repPeakAngle: number|null` — best angle achieved during the current rep (tracked while `inRep`)
- `framesBelowThreshold: number` — debounce counter; rep transitions only after this reaches 2
- `repCount: number` — current set's rep count (manual + auto)
- `neutralFrameCount: number` — frames spent in neutral zone (for auto end-of-set in Unit 3)

**Per-exercise rep configuration** — add a `repPrimary` field to the `EXERCISES` config:
- `squat.repPrimary`: uses knee angle definition, trigger = `THRESHOLDS.squat.knee.yellow[1]`, direction = "lower" (angle must go below trigger to count)
- `deadlift.repPrimary`: uses hip angle definition, trigger = `THRESHOLDS.deadlift.hip.yellow[1]`, direction = "lower"
- `ohp.repPrimary`: uses elbow angle definition, trigger = `THRESHOLDS.ohp.elbow.yellow[0]`, direction = "higher"

**Rep state machine** runs inside `onFrame()` after pose is detected:
- Compute primary angle value for current exercise (reuse `computeAngle` result already computed for display)
- If direction = "lower": `isPastThreshold = (value !== null && value < trigger)`; if direction = "higher": `isPastThreshold = (value !== null && value > trigger)`
- When `!inRep && isPastThreshold`: increment `framesBelowThreshold`; if `>= 2`, set `inRep = true`, initialize `repPeakAngle`
- When `inRep && isPastThreshold`: update `repPeakAngle` (track deepest/highest point)
- When `inRep && !isPastThreshold`: count the rep, emit `rep_detected` event via `pushEvent`, increment `repCount`, set `inRep = false`, reset `framesBelowThreshold = 0`
- When `!inRep && !isPastThreshold`: reset `framesBelowThreshold = 0`

**UI additions:**
- Replace the small angle panel with a two-zone layout: top zone = live angle readouts (smaller, existing behavior), bottom zone = set tracker (rep counter + controls)
- Set tracker zone: `SET 1 · REPS: [−] [8] [+]` — large font, readable from 4 ft
- Below that: `[End Set]` button (prominent; implemented in Unit 3) and `[Reset]` link (resets repCount to 0, does not emit set_end)
- Increment/decrement buttons directly update `repCount` (no min/max enforcement other than `max(0, repCount - 1)` for decrement)

**Technical design:**

```
// Rep state machine (directional guidance, not code to reproduce):
repDetectionTick(primaryAngleValue):
  isPast = isPastThreshold(primaryAngleValue, repConfig)

  if not inRep:
    if isPast:  framesBelowThreshold++
    else:       framesBelowThreshold = 0

    if framesBelowThreshold >= 2:
      inRep = true
      repPeakAngle = primaryAngleValue

  else:  // inRep = true
    if isPast:
      repPeakAngle = updatePeak(repPeakAngle, primaryAngleValue, repConfig.direction)
    else:
      // Crossed back — rep complete
      quality = classify(repPeakAngle, thresholdRuleForPrimary)
      pushEvent({ type: "rep_detected", repNum: ++repCount, peakAngle: repPeakAngle, quality })
      updateRepCounterUI(repCount)
      inRep = false
      framesBelowThreshold = 0
      repPeakAngle = null
```

**Patterns to follow:**
- `onFrame()` in stage4.html (from Unit 1) — add rep machine call after pose detection, before `maybeLog`
- `classify()` in stage3.html:257 — reuse directly for rep quality

**Test scenarios:**
- Do 5 clean squats at good depth; verify counter reaches 5 without tapping correction buttons
- Pause at the bottom of a squat for 2 seconds; verify only 1 rep is counted when returning to standing (not multiple)
- Do a very shallow squat that stays in the red range; verify it is NOT counted as a rep
- Tap `−` after an erroneously counted rep; verify counter decrements and does not go below 0
- Switch exercises mid-set; verify rep counter resets to 0 on exercise switch

**Verification:**
- Rep counter increments once per full rep cycle (down past threshold, back up through threshold)
- Shallow reps (red range only) produce no count increment
- `rep_detected` events appear in the session log JSON with correct `repNum` and `quality` fields

---

- [x] **Unit 3: End Set trigger + auto-detect countdown**

**Goal:** Add the "End Set" button and an auto-detect countdown that triggers when the user has been at neutral for 3 seconds after completing reps.

**Requirements:** R3

**Dependencies:** Unit 2 (rep state machine + rep counter)

**Files:**
- Modify: `stage4.html`

**Approach:**

**Manual "End Set" button:** Already in the UI from Unit 2. On tap: transition to summary state (Unit 4 wires this up), emit `set_end` event via `pushEvent`.

**Auto end-of-set detection — neutral timer:**
- "Neutral" for current exercise: squat/deadlift = primary angle > 155°; OHP = elbow angle < 120°. Store as `NEUTRAL_THRESHOLD` per exercise in the `EXERCISES` config.
- In `onFrame()`, after rep machine runs: if state is `active_set` (not `inRep`) AND primaryAngle is in neutral range AND `repCount >= 1`: increment `neutralFrameCount`; else reset to 0.
- When `neutralFrameCount >= 90` (3 seconds at 30fps — implementer to calibrate against actual rVFC cadence): show the auto-end countdown overlay.

**Countdown overlay:**
- Semi-transparent full-screen overlay on top of the live camera + skeleton
- Large centered text: "Set ending in 3…" → "2…" → "1…"
- Underneath: "Tap to cancel"
- Tapping anywhere on the overlay resets `neutralFrameCount = 0` and hides the overlay
- After countdown reaches 0: transition to summary state

**State variable to add:** `countdown: number | null` (null = no countdown, otherwise 3/2/1)

**Patterns to follow:**
- Loading overlay pattern from stage3.html (position: fixed, inset: 0, z-index layering) for the countdown overlay
- `loadingOverlay.classList.add('hidden')` pattern for show/hide

**Test scenarios:**
- Complete 3 reps, then stand still; verify countdown appears after ~3 seconds
- Tap anywhere during countdown; verify it cancels and live tracking resumes
- Press "End Set" manually before countdown; verify summary appears immediately without waiting
- Set repCount to 0 (no reps done); verify that standing in neutral does NOT trigger the countdown
- Switch exercises while countdown is showing; verify countdown cancels

**Verification:**
- Countdown overlay appears within 3–4 seconds of returning to neutral after completing reps
- Cancellation via tap always works and does not leave the app in a stuck state
- Manual "End Set" always triggers summary immediately regardless of countdown state

---

- [x] **Unit 4: Between-set summary screen**

**Goal:** Full-screen summary takeover showing rep count, per-angle stats (min/avg/max), and rep quality breakdown (green/yellow/red counts). "Next Set" returns to live tracking.

**Requirements:** R4, R7 (set_end event)

**Dependencies:** Unit 3 (End Set trigger wires into this screen)

**Files:**
- Modify: `stage4.html`

**Approach:**

**Summary screen UI structure (full-screen, replaces live view):**
```
SET 2 COMPLETE                    [large header]
─────────────────────────────────
REPS: 8

FORM BREAKDOWN
  Knee    min 72°   avg 85°   max 102°
          ██████░░  (6 green · 2 yellow)

  Hip     min 88°   avg 96°   max 114°
          ████░░░░  (4 green · 3 yellow · 1 red)

─────────────────────────────────
[          Next Set          ]   [large button]
```

**Stat computation:**
- Collect all `type: "sample"` log entries where `setNum === currentSetNum`
- Per primary angle: filter out nulls, compute min/avg/max
- Rep quality: from `type: "rep_detected"` events for the current set, count quality field values

**State transitions:**
- On entering summary: `running = false` (pause inference loop), hide live UI, show summary screen
- Emit `set_end` event: `{ type: "set_end", ts, setNum, repCount (corrected), stats }`
- On "Next Set": increment `setNumber`, reset `repCount = 0`, `inRep = false`, `neutralFrameCount = 0`, resume inference loop (`running = true`, `scheduleFrame()`), hide summary, show live UI

**Show only the two primary angles for the active exercise** in the summary (not all 6). The summary is readable at a glance, not a data dump.

**Patterns to follow:**
- Loading overlay CSS pattern (position fixed, inset 0, flex column) for the summary screen — same layering approach but without the hidden/visible toggling for the loading spinner
- `updateAnglePanelFast` DOM update pattern for summary stats (minimize layout thrash by pre-rendering to a fixed DOM structure)

**Test scenarios:**
- Complete a set of 5 reps; tap "End Set"; verify summary shows correct rep count and angle stats
- Verify min angle in summary is the deepest point observed during any rep (for squat: smallest knee angle logged during the set)
- Tap "Next Set"; verify set counter increments, rep counter resets to 0, and camera feed resumes
- Complete two consecutive sets; verify the second set's summary does not include data from the first set
- Empty set (0 reps): tap "End Set"; verify summary shows 0 reps and "—" for stats that have no data

**Verification:**
- Summary screen appears within 1 second of "End Set"
- Rep quality breakdown counts sum to the displayed rep count
- "Next Set" consistently returns to a functional live tracking state
- `set_end` event appears in exported JSON with correct `setNum` and `repCount`

---

- [x] **Unit 5: Threshold editor + localStorage persistence**

**Goal:** Settings panel where each angle's key threshold can be adjusted with ±5° buttons. Changes persist to localStorage and take effect immediately in live color feedback.

**Requirements:** R5

**Dependencies:** Unit 1 (stage4.html scaffold, THRESHOLDS must be mutable not const)

**Files:**
- Modify: `stage4.html`

**Approach:**

**Make THRESHOLDS mutable:**
- Change `const THRESHOLDS` to `let THRESHOLDS` and move the default values to a `DEFAULT_THRESHOLDS` constant.
- On page load: attempt `JSON.parse(localStorage.getItem('form-tracker-thresholds'))`. If valid, merge into `THRESHOLDS`; otherwise use defaults.
- On any threshold change: `localStorage.setItem('form-tracker-thresholds', JSON.stringify(THRESHOLDS))`.

**Settings button:**
- A gear icon `⚙` button in the status bar (alongside flip and download buttons).
- Tapping it shows the settings panel (slide up from bottom or full-screen overlay — implementer's choice, but must be dismissible without changing thresholds).

**Settings panel layout:**
- Header: "Thresholds" + "Reset to Defaults" link + close button
- Per exercise section (Squat / Deadlift / OHP):
  - Per key threshold: label + `[−5]  [value]  [+5]` with large touch targets (min 44px)
  - Exposed thresholds:
    - Squat Knee depth: `THRESHOLDS.squat.knee.green[1]` (label: "Knee target depth ≤ ___°")
    - Squat Hip depth: `THRESHOLDS.squat.hip.green[1]`
    - Deadlift Hip hinge: `THRESHOLDS.deadlift.hip.green[1]`
    - Deadlift Back min: `THRESHOLDS.deadlift.back.green[0]` (label: "Spine min ___°")
    - Deadlift Back max: `THRESHOLDS.deadlift.back.green[1]` (label: "Spine max ___°")
    - OHP Elbow lockout: `THRESHOLDS.ohp.elbow.green[0]` (label: "Elbow lockout ≥ ___°")
    - OHP Shoulder elevation: `THRESHOLDS.ohp.shld.green[0]`
  - Yellow range is auto-computed as ±20° from the green boundary (implementer to apply consistently)

**On ±5° tap:**
1. Update the `THRESHOLDS` object in memory
2. Update the display value in the settings panel
3. Persist to localStorage
4. Emit `threshold_changed` event via `pushEvent`
5. The live color feedback picks up the change on the next frame (classify() always reads from the current `THRESHOLDS` object)

**"Reset to Defaults" link:** restores `THRESHOLDS` from `DEFAULT_THRESHOLDS`, clears localStorage key, re-renders all inputs, emits a threshold_changed event for each changed value.

**Patterns to follow:**
- Flip button and download button patterns in stage4.html status bar for the gear icon placement
- Loading overlay CSS for the settings panel (position fixed, z-index layering)
- `classify()` already references `THRESHOLDS` by reference — making it mutable is sufficient for immediate effect

**Test scenarios:**
- Open settings panel; increase squat knee green threshold by 10°; close panel; do a squat that was previously yellow; verify it now shows green
- Refresh the page; open settings panel; verify the customized threshold value is still shown (localStorage persistence)
- Tap "Reset to Defaults"; verify thresholds return to original values and localStorage is cleared
- Verify that changing thresholds also changes the rep detection trigger (since rep trigger reads from THRESHOLDS)
- Open settings during an active set; verify live angle colors update immediately when a threshold is changed without closing the panel

**Verification:**
- All 7 threshold values are adjustable and immediately reflected in live feedback
- Custom thresholds survive page refresh
- `threshold_changed` events appear in the exported JSON log
- Reset to Defaults removes localStorage entry and restores all hardcoded defaults

---

## System-Wide Impact

- **Inference loop (onFrame):** Gains rep state machine execution and neutral timer update on every frame. The additional computation is O(1) — a few comparisons — and will not affect frame rate.
- **THRESHOLDS mutability:** Changing `const` to `let` means no other code can rely on THRESHOLDS being frozen. `classify()` and `computeAngle()` already read THRESHOLDS by reference; no other callers.
- **Session state resets on exercise switch:** `repCount`, `inRep`, `neutralFrameCount` should reset when the user changes exercise mid-session. This is a behavioral choice that the implementer must explicitly handle in `selectExercise()`.
- **Summary screen pauses inference (`running = false`):** The canvas freezes on the last frame. Skeleton and FPS counter stop updating. This is intentional — the user is reading the summary, not doing reps. Resuming (`running = true`) must call `scheduleFrame()` once to restart the rVFC/rAF chain.
- **Log growth:** With all 6 angles + visibility scores per sample, each `"sample"` entry is ~400–500 bytes. At 5fps for 20 minutes (longer session): ~3000 entries × 450 bytes ≈ 1.35 MB. Still within iOS Web Share limit. No mitigation needed for sessions under 30 minutes.

## Risks & Dependencies

- **Rep detection false positives on fast movements:** The 2-frame debounce (~60ms at 30fps) may not be sufficient for very rapid reps. If testing shows double-counting, increase to 3–4 frames. Mitigated by the `+/-` correction buttons.
- **Neutral threshold calibration:** The 155° "standing neutral" values are planning estimates. If users have limited range of motion and never reach 155° even when standing between reps, the auto end-of-set will never trigger. Implementer must verify against real observed values during testing and expose these as constants for easy tuning.
- **rVFC cadence vs frame-based timing:** The neutral timer uses frame counts (90 frames ≈ 3s at 30fps). If `requestVideoFrameCallback` fires at a different rate (e.g., 24fps on some devices), the 3-second timeout will be longer. Implementer may prefer `performance.now()`-based timing instead of frame counts for the neutral timer.
- **Summary stat accuracy depends on log timing:** If the logging interval (200ms / 5fps) misses the exact moment of deepest squat, the logged min may not reflect the true peak. Rep peak tracking in the state machine (Unit 2) is separate from the log and runs every frame — use `rep_detected.peakAngle` for rep quality, not derived from log samples.
- **stage3.html has no tests:** There is no automated test suite. All verification is manual, phone-based. The plan's "Test scenarios" are manual test scripts.

## Sources & References

- **Origin document:** [docs/brainstorms/2026-03-28-001-set-level-feedback-requirements.md](../brainstorms/2026-03-28-001-set-level-feedback-requirements.md)
- Base implementation: `stage3.html` (all patterns carried forward)
- Prior plan: [docs/plans/2026-03-27-001-feat-exercise-form-tracker-poc-plan.md](2026-03-27-001-feat-exercise-form-tracker-poc-plan.md)
