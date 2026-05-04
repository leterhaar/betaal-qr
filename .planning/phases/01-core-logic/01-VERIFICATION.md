---
phase: 01-core-logic
verified: 2026-03-23T16:00:00Z
status: human_needed
score: 13/13 must-haves verified
re_verification: false
human_verification:
  - test: "Open index.html via file:// in a browser, open DevTools console, run runTests()"
    expected: "All assertions print as PASS, final line reads '36 passed, 0 failed' (or similar), no console errors on load"
    why_human: "Browser execution environment cannot be verified programmatically — file:// protocol, DOM access in renderQR, and qrcode library SVG rendering all require a real browser"
  - test: "Scan the QR code produced by renderQR(formatEPCPayload({iban:'NL91ABNA0417164300', name:'Acme B.V.', amount:25.50})) with ING, Rabobank, or ABN AMRO mobile app"
    expected: "Banking app pre-fills SEPA transfer with IBAN NL91ABNA0417164300, amount EUR 25.50, beneficiary Acme B.V."
    why_human: "QR scan correctness in real banking apps cannot be verified without a physical device and live app"
---

# Phase 1: Core Logic Verification Report

**Phase Goal:** A browser-console-testable EPC payment pipeline — paste a payment object in, get a valid EPC string and extracted fields out
**Verified:** 2026-03-23T16:00:00Z
**Status:** human_needed — all automated checks pass; two browser/device checks remain
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| #  | Truth | Status | Evidence |
|----|-------|--------|----------|
| 1  | `extractIBAN('NL91 ABNA 0417 1643 00')` returns `'NL91ABNA0417164300'` (space-normalized, uppercase) | VERIFIED | Spot-check PASS; code strips spaces, uppercases, MOD-97 gates |
| 2  | `extractIBAN` returns null when given text with no IBAN | VERIFIED | Spot-check PASS; returns null for 'Just some random text' |
| 3  | `validateIBAN('NL91ABNA0417164300')` returns true (passes MOD-97) | VERIFIED | Spot-check PASS; iterative mod-97 remainder === 1 |
| 4  | `validateIBAN('NL91ABNA0417164301')` returns false (fails MOD-97) | VERIFIED | Spot-check PASS |
| 5  | `extractAmount('Totaal: EUR 1.234,56')` returns 1234.56 (Dutch locale) | VERIFIED | Spot-check PASS; last-separator detection correct |
| 6  | `extractAmount('Amount due: 1,234.56')` returns 1234.56 (English locale) | VERIFIED | Spot-check PASS |
| 7  | `extractName` returns a string from labeled text (t.n.v., begunstigde) or null if nothing found | VERIFIED | Spot-check PASS for 't.n.v. Acme B.V.' and null fallback |
| 8  | `formatEPCPayload({iban, name, amount:100})` produces a 12-line LF-separated string starting with BCD | VERIFIED | Spot-check PASS; 12 lines, starts 'BCD\n002\n1\nSCT\n' |
| 9  | `formatEPCPayload` output line 2 is '002' (version 2, BIC optional) | VERIFIED | Spot-check PASS |
| 10 | `formatEPCPayload` output line 8 is 'EUR100.00' for amount 100 | VERIFIED | Spot-check PASS |
| 11 | `formatEPCPayload` with amount null produces empty string on line 8 | VERIFIED | Spot-check PASS |
| 12 | `validateEPCPayload` rejects payloads over 331 bytes | VERIFIED | Spot-check PASS; >331-byte payload returns valid:false |
| 13 | `parsePaymentText` orchestrates extraction and returns {iban, name, amount} object | VERIFIED | Spot-check PASS; 'Betaal aan NL91 ABNA 0417 1643 00 bedrag EUR 50,00' → iban+amount correct |

**Score:** 13/13 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `index.html` | All extraction, EPC, and QR functions | VERIFIED | 325 lines; all required functions present and substantive |
| `lib/qrcode.js` | QR code generation library (vendored) | VERIFIED | 56,694 bytes — matches expected ~55KB |
| `lib/qrcode_UTF8.js` | UTF-8 companion for QR library (vendored) | VERIFIED | 793 bytes — matches expected ~800 bytes |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `normalizeText` | `extractIBAN`, `extractAmount`, `extractName` | Called as first step before regex | WIRED | `normalizeText(` appears 4 times in index.html (once per extraction function + one in parsePaymentText) |
| `extractIBAN` | `validateIBAN` | MOD-97 check on each candidate | WIRED | `validateIBAN(candidates[j])` called in extractIBAN loop (line 74) |
| `parsePaymentText` | `extractIBAN`, `extractAmount`, `extractName` | Orchestrator calls all extractors | WIRED | All three calls present at lines 234–237 |
| `formatEPCPayload` | `validateEPCPayload` | Payload validated (in test harness) | WIRED | `validateEPCPayload(formatEPCPayload(...))` called in runTests; no auto-validation on formatEPCPayload call itself, but the plan only required it in tests — satisfied |
| `renderQR` | qrcode library | `qrcode(0, 'M')` API call | WIRED | Line 220: `var qr = qrcode(0, 'M');` — lib loaded via script tags before main script |

---

### Data-Flow Trace (Level 4)

Not applicable — all functions are pure (no DB, no fetch, no store). Data flows in through function arguments and out through return values. No dynamic data sources to trace.

