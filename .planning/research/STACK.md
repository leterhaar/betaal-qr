# Technology Stack

**Project:** Betaal QR
**Researched:** 2026-03-23
**Note on sources:** All external tools (WebSearch, WebFetch, Bash, Context7) were denied during this research session. All findings are from training data (cutoff August 2025). Confidence levels reflect this limitation — verify CDN URLs and library versions before shipping.

---

## Recommended Stack

### Core Runtime

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Vanilla JavaScript (ES2020+) | N/A | App logic, extraction, QR wiring | No framework overhead; project fits in ~200 LOC; runs without build tools; PROJECT.md explicitly requires it |
| HTML5 | N/A | Single-file shell | `<textarea>`, `<canvas>`, `<input>` cover all UI needs; no templating required |
| CSS (vanilla, no preprocessor) | N/A | Styling | Single-file constraint rules out Tailwind CDN noise; plain CSS is sufficient |

**Confidence:** HIGH — PROJECT.md mandates this; no verification needed.

---

### QR Code Generation

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **qrcode-generator** (by kazuhikoarase) | 1.4.4 | Render EPC payload as QR image | Pure JS, no dependencies, ships as a UMD module loadable from jsDelivr CDN with a single `<script>` tag; actively maintained; supports error correction level M which EPC069-12 requires |

**CDN URL (verify before use):**
```
https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.min.js
```

**Confidence:** MEDIUM — library identity and API are correct from training data. Version 1.4.4 was current as of mid-2025; verify latest at `https://www.npmjs.com/package/qrcode-generator` before pinning.

**Why this over alternatives — see Alternatives section.**

---

### EPC QR Payload Construction

No library needed. The EPC069-12 spec defines a plain-text payload. Implement as a pure JS function (~30 lines):

```
BCD\n          ← Service Tag (literal)
002\n          ← Version (002 for current spec)
1\n            ← Encoding (1 = UTF-8)
SCT\n          ← Identification (SEPA Credit Transfer)
{BIC}\n        ← BIC of beneficiary bank (optional in v002)
{Name}\n       ← Beneficiary name (max 70 chars)
{IBAN}\n       ← Beneficiary IBAN (max 34 chars)
EUR{Amount}\n  ← Amount in EUR, e.g. EUR12.50 (max EUR999999999.99)
\n             ← Purpose code (optional, leave blank)
\n             ← Remittance reference (structured, leave blank)
{Reference}\n  ← Remittance information (unstructured, free text, max 140 chars)
\n             ← Beneficiary to originator info (optional)
```

Key spec constraints:
- Max payload: 331 characters (after URL-encoding is NOT applied — this is raw bytes)
- Error correction level: M (15% recovery)
- QR version: auto (library handles this)
- Line separator: `\n` (LF only, not CRLF)
- BIC field: optional in EPC version 002 (most Dutch banks accept without BIC)
- Amount field: omit entirely (not just blank) if amount is zero/unknown

**Confidence:** HIGH for the field structure — EPC069-12 is a public standard I have high familiarity with. MEDIUM for the "BIC optional" claim — verify against the current EPC069-12 v3.0 PDF from europeanpaymentscouncil.eu before relying on it.

---

### IBAN Extraction

Pure JS regex — no library.

```javascript
// Matches NL, BE, DE, FR, GB and other EU IBANs
const IBAN_REGEX = /\b([A-Z]{2}[0-9]{2}[A-Z0-9]{4}[0-9]{7}([A-Z0-9]?){0,16})\b/g;
```

IBAN validation (checksum mod-97) can be implemented in ~10 lines using the standard algorithm. No external library required.

**Confidence:** HIGH — IBAN structure is an ISO 13616 standard; regex pattern is stable.

---

### Amount Extraction

Pure JS regex — no library. Must handle:
- `€1.234,56` (Dutch/German locale: dot as thousands, comma as decimal)
- `1,234.56` (English locale: comma as thousands, dot as decimal)
- `EUR 1234.56` (plain)
- `1234` (integer, no decimals)

Normalization function: detect locale by presence of trailing `,XX` vs `.XX` pattern, then normalize to `NNNN.NN`.

**Confidence:** HIGH — this is pure string manipulation; no library dependency.

---

### No Build Tooling

