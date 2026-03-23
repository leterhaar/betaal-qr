---
phase: 01-core-logic
plan: 02
subsystem: core-logic
tags: [epc, qr, payload, validation, pipeline]
dependency_graph:
  requires: [01-01]
  provides: [formatEPCPayload, validateEPCPayload, parsePaymentText, renderQR, lib/qrcode.js, lib/qrcode_UTF8.js]
  affects: [index.html]
tech_stack:
  added: [qrcode-generator@2.0.4]
  patterns: [pure-function-pipeline, vendored-library, test-harness]
key_files:
  created: [lib/qrcode.js, lib/qrcode_UTF8.js]
  modified: [index.html]
decisions:
  - "validateEPCPayload returns valid:false when either byte limit OR header check fails (combined guard)"
  - "renderQR defaults to #qr-output div when no containerElement passed"
  - "qrcode_UTF8.js loaded after qrcode.js to patch stringToBytes for UTF-8 beneficiary names"
metrics:
  duration: ~5 minutes
  completed: "2026-03-23"
  tasks: 2
  files: 3
---

# Phase 01 Plan 02: EPC Payload, QR Library, and Pipeline Summary

One-liner: EPC069-12 payload formatter, 331-byte validator, QR renderer wired end-to-end via parsePaymentText orchestrator using vendored qrcode-generator@2.0.4.

## What Was Built

Complete Phase 1 pipeline added to `index.html`:

- **formatEPCPayload** — assembles the 12-line EPC069-12 v3.1 payload (LF-separated, version 002, BIC optional, UTF-8, amounts as EURnnn.nn)
- **validateEPCPayload** — byte-counts payload with TextEncoder, checks 331-byte limit and BCD header
- **parsePaymentText** — orchestrator calling normalizeText → extractIBAN → extractAmount → extractName, returns `{iban, name, amount}`
- **renderQR** — thin wrapper over qrcode-generator, produces SVG into `#qr-output` div
- **lib/qrcode.js** — qrcode-generator v2.0.4 vendored from jsDelivr (~55KB)
- **lib/qrcode_UTF8.js** — companion file patching stringToBytes for UTF-8 encoding (793 bytes)
- Extended `runTests()` with 16 new test cases covering EPC format, validation, orchestrator, and end-to-end QR rendering

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| Task 1 | 439a654 | feat(01-02): add formatEPCPayload, validateEPCPayload, parsePaymentText |
| Task 2 | 5ccb062 | feat(01-02): vendor QR library, add renderQR, extend test harness |

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| validateEPCPayload checks both byte limit AND header | Either failure makes the payload invalid for banking apps |
| renderQR defaults to #qr-output | Matches the div added to body; allows console testing without arguments |
| Load qrcode_UTF8.js after qrcode.js | Companion file patches global stringToBytes at load time — order is mandatory |
| Vendor to lib/ not CDN | Supports offline/file:// use per CLAUDE.md and project requirements |

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

None — all functions are fully wired. The `#qr-output` div and `renderQR` are functional but the UI to trigger them comes in Phase 2.

## Self-Check: PASSED

- lib/qrcode.js: FOUND (56694 bytes)
- lib/qrcode_UTF8.js: FOUND (793 bytes)
- index.html contains function formatEPCPayload: FOUND
- index.html contains function validateEPCPayload: FOUND
- index.html contains function parsePaymentText: FOUND
- index.html contains function renderQR: FOUND
- Commits 439a654 and 5ccb062: FOUND in git log
