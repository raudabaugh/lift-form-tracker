---
title: Remove set-end autodetection, rely on manual End Set only
type: fix
status: completed
date: 2026-03-31
---

# Remove Set-End Autodetection

The neutral-timer → countdown pipeline that auto-ends sets is not reliable enough. Remove it entirely and rely solely on the manual "End Set" button.

## Problem Statement

The autodetection path (`updateNeutralTimer` → `startCountdown` → `triggerEndSet`) triggers false positives when the user pauses mid-set at a neutral joint angle. The 3-second countdown is disruptive and the cancel interaction adds friction. The manual "End Set" button already covers the use case cleanly.

## Proposed Solution

Delete all code related to autodetection. The state machine simplifies from four states (`idle | active_set | countdown | summary`) to three (`idle | active_set | summary`). The `endSet()` → `triggerEndSet()` path is unchanged and remains the sole way to end a set.

## Acceptance Criteria

- [ ] Holding a neutral position during a set never triggers a countdown or auto-ends the set
- [ ] "End Set" button still transitions from `active_set` → `summary` correctly
- [ ] `triggerEndSet()` does not throw any errors (no null DOM references)
- [ ] `appState` never reaches `'countdown'`
- [ ] No orphaned variable declarations or dead conditional branches reference `'countdown'`

## Implementation Checklist

All changes are in `stage4.html`.

### Variables to remove (lines ~784–786)

```js
// Remove all three:
let neutralTimerStart = null;
let countdownValue = 3;
let countdownTimer = null;
```

### Functions to remove entirely

- `updateNeutralTimer(primaryValue)` — lines 995–1008
- `startCountdown()` — lines 1010–1026
- `window.cancelCountdown` — lines 1028–1033

Remove the section comment above them ("Neutral timer + auto end-of-set countdown") as well.

### Render loop changes (lines ~1540–1565)

1. Remove the call to `updateNeutralTimer(primaryValue)` (line ~1546)
2. Narrow the outer condition from `appState === 'active_set' || appState === 'countdown'` → `appState === 'active_set'` (line ~1542)
3. Remove the no-pose block that resets `neutralTimerStart` and calls `cancelCountdown()` (lines ~1560–1562)

### `endSet()` cleanup (lines ~1066–1070)

Remove `cancelCountdown()` call and tighten the guard:

```js
// Before:
window.endSet = function () {
  if (appState === 'countdown') cancelCountdown();
  if (appState !== 'active_set' && appState !== 'countdown') return;
  triggerEndSet();
};

// After:
window.endSet = function () {
  if (appState !== 'active_set') return;
  triggerEndSet();
};
```

### `triggerEndSet()` cleanup (lines ~1088–1089) ⚠️ Critical

`triggerEndSet()` contains two countdown-related lines that reference the now-deleted DOM element. Without removing them, every manual "End Set" press throws `TypeError: Cannot read properties of null` and the app gets stuck in `active_set`:

```js
// Remove these two lines from inside triggerEndSet():
document.getElementById('countdown-overlay').classList.add('hidden');
if (countdownTimer) { clearInterval(countdownTimer); countdownTimer = null; }
```

### `resetCurrentSet()` cleanup (~line 1073)

Remove the `cancelCountdown()` call inside `resetCurrentSet()`.

### `resetRepState()` cleanup (~line 1042)

Remove `neutralTimerStart = null;` — the variable no longer exists.

### `selectExercise()` cleanup (~line 1290–1291)

Remove `cancelCountdown()` call and narrow its outer condition from `|| appState === 'countdown'` to just `appState === 'active_set'`.

### HTML to remove (lines ~535–540)

```html
<!-- Remove entire block: -->
<div id="countdown-overlay" class="hidden" onclick="cancelCountdown()">
  <div id="countdown-number">3</div>
  <div class="countdown-label">seconds until set ends</div>
  <div class="countdown-cancel">Tap to cancel</div>
</div>
```

### CSS to remove (lines ~277–293)

Remove all rules for: `#countdown-overlay`, `#countdown-overlay.hidden`, `.countdown-number`, `.countdown-label`, `.countdown-cancel`.

### Comment cleanup

Update the `appState` declaration comment from:
```js
// 'idle' | 'active_set' | 'countdown' | 'summary'
```
to:
```js
// 'idle' | 'active_set' | 'summary'
```

## Sources

- Implementation: `stage4.html` (single file, all changes)
- Related fix commit: `b7810bc` (8 state machine bugs, autodetection edge cases)
- Related fix commit: `783960a` (OHP neutral return requirement)
