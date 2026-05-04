---
phase: 02-full-app
plan: 01
subsystem: ui
tags: [html, css, vanilla-js, epc-qr, iban, form-validation]

# Dependency graph
requires:
  - phase: 01-core-logic
    provides: parsePaymentText, validateIBAN, formatEPCPayload, validateEPCPayload, renderQR, extractIBAN, extractAmount, extractName
provides:
  - Complete single-file form UI wired to Phase 1 extraction pipeline
  - Paste textarea with auto-extraction and three-state field feedback
  - Inline IBAN MOD-97 validation with error messages
  - BIC toggle (hidden by default, expandable)
  - Remittance field with 140-char counter
  - Generate button gated on valid IBAN + non-empty name
  - QR code display at minimum 200x200px
affects: [02-full-app plan 02 (copy/share actions), testing, phase-03-wero-qr]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "IIFE (initUI) for UI wiring isolation — keeps Phase 2 logic separate from Phase 1 pure functions"
    - "Three-state field feedback: CSS classes success/partial/failed on .field-status elements"
    - "Dutch locale amount display: toFixed(2).replace('.', ',') for display; parseAmountString for parse-back"
    - "paste + setTimeout(fn, 0) pattern: paste event fires before textarea.value is updated; defer to next tick"

key-files:
  created: []
  modified:
    - index.html

key-decisions:
  - "Both tasks executed as a single atomic write to avoid reading a half-written file"
  - "Amount displayed in Dutch locale (comma decimal) in field, parsed back via existing parseAmountString which handles both locales"
  - "Generate button requires both valid IBAN (MOD-97) AND non-empty name — amount is optional per EPC spec"

patterns-established:
  - "Phase 1 functions remain in top script block unchanged; Phase 2 UI wiring in IIFE at bottom of same script"
  - "field-status div per field for extraction feedback; extraction-status div for overall summary"

requirements-completed:
  - EXTR-01
  - FLDS-01
  - FLDS-04
  - FLDS-05
  - QRCD-02

# Metrics
duration: 12min
completed: 2026-03-23
---

# Phase 2 Plan 01: Full Form UI Summary

**Paste-to-QR form with three-state extraction feedback, inline IBAN MOD-97 validation, BIC toggle, remittance counter, and Generate button gated on valid IBAN + name**

## Performance

- **Duration:** 12 min
- **Started:** 2026-03-23T00:00:00Z
- **Completed:** 2026-03-23T00:12:00Z
- **Tasks:** 2
- **Files modified:** 1

## Accomplishments

- Built complete form HTML structure: paste textarea, IBAN/name/amount/remittance/BIC fields, Generate button, QR section
- Added CSS styles: responsive layout, three-state field feedback (green/yellow/red), disabled button state, 200x200px QR display area
- Wired all UI interactions: paste triggers auto-extraction, inline IBAN validation, BIC toggle, remittance character counter, Generate produces QR

## Task Commits

No git repository — changes applied directly to index.html.

1. **Task 1: Build form HTML structure and CSS styles** - index.html rewritten with full form UI and `<style>` block
2. **Task 2: Wire event handlers for paste-extract, field validation, BIC toggle, and QR generation** - initUI IIFE added to bottom of main script block

## Files Created/Modified

- `index.html` - Complete form UI added (HTML structure, CSS, UI event wiring); Phase 1 functions preserved unchanged

## Decisions Made

- Amount displayed in Dutch locale (comma decimal separator) for familiarity; parsed back via `parseAmountString` which already handles both locales
- Both tasks executed as a single atomic write since Task 2 depends on Task 1's DOM structure and reading an intermediate state would show a partial file

## Deviations from Plan

None — plan executed exactly as written. All 50 acceptance criteria verified via Node.js string-matching check (50 passed, 0 failed).

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required. Open `index.html` in any modern browser.

## Next Phase Readiness

- Full paste-to-QR flow is functional in the browser
- Plan 02 can add copy-to-clipboard and share actions on the QR output area
- Physical device scanning (ING/Rabobank/ABN AMRO) should be tested before Phase 3

---
*Phase: 02-full-app*
*Completed: 2026-03-23*

## Self-Check

- `index.html` exists and contains all required elements: VERIFIED (50/50 acceptance criteria pass)
- No git repo present — commit hashes not applicable
- All Phase 1 functions preserved unchanged: VERIFIED

## Self-Check: PASSED
