# Project Research Summary

**Project:** Betaal QR
**Domain:** Static client-side EPC/SEPA payment QR code generator
**Researched:** 2026-03-23
**Confidence:** HIGH (spec-level decisions) / MEDIUM (extraction heuristics, library versions)

## Executive Summary

Betaal QR is a single-file, zero-dependency static web app that accepts raw invoice text, extracts IBAN/amount/name via regex, lets the user confirm the extracted fields, and generates an EPC QR code (EPC069-12 standard) that Dutch banking apps (ING, Rabobank, ABN AMRO) can scan to pre-fill a SEPA Credit Transfer. The right approach is a linear pipeline — paste area to text parser to field editor to EPC formatter to QR renderer — implemented entirely in vanilla HTML/CSS/JS with one CDN-loaded QR library. No framework, no build step, no server. The entire app fits in under 300 lines of code.

The recommended build order starts with the two pure-function layers (EPC formatter and text parser) because they carry the highest correctness risk and are independently testable without a browser. The QR renderer comes third, the form UI fourth, and DOM wiring last. This inside-out order surfaces spec compliance failures early — a wrong EPC payload discovered at step 1 costs minutes; the same bug discovered during UI testing costs hours.

The highest risks are all in data correctness, not technical complexity: (1) Dutch locale number format ambiguity (`1.234,56` vs `1,234.56`) can cause silent wrong-amount payments; (2) incorrect EPC payload structure (wrong version, `\r\n` line endings, missing blank lines for optional fields) causes the QR to be unscannable in banking apps; (3) IBAN extraction without MOD-97 checksum validation allows format-correct but account-incorrect IBANs to pass silently. All three risks are mitigated by a mandatory user review step before QR generation — the review step is a safety gate, not optional UX polish.

---

## Key Findings

### Recommended Stack

The project is fully constrained by PROJECT.md: vanilla JavaScript (ES2020+), plain HTML5, plain CSS, no build tools, no Node, no transpilation. The app must open directly as `index.html` in a browser. The only external dependency is a single QR code generation library loaded via CDN script tag.

The recommended QR library is **qrcode-generator** by kazuhikoarase (v1.4.4, jsDelivr CDN), a pure-JS UMD module with no sub-dependencies that supports error correction level M as required by EPC069-12. For offline/reliable use, the library should be vendored locally at `/lib/qrcode.min.js` rather than loaded from CDN at runtime. Fallback option is **qr-creator** (v1.0.0), which has a simpler API and also supports SVG output.

All other logic — IBAN extraction, amount parsing, EPC payload assembly, IBAN checksum validation — is implemented as inline pure functions. No external library is needed for any of these.

**Core technologies:**
- **Vanilla JS (ES2020+):** All app logic — mandated by PROJECT.md, ~200-300 LOC total
- **HTML5:** Single-file shell with textarea, inputs, canvas, and button elements
- **Plain CSS:** Styling — no preprocessor, no framework
- **qrcode-generator (v1.4.4):** QR image generation — only external dependency, loadable from CDN or vendored locally
- **Regex + mod-97 inline functions:** IBAN extraction and validation, amount parsing, name heuristics — no library needed

### Expected Features

The EPC QR standard mandates: BCD service tag, version 002, UTF-8 encoding (character set 1), SCT identification, beneficiary name (max 70 chars), beneficiary IBAN (max 34 chars), and an optional amount field formatted as `EURnnn.nn`. All 12 payload lines must be present; optional fields use blank lines as placeholders. Total payload must not exceed 331 bytes.

**Must have (table stakes):**
- IBAN input with MOD-97 checksum validation — wrong IBAN = lost money
- Beneficiary name input with 70-char enforcement
- Amount input with locale-aware normalisation (Dutch `1.234,56` and English `1,234.56`)
- EPC QR code generation and display (min 200x200px canvas, error correction level M)
- Inline validation errors before generation — no alert dialogs
- Review/edit step after extraction, before QR generation
- Works offline, opens as file:// without install

