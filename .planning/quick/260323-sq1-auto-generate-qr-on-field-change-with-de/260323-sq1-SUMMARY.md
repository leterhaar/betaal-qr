---
phase: quick
plan: 260323-sq1
subsystem: ui
tags: [auto-generate, debounce, animation, ux]
dependency_graph:
  requires: []
  provides: [auto-generate-qr, qr-refresh-animation]
  affects: [index.html]
tech_stack:
  added: []
  patterns: [debounced-input-listener, css-animation-reflow-trick]
key_files:
  created: []
  modified:
    - index.html
decisions:
  - 600ms debounce chosen as balance between responsiveness and keystroke noise
  - Generate button hidden (not removed) to keep click-path code intact as fallback
  - void offsetWidth reflow trick used to force CSS animation restart on class re-add
metrics:
  duration: ~10min
  completed: "2026-03-23"
  tasks_completed: 1
  files_modified: 1
---

# Quick Task 260323-sq1: Auto-generate QR on field change with debounce

**One-liner:** 600ms debounced auto-generation on all form fields + CSS opacity/scale pulse to signal QR updates, hiding the Generate button for a zero-click paste-to-QR workflow.

## What Was Built

Added debounced auto-generate logic so the QR code appears automatically after a natural typing pause — no button press required. A subtle CSS animation (opacity fade + slight scale) signals each QR refresh.

### Changes Made to `index.html`

**1. CSS — `@keyframes qr-refresh` + `.qr-refreshing` class** (after `.copy-feedback.visible`)
- Opacity fades to 0.55 and scale drops to 0.97 at 30%, then returns to 1 — brief but perceptible
- Applied and re-applied via a `void offsetWidth` reflow trick to restart the animation on each update

**2. JS — `autoGenerate()` function** inside `initUI()`, after `updateGenerateButton` definition
- Clears and resets a `autoGenerateTimer` on every call (debounce)
- Fires after 600ms: checks `validateIBAN(ibanVal) && nameVal` before proceeding
- Silently swallows errors — the Generate button click-path still shows `alert()` for explicit errors

**3. JS — wired `autoGenerate()` into field listeners**
- `fieldIBAN` input: after `updateGenerateButton()`
- `fieldName` input: after `updateGenerateButton()`
- `fieldAmount` input: new listener added (`updateGenerateButton` + `autoGenerate`)
- `fieldRemittance` input: after counter update
- `fieldBIC` input: new listener added
- `handlePaste()`: at end, after `updateGenerateButton()`

**4. HTML — Generate button hidden**
- Added `hidden` class to `<button id="btn-generate">` — button and its click listener remain fully intact

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None.

## Self-Check

- [x] `index.html` modified with all four changes
- [x] CSS `@keyframes qr-refresh` and `.qr-refreshing` present after `.copy-feedback.visible`
- [x] `autoGenerate()` function defined inside `initUI()` scope with 600ms debounce
- [x] All five field listeners wired (`fieldIBAN`, `fieldName`, `fieldAmount`, `fieldRemittance`, `fieldBIC`)
- [x] `handlePaste()` calls `autoGenerate()` after `updateGenerateButton()`
- [x] Generate button has `class="btn-primary hidden"`
- [x] No existing extraction, validation, QR rendering, or clipboard logic changed

## Self-Check: PASSED