---

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| extractIBAN space-normalized | `extractIBAN('NL91 ABNA 0417 1643 00')` | `'NL91ABNA0417164300'` | PASS |
| extractIBAN returns null | `extractIBAN('Just some random text')` | `null` | PASS |
| validateIBAN true | `validateIBAN('NL91ABNA0417164300')` | `true` | PASS |
| validateIBAN false | `validateIBAN('NL91ABNA0417164301')` | `false` | PASS |
| extractAmount Dutch locale | `extractAmount('Totaal: EUR 1.234,56')` | `1234.56` | PASS |
| extractAmount English locale | `extractAmount('Amount due: 1,234.56')` | `1234.56` | PASS |
| extractName label match | `extractName('t.n.v. Acme B.V.')` | `'Acme B.V.'` | PASS |
| formatEPCPayload 12 lines | `.split('\n').length` | `12` | PASS |
| formatEPCPayload BCD header | `.startsWith('BCD\n002\n1\nSCT\n')` | `true` | PASS |
| formatEPCPayload line 2 | `[1]` | `'002'` | PASS |
| formatEPCPayload amount line | `[7]` | `'EUR100.00'` | PASS |
| formatEPCPayload null amount | `amount:null → [7]` | `''` | PASS |
| validateEPCPayload over limit | 320-byte padding → valid | `false` | PASS |
| parsePaymentText orchestrates | full invoice string | `iban + amount correct` | PASS |
| no import/export | file scan | absent | PASS |
| no fetch/XHR | file scan | absent | PASS |
| normalizeText wired to extractors | call-count scan | 4 calls | PASS |

**17/17 spot-checks passed**

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| EXTR-02 | 01-01-PLAN | App extracts IBAN from pasted text using regex pattern matching | SATISFIED | `extractIBAN` uses `/\b([A-Z]{2}\d{2}(?:\s?[A-Z0-9]{4}){3,7}...)\b/gi` with MOD-97 gating |
| EXTR-03 | 01-01-PLAN | App extracts payment amount (handles EUR 1.234,56 / 1,234.56 formats) | SATISFIED | `extractAmount` + `parseAmountString` handle Dutch/English locales via last-separator detection |
| EXTR-04 | 01-01-PLAN | App extracts counterparty name using heuristics (t.n.v., begunstigde, IBAN proximity) | SATISFIED | `extractName` uses label regex (t.n.v., begunstigde, naam, beneficiary) + IBAN proximity fallback |
| FLDS-02 | 01-01-PLAN | IBAN validated using MOD-97 checksum | SATISFIED | `validateIBAN` implements iterative MOD-97 (remainder === 1); called in `extractIBAN` |
| FLDS-03 | 01-01-PLAN | Beneficiary name field enforces 70-character EPC limit | SATISFIED | `extractName` applies `.substring(0, 70)`; `formatEPCPayload` applies `name.substring(0, 70)` on line 6 |
| QRCD-01 | 01-02-PLAN | App generates valid EPC QR code (v002, UTF-8, BIC optional) from confirmed fields | SATISFIED | `formatEPCPayload` builds 12-line EPC069-12 v3.1 payload; `renderQR` wraps qrcode-generator; browser test needed to confirm SVG renders |
| QRCD-03 | 01-02-PLAN | App validates payload does not exceed 331-byte EPC limit before generating | SATISFIED | `validateEPCPayload` uses `TextEncoder` byte-count check against 331 |
| GENL-01 | 01-02-PLAN | App runs as a single static HTML+JS file | SATISFIED | Single `index.html`; `lib/` files are local — no CDN, no server required |
| GENL-03 | 01-02-PLAN | All processing is client-side — no data leaves the browser | SATISFIED | No `fetch`, no `XMLHttpRequest` in index.html; lib files vendored locally |

**9/9 requirements satisfied**

No orphaned requirements: every ID listed in ROADMAP Phase 1 (`EXTR-02, EXTR-03, EXTR-04, FLDS-02, FLDS-03, QRCD-01, QRCD-03, GENL-01, GENL-03`) appears in a plan's `requirements` field and has been verified above.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None found | — | — | — | — |

Scanned for: TODO/FIXME/placeholder comments, `return null`/`return {}`/`return []` as stubs, empty handlers, no-op implementations, hardcoded empty values flowing to render, auto-executed `runTests()`. None found.

Note: `return null` appears in extractIBAN, extractAmount, extractName — these are correct "not found" sentinel returns, not stubs. Each function has substantial logic before the null return.

---

### Human Verification Required

#### 1. runTests() in browser console

**Test:** Open `index.html` via `file://` URL in Chrome, Firefox, or Edge. Open DevTools console. Type `runTests()` and press Enter.
**Expected:** All assertions print PASS; final summary shows 0 failed. No errors in console on page load (lib files load from `lib/` correctly).
**Why human:** Requires real browser environment — `file://` protocol behavior, DOM readiness for `renderQR`, and actual qrcode library SVG output all need a browser to confirm.

#### 2. End-to-end QR scan in Dutch banking app

**Test:** In the browser console, run:
```javascript
renderQR(formatEPCPayload({iban:'NL91ABNA0417164300', name:'Acme B.V.', amount:25.50}))
```
Then scan the QR code that appears on the page with ING, Rabobank, or ABN AMRO mobile app.
**Expected:** App pre-fills a SEPA credit transfer with IBAN `NL91ABNA0417164300`, amount EUR 25.50, beneficiary name `Acme B.V.`.
**Why human:** Requires physical device with banking app. EPC QR scan correctness is the ultimate acceptance test for QRCD-01.

---

### Gaps Summary

No gaps. All 13 must-have truths verified, all 9 requirements satisfied, all 3 artifacts present and substantive, all key links wired, 17/17 spot-checks pass.

Two items require human verification because they depend on browser rendering and a physical banking app scan. These are not blocking gaps — they are acceptance checks for already-correct code.

---

_Verified: 2026-03-23T16:00:00Z_
_Verifier: Claude (gsd-verifier)_