**Should have (differentiators):**
- Auto-extraction from pasted invoice text (IBAN regex + amount regex + name heuristic) — the core differentiator that justifies building this tool
- Unstructured remittance field (invoice reference, max 140 chars) — low complexity, high practical value
- Download QR as PNG — canvas `toDataURL()`, low complexity
- Explicit UI feedback for partial/failed extraction (three states: success, partial, failed)
- Copy QR to clipboard — nice-to-have, low complexity

**Defer (v2+):**
- Structured remittance reference (ISO 11649 creditor reference with check digit) — non-trivial calculation
- localStorage persistence of last-used beneficiary
- BIC field (visible but hidden by default — v002 makes it unnecessary for Dutch banks)
- Purpose code dropdown (ISO 20022 codes — niche B2B use case)
- Dark/light mode (`prefers-color-scheme`)
- Print-friendly layout (`@media print`)

**Explicit anti-features (never build):**
- AI/LLM extraction, OCR/PDF upload, server-side processing, payment execution, multi-payment batch, user accounts

### Architecture Approach

The app is a linear synchronous pipeline with four logical layers embedded in a single HTML file. There is no routing, no state management library, and no build step. The pipeline is: paste area collects raw text, text parser extracts structured fields (pure function, no DOM access), field editor lets the user review and correct extracted values, EPC formatter assembles the payload string (pure function, owns all format rules), and QR renderer passes the string to the library and updates the canvas.

**Major components:**
1. **Text Parser** (`parsePaymentText(rawString) -> ParsedPayment`) — pure function; extracts IBAN, amount, name from raw invoice text; independently testable
2. **EPC Formatter** (`formatEPCPayload(confirmed) -> string`) — pure function; owns all EPC069-12 format knowledge; single source of truth for payload structure
3. **Field Validator** (`validateFields(confirmed) -> errors`) — runs between Field Editor and EPC Formatter; blocks generation if IBAN checksum fails, amount invalid, or name missing
4. **QR Renderer** (`renderQR(payload) -> void`) — thin wrapper around qrcode-generator library; renders to canvas; separate from formatter
5. **DOM Wiring** — event listeners only, no logic; calls the pure functions above and applies results to DOM

Build order: EPC Formatter first (no deps, highest spec risk), Text Parser second (no deps), QR Renderer third, Field Editor UI fourth, DOM wiring last.

### Critical Pitfalls

1. **Amount locale ambiguity (`1.234,56` vs `1,234.56`)** — Detect decimal separator by identifying the last separator in the number string; strip currency symbols first; always use `.toFixed(2)` for EPC payload. Test with 8+ format variants including Dutch large amounts. Silent wrong amounts destroy user trust.

2. **EPC payload field order and line termination** — Use exactly 12 lines with `\n` (LF only, not CRLF); blank lines are required placeholders for omitted optional fields; amount must be `EUR1234.56` (not `EUR1234,56` or `EUR 1234.56`); use version `002` not `001`. Write a unit test asserting exact payload string output.

3. **IBAN extraction without MOD-97 validation** — Format-correct IBANs with wrong check digits pass regex silently. MOD-97 is ~10 lines of JS; implement it as a hard requirement alongside IBAN regex. If multiple IBANs found, the one passing MOD-97 wins.

4. **IBAN formatted with spaces fails regex** — Invoice IBANs appear as `NL91 ABNA 0417 1643 00`. Strip spaces and hyphens from IBAN candidate strings before validation. Test with spaced, hyphenated, and mixed-case variants.

5. **UTF-8 encoding not explicitly set on QR library** — Beneficiary names with accented characters (Société, Müller) produce garbled QR text if library defaults to Latin-1. Explicitly configure UTF-8. Test with at least one non-ASCII name before shipping.

6. **QR density too high for mobile banking app scanners** — Use error correction level M (not H); enforce 70-char name limit; limit remittance field; render at minimum 200x200px. Physical device testing with ING/Rabobank/ABN AMRO apps is mandatory before declaring done.

---

## Implications for Roadmap

Based on the research, the app decomposes naturally into 3 phases that follow the component build order recommended in ARCHITECTURE.md.

### Phase 1: Core Logic (Formatter + Parser + Validator)

