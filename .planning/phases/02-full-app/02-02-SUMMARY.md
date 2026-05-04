---
phase: 02-full-app
plan: 02
subsystem: ui
tags: [html, css, vanilla-js, clipboard, dark-mode, epc-qr]

# Dependency graph
requires:
  - phase: 02-full-app
    plan: 01
    provides: Full form UI with QR generation, initUI IIFE, qr-output element
provides:
  - Copy-to-clipboard button with SVG-to-canvas-to-PNG pipeline
  - OS-aware dark/light mode via prefers-color-scheme media query
  - Task 2 pending human verification: QR scan in Dutch banking app
affects: [phase-03-wero-qr]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "SVG-to-canvas-to-PNG clipboard pipeline: XMLSerializer -> Blob URL -> Image -> Canvas -> toBlob -> ClipboardItem"
    - "Dark mode via @media (prefers-color-scheme: dark) with CSS custom properties override"
    - "QR area stays white in dark mode — critical for QR scan reliability"
    - "Copy feedback via opacity transition (CSS .visible class, setTimeout to dismiss)"

key-files:
  created: []
  modified:
    - index.html

key-decisions:
  - "SVG-to-canvas-to-PNG pipeline required because navigator.clipboard.write only accepts PNG blobs, not SVG"
  - "Canvas renders at 400x400 for good image quality when pasted into other apps"
  - "Dark mode keeps .qr-output background white (#ffffff) — QR codes require white background for scanning reliability"
  - "Clipboard API requires secure context (HTTPS or localhost); catch handler shows Dutch fallback message for file:// users"

requirements-completed:
  - QRCD-04
  - QRCD-05
  - GENL-02

# Metrics
duration: 8min
completed: 2026-03-23
---

# Phase 2 Plan 02: Copy-to-Clipboard and Dark Mode Summary

**Copy button with SVG-to-canvas-to-PNG clipboard pipeline and OS-aware dark/light mode via prefers-color-scheme; Task 2 (QR scan verification) pending human approval**

## Performance

- **Duration:** 8 min
- **Started:** 2026-03-23
- **Completed:** 2026-03-23
- **Tasks:** 1 of 2 complete (Task 2 is a human-verify checkpoint)
- **Files modified:** 1

## Accomplishments

- Added `#btn-copy` button and `#copy-feedback` span to the QR actions area
- Implemented SVG-to-canvas-to-PNG clipboard pipeline using XMLSerializer, Blob URLs, Canvas 2D API, and `navigator.clipboard.write`
- White background explicitly drawn on canvas before SVG overlay (scanning reliability)
- Dutch-language feedback: "Gekopieerd!" on success, "Kopiëren mislukt — probeer rechtermuisklik" on failure
- Added `.btn-secondary` and `.copy-feedback` CSS classes with opacity transition feedback
- Added `@media (prefers-color-scheme: dark)` block covering all UI elements: body, subtitle, inputs, field states, buttons, QR area, char-count
- `.qr-output` background stays `#ffffff` in dark mode to ensure QR code remains scannable

## Task Commits

No git repository — changes applied directly to index.html.

1. **Task 1: Add copy-to-clipboard button and dark/light mode CSS** - index.html updated with copy button HTML, copy button CSS, dark mode CSS, and copy-to-clipboard JS

## Files Created/Modified

- `index.html` - Copy button HTML, CSS for `.btn-secondary` and `.copy-feedback`, `@media (prefers-color-scheme: dark)` block, and copy-to-clipboard IIFE extension

## Decisions Made

- SVG-to-canvas-to-PNG pipeline: `navigator.clipboard.write` only accepts `image/png` ClipboardItems, not SVG blobs directly
- Canvas renders at 400x400px for quality output when pasted into documents or image editors
- `.qr-output` stays white in dark mode — QR scanner cameras need high contrast black-on-white
- Catch handler on clipboard API failure shows Dutch fallback message (file:// users won't have Clipboard API access)

## Deviations from Plan

None — Task 1 executed exactly as specified. All acceptance criteria verified.

## Pending Human Verification

**Task 2: Verify QR code scans in Dutch banking app**
- Status: PENDING — requires physical device verification
- Type: checkpoint:human-verify (blocking gate)
- What to verify:
  1. Open index.html (use `python -m http.server 8000` for clipboard API support)
  2. Paste: "Factuur 2024-001 t.n.v. Test B.V. IBAN: NL91 ABNA 0417 1643 00 Totaal: EUR 25,00"
  3. Verify IBAN, name, and amount fields auto-populate with green status indicators
  4. Click "Genereer QR Code" — QR code appears at least 200x200px
  5. Scan with Bunq/ING/Rabobank/ABN AMRO banking app
  6. Verify: IBAN NL91ABNA0417164300, name "Test B.V.", amount EUR 25.00
  7. Click "Kopieer QR naar klembord", paste into an image-accepting app
  8. Toggle OS dark mode — verify app adapts, QR area stays white
- Resume signal: Type "approved" if QR scans correctly, or describe any issues

## Issues Encountered

None.

## User Setup Required

For clipboard API: serve via `python -m http.server 8000` (or any local server) — `navigator.clipboard.write` requires a secure context (HTTPS or localhost). The app degrades gracefully on file:// with a Dutch error message.

## Known Stubs

None — all UI elements are fully wired. Copy button requires a QR code to be present (button does nothing if no SVG is in `#qr-output`, which is intentional).

## Next Phase Readiness

- After Task 2 human verification is approved: Phase 2 is complete, Phase 3 (Wero QR) can begin
- All Phase 2 requirements (QRCD-04, QRCD-05, GENL-02) marked complete in REQUIREMENTS.md

---
*Phase: 02-full-app*
*Completed: 2026-03-23*

## Self-Check

- `index.html` contains `id="btn-copy"`: VERIFIED (line 250)
- `index.html` contains `id="copy-feedback"`: VERIFIED
- `index.html` contains `navigator.clipboard.write`: VERIFIED
- `index.html` contains `new ClipboardItem`: VERIFIED
- `index.html` contains `canvas.toBlob`: VERIFIED
- `index.html` contains `new XMLSerializer().serializeToString`: VERIFIED
- `index.html` contains `ctx.fillStyle = '#ffffff'`: VERIFIED
- `index.html` contains `canvas.width = 400`: VERIFIED (line 810)
- `index.html` contains `Gekopieerd!`: VERIFIED (line 828)
- `index.html` contains `@media (prefers-color-scheme: dark)`: VERIFIED (line 149)
- Dark mode body background `#1a1a1a`: VERIFIED (line 151)
- Dark mode body color `#e0e0e0`: VERIFIED
- Dark mode `.qr-output` background `#ffffff`: VERIFIED (line 188)
- Dark mode `.btn-primary` background `#4da3ff`: VERIFIED (line 174)
- Dark mode `textarea, input[type="text"]` background `#2a2a2a`: VERIFIED
- `.btn-secondary` CSS class present: VERIFIED
- `.copy-feedback` CSS with `opacity` transition: VERIFIED
- All Phase 1 functions unmodified: VERIFIED (only initUI IIFE extended)
- No git repo — commit hashes not applicable

## Self-Check: PASSED
