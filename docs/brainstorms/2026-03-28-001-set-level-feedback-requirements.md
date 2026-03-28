---
date: 2026-03-28
topic: set-level-feedback
---

# Set-Level Feedback, Rep Tracking, and Threshold Configuration

## Problem Frame

POC testing validated that MediaPipe Pose works reliably at 3–4 ft on mobile. Three product problems surfaced:

1. **Feedback timing is wrong.** Real-time angle colors during a rep aren't useful — the athlete is focused on the movement. The valuable moment is *between sets*, when they can process what happened and adjust.
2. **Logging is incomplete.** The JSON export suppresses angles below the visibility threshold and only logs the currently selected exercise, hiding data that's needed to understand model behavior and calibrate thresholds.
3. **Thresholds aren't personalizable.** Appropriate angle ranges vary by individual physiology and trainer judgment. Hardcoded constants require a code edit to change.

A fourth problem — video annotation for ground-truth labeling — is real but deferred. In-session rep markers (below) provide labeled angle data without video infrastructure.

## Requirements

- **R1. Automatic rep detection** — During an active set, detect each rep by tracking angle transitions through a configurable depth threshold for the active exercise:
  - Squat: rep counted when knee angle descends below the green/yellow boundary and returns above it
  - Deadlift: rep counted when hip angle descends and returns
  - OHP: rep counted when elbow angle ascends above the green/yellow boundary and returns
  - Display a live rep counter prominently during the set

- **R2. Rep count correction** — Prominent `+` and `−` buttons allow the user to correct the rep counter at any time during or after the set. Correcting to 0 or pressing a "Reset" action clears the set without triggering a summary.

- **R3. End Set trigger** — A prominent "End Set" button triggers the between-set summary. Optionally: auto-trigger after a sustained pause in movement (no primary angle change > 10° for 3+ seconds), with the auto-trigger cancellable.

- **R4. Between-set summary** — After "End Set", a summary overlay appears showing:
  - Rep count (after any correction)
  - Per primary-angle stats: min / avg / max across the set
  - Rep quality breakdown: count of reps that peaked in green, yellow, red range for each primary angle
  - Displayed as a **full-screen takeover** replacing the live view; a "Next Set" button returns to the camera

- **R5. Threshold editor** — A settings panel accessible during a session that allows per-angle threshold adjustment. Changes take effect immediately in the live color feedback. Settings are persisted to `localStorage` and survive page refresh. Falls back to hardcoded defaults if no saved settings exist.

- **R6. Complete logging** — The session log captures all six angle values (squat + deadlift + OHP) at every sample interval, regardless of which exercise is selected or whether landmarks are visible. Each entry includes:
  - Timestamp
  - Currently selected exercise
  - All six angle values (null if computation failed due to geometry, not suppressed for low visibility)
  - Visibility score for each constituent landmark alongside its angle value
  - Set and rep context (set number, rep number at time of sample)

- **R7. Set structure in log** — The log includes structured events alongside sample data:
  - `set_start`: timestamp, set number, active exercise, threshold settings in use
  - `rep_detected`: timestamp, set number, rep number, peak angle values during that rep, classification (green/yellow/red)
  - `set_end`: timestamp, set number, corrected rep count, summary stats
  - `threshold_changed`: timestamp, which threshold changed, old value, new value

## Success Criteria

- Auto rep counter is within ±1 rep on 80%+ of clean sets (no assisted correction needed for most sets)
- Threshold editor changes are reflected in the live feedback within one frame
- Set summary appears within 1 second of "End Set"
- Downloaded JSON contains all six angle values per sample regardless of selected exercise and regardless of visibility score
- A 10-minute session log with complete angle data and set structure is small enough to share via Web Share API on iOS

## Scope Boundaries

- No backend — all persistence in `localStorage` and in-memory session log
- No automatic exercise detection — manual selection only
- No video recording or playback
- One threshold profile per device (no named profiles or multi-client support)
- No audio/haptic cues
- Rep quality classification uses the peak angle of each rep (e.g., deepest knee angle during a squat), not the average
- Auto end-of-set detection is best-effort; the manual "End Set" button is always the primary trigger

## Key Decisions

- **Between-set (not real-time) as the primary feedback moment**: Real-time color changes during a rep are noise. The summary after a set is actionable.
- **In-session rep markers replace video annotation for now**: Labeled angle data (rep X of set Y, classified as green/yellow/red) is sufficient for threshold calibration. Video review is deferred to the backend phase.
- **Log everything, filter nothing**: The export captures all angles with visibility metadata. Suppression is a display concern, not a logging concern.
- **localStorage for threshold profiles**: Enough for the trainer-on-device use case. A backend profile system belongs to the next phase when trainer-client relationships exist.
- **Hybrid rep detection (auto + manual correction)**: Fully automatic is unreliable; fully manual is tedious. Auto-detect with +/- correction is the right balance for this phase.

## Dependencies / Assumptions

- Builds on top of the existing Stage 3 HTML file; no new deployment infrastructure needed
- Rep detection relies on the same angle computation already running per frame; no additional ML required
- Trainer configures thresholds directly on the client's device before a session
- The threshold editor is a developer/trainer tool in this phase, not end-user-facing

## Outstanding Questions

### Resolve Before Planning

_(none — all blocking questions resolved)_

### Deferred to Planning

- [Affects R1][Technical] What debounce window prevents double-counting a rep (e.g., brief angle reversal at the bottom of a squat)?
- [Affects R3][Technical] What "sustained pause" threshold (duration + angle stability) reliably distinguishes end-of-set from momentary rest at the bottom?
- [Affects R5][Technical] What is the simplest UI for the threshold editor — sliders, number inputs, or a range picker? Evaluate against mobile touch targets.
- [Affects R6][Needs research] Will a 10-min session log with 6 angles + visibility scores at 5 fps exceed iOS Web Share API file size limits?

## Next Steps

→ `/ce:plan` for structured implementation planning