**Rationale:** The pure-function layers carry the highest correctness risk and have no UI dependencies. Building and verifying them first means spec compliance failures surface in isolation, not buried under UI code. EPC payload bugs discovered here cost minutes; discovered during integration testing cost hours.

**Delivers:** A working, browser-console-testable EPC QR pipeline. Paste a payment object into the console, get a valid EPC string out.

**Addresses features:**
- EPC payload assembly (all 12 fields, correct order, LF line endings, version 002)
- IBAN extraction regex + MOD-97 checksum validation
- Amount extraction with Dutch/English locale normalisation
- Name extraction heuristics (proximity-to-IBAN, Dutch label patterns)
- Field validation (IBAN checksum, amount range, 70-char name limit)

**Avoids pitfalls:**
- Pitfall 2 (EPC field order / line termination) — unit tested before any UI
- Pitfall 3 (amount locale ambiguity) — explicit test suite with 8+ amount formats
- Pitfall 5 (IBAN checksum skipped) — MOD-97 built as core, not polish
- Pitfall 12 (trailing zeros in amount) — `.toFixed(2)` enforced in formatter

**Research flag:** SKIP — all patterns are well-documented from EPC069-12 spec and ISO 13616. No phase-level research needed.

---

### Phase 2: QR Rendering + Field UI

**Rationale:** QR rendering and the form UI are independent of each other but both depend on Phase 1 functions being correct. Building them together as Phase 2 produces a working end-to-end app (manual data entry path) before the auto-extraction feature is layered on.

**Delivers:** A functional form-based EPC QR generator. User enters IBAN, name, amount manually, clicks Generate, scans QR in banking app. The core user flow works.

**Uses:**
- qrcode-generator (v1.4.4) vendored locally — avoids CDN dependency for offline use
- HTML5 canvas for QR display
- Inline validation error display (no alert dialogs)
- Download as PNG via `canvas.toDataURL()`

**Implements:**
- QR Renderer component
- Field Editor UI (IBAN, name, amount, remittance inputs)
- Validate-before-render gate
- Generate button wiring

**Avoids pitfalls:**
- Pitfall 1 (UTF-8 encoding) — explicitly configured on QR library call, tested with accented name
- Pitfall 7 (BIC version 001 vs 002) — version 002 hardcoded, BIC field not shown
- Pitfall 8 (QR density) — error correction level M, minimum 200x200px canvas, SVG preferred if supported
- Pitfall 14 (blurry canvas) — use SVG output or retina-scaled canvas

**Research flag:** SKIP — QR library integration is straightforward; library API is well-documented.

---

### Phase 3: Auto-Extraction + UX Polish

**Rationale:** Auto-extraction (the core differentiator) is built last because it layers on top of the already-working manual flow. If extraction fails or produces wrong results, the user can always correct fields manually — the safety net is already in place. Polish features (copy to clipboard, extraction feedback states) come last.

**Delivers:** The complete product. User pastes invoice text, fields auto-populate, user confirms/corrects, generates and downloads QR.

**Addresses features:**
- Paste area with auto-extraction trigger
- Multiple IBAN candidates UI (user picks correct one)
- Three-state extraction feedback (success / partial / failed)
- Copy QR to clipboard (Clipboard API)
- Explicit "could not find amount — please enter manually" messages

**Avoids pitfalls:**
- Pitfall 4 (IBAN formatted with spaces) — normalization applied before regex matching
- Pitfall 9 (multiple IBANs — wrong one selected) — all candidates shown, user selects
- Pitfall 10 (amount regex matches non-payment numbers) — amount anchored to currency labels
- Pitfall 11 (non-breaking spaces / Unicode dashes in pasted text) — input normalization first step
- Pitfall 15 (no feedback when extraction fails) — three explicit UI states

**Research flag:** NEEDS ATTENTION — name extraction heuristics (proximity to IBAN, Dutch label patterns) are the lowest-confidence part of the research. Test with real Dutch invoice samples early in this phase.

---

### Phase Ordering Rationale

