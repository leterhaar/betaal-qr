---
phase: 01-core-logic
plan: 01
subsystem: extraction
tags: [iban, mod97, amount-parsing, regex, epc, vanilla-js]

# Dependency graph
requires: []
provides:
  - normalizeText: Unicode artifact stripping for copy-paste robustness
  - validateIBAN: ISO 13616 MOD-97 iterative check digit validation
  - extractIBAN: space-tolerant IBAN regex with MOD-97 gating, returns first valid IBAN
  - parseAmountString: Dutch/English locale detection by last-separator position
  - extractAmount: currency-anchored regex returning last valid match
  - extractName: Dutch label heuristics + IBAN proximity fallback, 70-char EPC limit enforced
  - runTests: inline browser-console test harness (20 assertions, 0 failures)
affects: [01-core-logic-02, 02-ui]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Pure function pipeline — all extraction functions have no DOM access or side effects
    - Text normalization first — normalizeText called at top of every parsing function
    - Locale detection by last separator — Dutch vs English amount parsing
    - Iterative MOD-97 — avoids BigInt, works in all browsers
    - Currency-anchored amount regex — prevents false positives on VAT rates, dates, invoice numbers

key-files:
  created:
    - index.html
  modified: []

key-decisions:
  - "Regex/pattern matching (no AI/LLM) for extraction — offline, free, sufficient for structured payment data"
  - "First valid IBAN wins when multiple IBANs found — user constraint from CONTEXT.md"
  - "extractAmount returns last valid match — invoice totals appear at bottom of document"
  - "extractName returns null rather than guessing — Phase 2 UI will prompt user to correct"
  - "validateIBAN uses iterative mod-97 — avoids BigInt requirement, works in all browsers"

patterns-established:
  - "Pattern 1: All pure functions declared at top-level script scope for browser console access — no export/import"
  - "Pattern 2: normalizeText(text) called as first statement in every extraction function"
  - "Pattern 3: Amount locale detected by comparing lastIndexOf(',') vs lastIndexOf('.') — whichever is larger is the decimal separator"
  - "Pattern 4: IBAN regex allows optional spaces every 4 chars, strips spaces before MOD-97 check"

requirements-completed: [EXTR-02, EXTR-03, EXTR-04, FLDS-02, FLDS-03]

# Metrics
duration: 2min
completed: 2026-03-23
---

# Phase 1 Plan 01: Core Extraction and Validation Functions Summary

**Five pure JS functions (normalizeText, validateIBAN, extractIBAN, extractAmount, extractName) with Dutch-locale IBAN/amount parsing, MOD-97 validation, and an inline 20-case test harness — all in a single index.html callable from the browser console**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-23T14:28:24Z
- **Completed:** 2026-03-23T14:30:02Z
- **Tasks:** 2 (both completed in single file)
- **Files modified:** 1

## Accomplishments

- All five extraction/validation functions implemented as pure functions with no DOM access, no network calls, no side effects
- IBAN extraction handles both space-separated (`NL91 ABNA 0417 1643 00`) and contiguous (`NL91ABNA0417164300`) formats, validated with iterative MOD-97
- Amount parsing correctly handles Dutch locale (1.234,56 -> 1234.56) and English locale (1,234.56 -> 1234.56) via last-separator position detection
- Name extraction uses Dutch label heuristics (t.n.v., begunstigde, naam, ten name van) with IBAN proximity fallback; enforces 70-char EPC limit
- Inline test harness `runTests()` with 20 assert cases confirmed 0 failures via Node.js evaluation

## Task Commits

Each task was committed atomically:

1. **Task 1+2: Create index.html with extraction functions and test harness** - `d80e6fa` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified

- `index.html` - All five extraction/validation pure functions + runTests() inline test harness (220 lines)

## Decisions Made

- Tasks 1 and 2 were implemented together in a single commit since the test harness (`runTests()`) is a natural part of the same `<script>` block as the functions it tests, and both tasks target the same file.
- Used `var` instead of `let`/`const` for ES5 compatibility with older browsers that might load a `file://` page — conservative choice for a local tool.
- `extractName` uses a two-pass approach: label regex first, IBAN proximity fallback second. Returns `null` on no match rather than guessing.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required. All code runs as a static `file://` page in any modern browser.

## Next Phase Readiness

- All extraction functions are complete and verified: ready for Plan 02 (EPC payload formatter + QR renderer)
- `runTests()` can be extended in Phase 2 to cover `formatEPCPayload` and QR generation
- No blockers for Plan 02

---
*Phase: 01-core-logic*
*Completed: 2026-03-23*