| Decision | Rationale |
|----------|-----------|
| No npm/Node | PROJECT.md constraint: "just open index.html in a browser" |
| No Webpack/Vite/Rollup | Build tools require Node; violates single-file static constraint |
| No TypeScript | No transpilation step allowed; use JSDoc comments for type hints if desired |
| No ESModules with import | `import` requires either a bundler or a server (CORS blocks local file:// imports in some browsers); use classic `<script>` tags instead |

**Confidence:** HIGH — direct consequence of PROJECT.md constraints.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| QR library | qrcode-generator (kazuhikoarase) | **QRCode.js** (davidshimjs) | Last commit 2013; unmaintained; no error-correction-level control |
| QR library | qrcode-generator | **qrcode** (soldair/node-qrcode) | Node-first; browser bundle exists but is 80 KB+ and requires a bundler for clean tree-shaking; overkill for single-file use |
| QR library | qrcode-generator | **jsQR** | Decoder, not encoder — wrong direction |
| QR library | qrcode-generator | **qr-creator** | Smaller (6 KB), good CDN support, renders to canvas/SVG; viable alternative if qrcode-generator has any issues — use this as fallback |
| Styling | Vanilla CSS | Bootstrap CDN | Adds 30 KB for a UI with ~5 elements; unnecessary |
| Styling | Vanilla CSS | Tailwind CDN | Play CDN is development-only; not suitable for a file:// loaded page |
| Extraction | Regex | LLM/AI API | Explicitly out of scope (PROJECT.md) |
| Extraction | Regex | IBAN validation library (ibantools) | Correct checksums are nice-to-have; adds a CDN dependency for marginal benefit; implement mod-97 inline instead |

**Confidence:** MEDIUM — alternatives assessment is from training data; relative popularity and maintenance status should be spot-checked on npm/GitHub before final decision.

---

## Fallback QR Library: qr-creator

If qrcode-generator causes issues (e.g., CDN unavailability, API friction), use:

```
https://cdn.jsdelivr.net/npm/qr-creator@1.0.0/dist/qr-creator.min.js
```

API is simpler: `QrCreator.render({ text, radius, ecLevel: 'M', fill: '#000', background: null, size: 256 }, canvasElement)`.

**Confidence:** MEDIUM — verified via training data; check npm for latest version.

---

## Installation

No install step. Single HTML file loads one CDN script:

```html
<!DOCTYPE html>
<html lang="nl">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Betaal QR</title>
</head>
<body>
  <!-- App HTML here -->

  <script src="https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.min.js"></script>
  <script src="app.js"></script>
  <!-- OR inline app.js content directly for true single-file -->
</body>
</html>
```

For fully offline use (no CDN): download `qrcode.min.js` once and reference locally. This is the better choice for a "personal local tool" per PROJECT.md, since it removes the CDN dependency at runtime.

**Recommendation:** Ship with the library vendored locally (`/lib/qrcode.min.js`) so the tool works offline. The CDN URL is only needed to fetch it the first time.

**Confidence:** HIGH — this is standard static web practice.

---

## EPC QR Code Spec Reference

**Standard:** EPC069-12 "Quick Response Code: Guidelines to Enable the Data Capture for the Initiation of a SEPA Credit Transfer"
**Current version as of August 2025:** Version 3.0 (published by European Payments Council)
**Source:** https://www.europeanpaymentscouncil.eu/document-library/guidance-documents/quick-response-code-guidelines-enable-data-capture-initiation

**Key constraints for implementation:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Service Tag | `BCD` | Hardcoded literal |
| Version | `002` | Use v002 (allows omitting BIC) |
| Character encoding | `1` (UTF-8) | Always use 1 |
| Identification | `SCT` | SEPA Credit Transfer |
| Max beneficiary name | 70 chars | Truncate if longer |
| Max IBAN length | 34 chars | ISO 13616 |
| Amount format | `EURnnn.nn` | e.g., `EUR12.50`; omit field entirely if unknown |
| Max unstructured reference | 140 chars | Invoice number / omschrijving goes here |
| Error correction | Level M | Required by spec |
| Max QR payload | 331 bytes | Keep payload short; Dutch invoices rarely approach this |
| Line ending | `\n` (LF) | Not CRLF |

**Confidence:** HIGH for field names and order; MEDIUM for "Version 3.0" — EPC may have published a minor revision; fetch the current PDF from europeanpaymentscouncil.eu to confirm version number and whether BIC remains optional.

---

## Sources

| Claim | Source | Confidence |
|-------|--------|------------|
| EPC QR field structure | EPC069-12 spec (training data from official EPC document) | HIGH |
| qrcode-generator API | npm package documentation (training data) | MEDIUM — verify version |
| qr-creator API | npm package documentation (training data) | MEDIUM — verify version |
| IBAN regex structure | ISO 13616 standard (training data) | HIGH |
| Amount format variations | Domain knowledge from Dutch invoicing conventions | MEDIUM |

**Verification tasks before build:**
1. Fetch https://www.europeanpaymentscouncil.eu to confirm current spec version and BIC-optional status
2. Check https://www.npmjs.com/package/qrcode-generator for latest version tag
3. Test QR code scannability with ING, Rabobank, ABN AMRO mobile apps (most important integration test)