- The inside-out build order (logic first, UI second, wiring third) is directly recommended by ARCHITECTURE.md's Component Build Order section and is the right call for a spec-compliance-sensitive domain.
- Phases 1 and 2 together produce a usable product (manual entry path). Phase 3 adds the differentiator. This means the app ships value even if Phase 3 extraction is imperfect.
- EPC spec correctness bugs discovered in Phase 1 are cheap. The same bugs discovered during Phase 3 integration testing are expensive. The phase split enforces early verification.

### Research Flags

Phases needing deeper research during planning:
- **Phase 3 (name extraction):** Heuristics for extracting beneficiary name from Dutch invoice text are medium-confidence. Gather 5-10 real Dutch invoice text samples before writing extraction logic. The "line immediately before IBAN" heuristic is a reasonable starting point but may need tuning.

Phases with standard patterns (skip research-phase):
- **Phase 1:** EPC069-12 and ISO 13616 are formal published standards. All rules are deterministic.
- **Phase 2:** QR library integration is standard; EPC format is known; HTML5 form patterns are standard.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | PROJECT.md mandates vanilla JS; QR library identity is correct; CDN URLs need version check before shipping |
| Features | HIGH | EPC069-12 field spec is formal standard; feature set is well-scoped; Dutch banking app support for v002 is well-established since 2016 |
| Architecture | HIGH | Linear pipeline pattern is unambiguous for this problem size; pure-function decomposition is standard practice |
| Pitfalls | HIGH (spec-level) / MEDIUM (heuristics) | EPC format rules and IBAN checksum are formally specified — HIGH. Dutch invoice label patterns and QR scanner density thresholds are domain knowledge — MEDIUM. |

**Overall confidence:** HIGH for the build approach and spec-level decisions; MEDIUM for extraction heuristics (name extraction in particular).

### Gaps to Address

- **EPC spec version:** Research cites both "Version 3.0 (2022)" and "Version 6.0 (2019)" across files — there is an inconsistency. Before implementing the formatter, fetch the current EPC069-12 PDF from europeanpaymentscouncil.eu to confirm the latest version number and verify that BIC remains optional in the current version.
- **qrcode-generator UTF-8 configuration:** The library's exact API for setting encoding is noted as LOW confidence. Read the library source or documentation before writing the QR renderer to confirm how to explicitly set UTF-8 byte encoding.
- **qrcode-generator CDN version:** Version 1.4.4 was current as of mid-2025 training data. Verify latest version at npmjs.com/package/qrcode-generator before pinning.
- **Device scanning validation:** No amount of unit testing replaces scanning the generated QR with actual ING, Rabobank, and ABN AMRO mobile apps. Schedule this early in Phase 2, not at the end.
- **Name extraction accuracy:** The name heuristic is the least reliable extraction component. Gather real Dutch invoice samples before writing the name extractor to calibrate the heuristic. Build the UI to make name correction easy (pre-filled but clearly editable).

---

## Sources

### Primary (HIGH confidence)
- EPC069-12 "Quick Response Code: Guidelines to Enable the Data Capture for the Initiation of a SEPA Credit Transfer" (European Payments Council) — all EPC payload format rules, field lengths, encoding requirements, version 002 BIC-optional status. Verify current version at: https://www.europeanpaymentscouncil.eu/document-library/guidance-documents
- ISO 13616-1: International Bank Account Number (IBAN) — MOD-97 check digit algorithm, country code/length constraints

### Secondary (MEDIUM confidence)
- npm package documentation for qrcode-generator (kazuhikoarase) v1.4.4 — QR library API, CDN URL, error correction level support. Verify at: https://www.npmjs.com/package/qrcode-generator
- npm package documentation for qr-creator v1.0.0 — fallback QR library; simpler API, SVG support
- Dutch banking app EPC QR support (ING, Rabobank, ABN AMRO) — all support EPC QR v002; well-established since 2016 BIC-free SEPA mandate

### Tertiary (LOW confidence)
- Dutch invoice formatting conventions and label patterns (`t.n.v.`, `Begunstigde:`, `Ontvanger:`) — domain knowledge, unverified against current invoice samples
- QR scanner density thresholds for mobile banking apps — domain knowledge, requires physical device testing to confirm

---
*Research completed: 2026-03-23*
*Ready for roadmap: yes*
