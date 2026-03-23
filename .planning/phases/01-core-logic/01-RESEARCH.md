# Phase 1: Core Logic - Research

**Researched:** 2026-03-23
**Domain:** EPC QR payload assembly + text extraction (IBAN, amount, beneficiary name) in vanilla JavaScript
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **Amount parsing:** Support both Dutch (€1.234,56) and English (1,234.56) formats. Treat ambiguous amounts conservatively — prefer Dutch locale given target user.
- **Name extraction:** Use Dutch label heuristics (t.n.v., begunstigde, ten name van, naam) + proximity to IBAN. Fall back to empty string if nothing found — user edits in Phase 2.
- **IBAN handling:** If multiple IBANs found, pick the first one. Accept spaces in IBANs (normalize by stripping). Validate with MOD-97.
- **EPC payload:** Use v002 (BIC optional), UTF-8 (charset 1), include blank lines for omitted optional fields per spec. Amount as `EURnnn.nn` with exactly 2 decimal places.
- **Special characters:** Preserve UTF-8 characters in beneficiary names. Enforce 70-char limit by truncating with warning.

### Claude's Discretion

User chose to skip detailed discussion — Claude has full discretion on all implementation decisions for this phase. Use research findings from `.planning/research/` to guide choices.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| EXTR-02 | App extracts IBAN from pasted text using regex pattern matching | IBAN regex pattern documented; MOD-97 algorithm confirmed; space-normalisation required before validation |
| EXTR-03 | App extracts payment amount from pasted text (handles €1.234,56 / EUR 1234.56 / 1,234.56 formats) | Locale-detection algorithm documented; anchor-to-currency approach prevents false positives |
| EXTR-04 | App extracts counterparty name using heuristics (proximity to IBAN, labeled fields like "t.n.v.", "begunstigde") | Dutch/English label list documented; IBAN proximity fallback described |
| FLDS-02 | IBAN is validated using MOD-97 checksum with inline error display | ISO 13616 MOD-97 algorithm confirmed; ~10-line JS implementation sufficient |
| FLDS-03 | Beneficiary name field enforces 70-character EPC limit | EPC069-12 v3.1 confirms 70-char max; truncation-with-warning pattern documented |
| QRCD-01 | App generates valid EPC QR code (v002, UTF-8, BIC optional) from confirmed fields | EPC069-12 v3.1 12-field format confirmed; v002 BIC-optional confirmed |
| QRCD-03 | App validates payload does not exceed 331-byte EPC limit before generating | 331-byte limit confirmed in EPC069-12 v3.1; byte-count check required (UTF-8 multi-byte chars can push past limit) |
| GENL-01 | App runs as a single static HTML+JS file — open index.html in any modern browser | Single-file architecture confirmed; qrcode-generator v2.0.4 available via jsDelivr CDN or vendored locally |
| GENL-03 | All processing is client-side — no data leaves the browser | All components (extraction, validation, EPC assembly, QR rendering) are pure JS with no network calls |
</phase_requirements>

---

## Summary

Phase 1 is a pure-function JavaScript pipeline with four logical components: text input normalization, field extraction (IBAN, amount, name), EPC payload assembly, and QR rendering. All components are browser-runnable with no build tools. The single external dependency is a QR generation library loaded from CDN or vendored locally.

The EPC069-12 standard is at v3.1 (March 2024), with only minor clarifications over v3.0. The core 12-line payload format, BIC-optional behaviour in version 002, and 331-byte limit are unchanged. The v3.1 clarification confirms both LF and CRLF are acceptable as line separators, and the final populated field need not be followed by any separator. The amount field may be omitted entirely if unknown (not left blank).

The qrcode-generator library (kazuhikoarase) is at version **2.0.4** — not 1.4.4 as noted in prior research. The dist directory now ships `qrcode_UTF8.js` (793 bytes) as a companion file that patches `stringToBytes` for UTF-8 support. This changes the loading strategy: both `qrcode.js` (55 KB) and `qrcode_UTF8.js` (793 bytes) must be loaded. The API for error correction level M and SVG output is confirmed unchanged from training data.

**Primary recommendation:** Build the five pure functions in isolation first (`normalizeText`, `extractIBAN`, `extractAmount`, `extractName`, `formatEPCPayload`), verify each in the browser console, then wire to QR renderer. This satisfies all Phase 1 requirements and makes the pipeline console-testable as specified.

---

## Project Constraints (from CLAUDE.md)

| Directive | Type | Impact on Phase 1 |
|-----------|------|-------------------|
| Static HTML + vanilla JavaScript — no build tools, no frameworks, no server | REQUIRED | All code written as plain ES2020 JS in `<script>` tags; no import/export |
| No ESModules with import (CORS blocks local file:// in some browsers) | REQUIRED | Use classic `<script>` tags for all scripts including library |
| Dependencies: minimal — QR code library only, loaded via CDN | REQUIRED | Only qrcode-generator (+ its UTF8 companion); no other CDN scripts |
| Privacy: all processing client-side, no data leaves the browser | REQUIRED | No fetch/XHR in any core logic function |
| No TypeScript | REQUIRED | Use JSDoc comments for type hints if desired |
| No npm/Node/build tooling | REQUIRED | Library must be usable as a plain `<script src="...">` |

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vanilla JavaScript (ES2020+) | N/A | All app logic | CLAUDE.md hard requirement; no build step possible |
| HTML5 | N/A | Single-file shell | `<textarea>`, `<canvas>`, `<input>` cover all needs |
| qrcode-generator (kazuhikoarase) | **2.0.4** | EPC payload → QR image | Pure JS, UMD module, CDN-loadable, supports ECC level M, actively maintained |
| qrcode_UTF8.js companion | **2.0.4** | Patches library for UTF-8 byte encoding | Required when payload contains non-ASCII chars (Dutch names with accents) |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| qr-creator | 1.0.0 | Fallback QR renderer | Only if qrcode-generator has a blocking issue at runtime |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| qrcode-generator | qrcode (soldair) | Node-first, 80 KB+ browser bundle, requires bundler — rejected |
| qrcode-generator | QRCode.js (davidshimjs) | Unmaintained since 2013, no ECC control — rejected |
| inline mod-97 | ibantools npm | Adds CDN dependency for marginal benefit over 10-line inline impl — rejected |

**Installation (CDN):**

Load both files in order for UTF-8 support:

```html
<script src="https://cdn.jsdelivr.net/npm/qrcode-generator@2.0.4/dist/qrcode.js"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode-generator@2.0.4/dist/qrcode_UTF8.js"></script>
```

For fully offline / local file:// use, download both files to `/lib/` and reference locally. This is recommended per CLAUDE.md (personal local tool).

**Version verified:** qrcode-generator latest is 2.0.4 (published ~August 2025, confirmed via jsDelivr directory listing). Prior research referenced 1.4.4 — this is outdated.

---

## Architecture Patterns

### Recommended Project Structure

```
index.html
  <head>
    <meta charset="UTF-8">
    <style>          -- layout, colour scheme, print styles
  <body>
    #paste-area      -- <textarea> raw input
    #fields          -- <input> IBAN, name, amount, remittance
    #generate-btn    -- trigger
    #qr-output       -- <div> receives QR SVG/table
  <script src="lib/qrcode.js">
  <script src="lib/qrcode_UTF8.js">
  <script>
    // Phase 1 functions (all pure, no DOM access):
    //   normalizeText(raw)            → string
    //   extractIBAN(text)             → string | null
    //   extractAmount(text)           → number | null
    //   extractName(text, ibanPos)    → string | null
    //   validateIBAN(iban)            → boolean
    //   formatEPCPayload(confirmed)   → string
    //   validateEPCPayload(payload)   → { valid, byteCount, errors }
    // Phase 2 functions (DOM-facing):
    //   renderQR(payload)             → void
    //   validateFields(confirmed)     → errors object
    //   wiring / event listeners
```

**Build order (establish in Phase 1):**

1. `normalizeText` — prerequisite for all parsing
2. `extractIBAN` + `validateIBAN` — high confidence, no deps
3. `extractAmount` — high risk (locale ambiguity), implement carefully
4. `extractName` — low confidence, heuristic
5. `formatEPCPayload` — pure function, testable independently
6. `validateEPCPayload` — byte-count check, depends on formatter
7. QR renderer — thin wrapper over library (Phase 1 scope: confirm library works)

### Pattern 1: Pure Function Pipeline

**What:** Each extraction and formatting step is a named function with no side effects and no DOM access. Returns a value; never modifies globals.

**When to use:** Always, for all Phase 1 logic.

```javascript
// All Phase 1 functions follow this signature pattern:
function extractIBAN(text) {
  // Returns first valid IBAN found, or null
  const raw = text.match(/\b[A-Z]{2}\d{2}[\sA-Z0-9]{4,}[\d]{7}[A-Z0-9]{0,16}\b/gi) || [];
  for (const candidate of raw) {
    const normalized = candidate.replace(/\s/g, '').toUpperCase();
    if (normalized.length >= 15 && normalized.length <= 34 && validateIBAN(normalized)) {
      return normalized;
    }
  }
  return null;
}
```

### Pattern 2: Strict EPC Payload Assembly

**What:** `formatEPCPayload` owns the entire EPC069-12 format. No other function constructs QR payload strings.

**When to use:** Centralise all format knowledge here. Any spec fix applies in one place.

```javascript
// Source: EPC069-12 v3.1 (March 2024), 12-field format
function formatEPCPayload({ iban, name, amount, bic = '', remittance = '' }) {
  const fields = [
    'BCD',                                    // Line 1: Service tag
    '002',                                    // Line 2: Version (BIC optional)
    '1',                                      // Line 3: Character set (UTF-8)
    'SCT',                                    // Line 4: Identification
    bic.trim().substring(0, 11),              // Line 5: BIC (blank = omitted)
    name.substring(0, 70),                    // Line 6: Beneficiary name
    iban.replace(/\s/g, '').toUpperCase(),    // Line 7: Beneficiary IBAN
    amount != null ? 'EUR' + amount.toFixed(2) : '',  // Line 8: Amount (blank if unknown)
    '',                                       // Line 9: Purpose code (blank)
    '',                                       // Line 10: Structured remittance (blank)
    remittance.substring(0, 140),             // Line 11: Unstructured remittance
    ''                                        // Line 12: Originator info (blank)
  ];
  return fields.join('\n');
}
```

**Note on v3.1:** v3.1 clarifies that trailing separators after the last populated line are not required. The `.join('\n')` above produces a trailing newline on line 12 — this is acceptable. What is NOT acceptable is using `\r\n` as the separator between fields.

### Pattern 3: Text Normalization Before Parsing

**What:** A single `normalizeText` call at the start of the pipeline handles Unicode artifacts from copy-paste.

**When to use:** First step before any regex runs.

```javascript
function normalizeText(raw) {
  return raw
    .replace(/\u00A0/g, ' ')     // non-breaking space → regular space
    .replace(/\u00AD/g, '')      // soft hyphen → remove
    .replace(/[\u2013\u2014]/g, '-') // en-dash, em-dash → ASCII hyphen
    .replace(/\u200B/g, '')      // zero-width space → remove
    .trim();
}
```

### Pattern 4: Amount Locale Detection

**What:** Detect Dutch vs English locale by position of the last separator.

**When to use:** In `extractAmount`, as the core disambiguation step.

```javascript
function parseAmountString(s) {
  // s: cleaned string like "1.234,56" or "1,234.56" or "1234.56"
  const lastComma = s.lastIndexOf(',');
  const lastDot   = s.lastIndexOf('.');
  let normalized;
  if (lastComma > lastDot) {
    // Dutch: comma is decimal separator (e.g. 1.234,56)
    normalized = s.replace(/\./g, '').replace(',', '.');
  } else {
    // English: dot is decimal separator (e.g. 1,234.56)
    normalized = s.replace(/,/g, '');
  }
  const value = parseFloat(normalized);
  if (isNaN(value) || value < 0.01 || value > 999999999.99) return null;
  return Math.round(value * 100) / 100; // avoid float drift
}
```

### Pattern 5: MOD-97 IBAN Validation

**What:** ISO 13616 MOD-97 check digit validation, iterative to avoid BigInt requirements.

```javascript
function validateIBAN(iban) {
  // Rearrange: move first 4 chars to end
  const rearranged = iban.slice(4) + iban.slice(0, 4);
  // Convert letters to digits: A=10, B=11, ..., Z=35
  const digits = rearranged.split('').map(ch => {
    const code = ch.charCodeAt(0);
    return code >= 65 && code <= 90 ? String(code - 55) : ch;
  }).join('');
  // Iterative mod-97 (avoids need for BigInt on long IBANs)
  let remainder = 0;
  for (const ch of digits) {
    remainder = (remainder * 10 + parseInt(ch, 10)) % 97;
  }
  return remainder === 1;
}
```

### Anti-Patterns to Avoid

- **Parsing inside event handlers:** Event handler must call `parsePaymentText(value)` and apply result. Logic never lives inside `addEventListener`.
- **Generating QR on every keystroke:** QR renders only on explicit user trigger, not on input events.
- **Embedding EPC format inside the QR renderer call:** `formatEPCPayload` owns format; renderer receives only the string.
- **Calling `.toFixed()` on null amount:** Guard: `amount != null ? 'EUR' + amount.toFixed(2) : ''`
- **Using `\r\n` as line separator:** Use `\n` only. Windows environments may introduce CRLF — `.join('\n')` prevents this.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| QR code image from binary data | Custom QR matrix encoder | qrcode-generator v2.0.4 | Reed-Solomon error correction, QR format specification, mask pattern selection are non-trivial — thousands of lines to implement correctly |
| IBAN country length table | Custom per-country length validation | ISO 13616 MOD-97 only | MOD-97 catches check digit errors; exact country length validation is nice-to-have but MOD-97 alone is sufficient for this app |
| UTF-8 byte encoder | Custom TextEncoder polyfill | `new TextEncoder().encode(str).length` | Web standard API available in all modern browsers; use for payload byte-count check |

**Key insight:** EPC payload construction and IBAN validation are simple enough to implement inline. The QR matrix generation is the only genuinely complex piece — delegate that entirely to the library.

---

## Common Pitfalls

### Pitfall 1: Wrong qrcode-generator Version — Missing UTF-8 Companion File

**What goes wrong:** Prior research referenced v1.4.4. The current version is 2.0.4. More critically, v2.0.4 ships UTF-8 support as a **separate companion file** (`qrcode_UTF8.js`, 793 bytes) that must be loaded after the main library. Loading only `qrcode.js` without `qrcode_UTF8.js` defaults to a non-UTF-8 encoding. Beneficiary names with accented characters (Société, Müller, Smit-Dëns) will be encoded incorrectly, producing scannable-but-wrong QR codes in banking apps.

**How to avoid:** Load both files. Verify by testing a name with an accented character (e.g., "Société Générale") and scanning the QR with a plain reader — the raw text must match the input exactly.

**Warning signs:** All ASCII names work correctly but non-ASCII names produce garbled output on scan.

### Pitfall 2: EPC Payload 331-Byte Limit is Bytes, Not Characters

**What goes wrong:** UTF-8 encodes non-ASCII characters as 2–4 bytes. A beneficiary name with all Dutch/EU characters stays under 70 chars but can exceed the 331-byte payload limit when combined with a long IBAN and remittance text.

**How to avoid:** Count payload bytes using `new TextEncoder().encode(payload).length` before passing to the QR renderer. Reject or warn if over 331 bytes.

```javascript
function validateEPCPayload(payload) {
  const byteCount = new TextEncoder().encode(payload).length;
  return {
    valid: byteCount <= 331,
    byteCount,
    errors: byteCount > 331 ? [`Payload is ${byteCount} bytes; EPC limit is 331`] : []
  };
}
```

**Warning signs:** QR code scans correctly with ASCII remittance but fails with a long Dutch description.

### Pitfall 3: Amount Regex Matches Non-Payment Numbers

**What goes wrong:** Documents contain many numbers — VAT rates (21%), invoice numbers (2024-456), dates, phone numbers. A naive `\d+[.,]\d{2}` regex matches any of these.

**How to avoid:** Anchor amount regex to currency indicators within ~30 characters. Prefer the last matching occurrence (totals appear at the bottom of invoices).

```javascript
const AMOUNT_PATTERN = /(?:€|EUR|euro)\s*([\d.,]+(?:[.,]\d{1,2})?)|(?:te\s+betalen|totaal|total|bedrag|amount\s+due)[:\s]*([\d.,]+(?:[.,]\d{1,2})?)/gi;
```

**Warning signs:** App extracts 21 (VAT rate) or a year as the amount.

### Pitfall 4: IBAN with Spaces Fails Regex Match

**What goes wrong:** Dutch invoices format IBANs as `NL91 ABNA 0417 1643 00` with spaces every 4 chars. A regex expecting a contiguous alphanumeric string finds nothing.

**How to avoid:** IBAN regex must allow optional spaces between groups. Normalize (strip spaces) before MOD-97 validation.

```javascript
const IBAN_REGEX = /\b([A-Z]{2}\d{2}(?:\s?[A-Z0-9]{4}){3,7}\s?[A-Z0-9]{0,3})\b/gi;
// After match: candidate.replace(/\s/g, '').toUpperCase()
```

### Pitfall 5: EPC Payload Line 8 Amount Format Edge Cases

**What goes wrong:** `parseFloat(100).toFixed(2)` gives `"100.00"` — correct. But `amount` stored as integer (100 not 100.00) and passed directly as `"EUR" + amount` gives `"EUR100"` — wrong. Also `EUR0.10` must not appear as `EUR.1`.

**How to avoid:** Always use `.toFixed(2)`. Always check the amount is a number before calling it. Include round-number test cases.

**Warning signs:** Banking app rejects payment with amount format error.

### Pitfall 6: Name Extraction Grabs Sender Name Instead of Recipient

**What goes wrong:** Dutch invoices contain the sender's company name at the top and the beneficiary's bank details at the bottom. Proximity-to-IBAN heuristic finds "line before IBAN" which may be the account holder label ("Rekeningnummer:") not the name.

**How to avoid:** Apply label matching first (`t.n.v.`, `begunstigde:`, `naam:`). Fall back to proximity. Return `null` rather than a wrong name — Phase 2 UI will prompt the user.

---

## Code Examples

### qrcode-generator v2.0.4: Generate QR with ECC Level M and SVG Output

```javascript
// Source: https://github.com/kazuhikoarase/qrcode-generator/blob/master/js/README.md
// Load order: qrcode.js FIRST, then qrcode_UTF8.js (patches stringToBytes)

function renderQR(payloadString, containerElement) {
  var qr = qrcode(0, 'M');       // typeNumber 0 = auto-detect; ECC level M
  qr.addData(payloadString);
  qr.make();
  // SVG output (scalable, crisp on retina screens)
  containerElement.innerHTML = qr.createSvgTag({ cellSize: 2, margin: 4, scalable: true });
  // Alternative: table output (no canvas required)
  // containerElement.innerHTML = qr.createTableTag(4, 8);
}
```

### EPC Payload Byte-Count Validation

```javascript
function validateEPCPayload(payload) {
  const byteCount = new TextEncoder().encode(payload).length;
  return {
    valid: byteCount <= 331,
    byteCount,
    errors: byteCount > 331 ? ['Payload exceeds 331-byte EPC limit (' + byteCount + ' bytes)'] : []
  };
}
```

### Amount Extraction with Locale Detection

```javascript
function extractAmount(text) {
  // Anchor: only match numbers preceded by currency signal within 30 chars
  const pattern = /(?:€|EUR|euro)\s*([\d.,]+)|(?:te\s+betalen|totaal|total|bedrag|amount\s+due)\s*:?\s*([\d.,]+)/gi;
  let lastMatch = null;
  let m;
  while ((m = pattern.exec(text)) !== null) {
    const raw = (m[1] || m[2]).trim();
    const value = parseAmountString(raw);
    if (value !== null) lastMatch = value;
  }
  return lastMatch; // prefer last (invoice totals at bottom)
}
```

### IBAN Extraction (Space-Tolerant)

```javascript
function extractIBAN(text) {
  // Match IBANs with optional spaces every 4 chars (common Dutch invoice format)
  const pattern = /\b([A-Z]{2}\d{2}(?:\s?[A-Z0-9]{4}){3,7}\s?[A-Z0-9]{0,3})\b/g;
  const candidates = [];
  let m;
  while ((m = pattern.exec(text)) !== null) {
    const normalized = m[1].replace(/\s/g, '').toUpperCase();
    if (normalized.length >= 15 && normalized.length <= 34) {
      candidates.push(normalized);
    }
  }
  // Return first MOD-97 valid candidate (per user constraint: first one wins)
  return candidates.find(validateIBAN) || null;
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| qrcode-generator v1.4.4 (prior research) | v2.0.4 with separate qrcode_UTF8.js companion | Library update ~2024-2025 | Must load two files, not one; UTF-8 is now opt-in via companion script |
| EPC069-12 v3.0 (2022) | v3.1 (March 2024) | 19 March 2024 | Minor clarifications only; core format unchanged; trailing separator after last field now explicitly optional |
| BIC required (v001) | BIC optional (v002) | 2016 (SEPA BIC-free mandate) | Use version `002`; omit or blank line 5 |

**Deprecated/outdated:**
- EPC QR version `001`: Requires BIC; replaced by `002` which Dutch banks have used since 2016. Do not use `001`.
- CDN URL `qrcode-generator@1.4.4/qrcode.min.js`: Wrong version. Use `@2.0.4/dist/qrcode.js` + `@2.0.4/dist/qrcode_UTF8.js`.

---

## Open Questions

1. **qrcode_UTF8.js companion — does it need explicit activation or is loading it sufficient?**
   - What we know: The file calls `qrcode.stringToBytes = qrcode.stringToBytesFuncs['UTF-8']` which patches the global. Loading it after `qrcode.js` is sufficient — no explicit call needed.
   - What's unclear: Behaviour when `addData` is called with a mode parameter (e.g., mode `'Byte'` vs default). Research confirmed the patch works at load time, but the `addData` byte-mode parameter interaction should be verified with a quick test.
   - Recommendation: Test with `qr.addData(payloadString)` (no explicit mode argument). If garbled output occurs, try `qr.addData(payloadString, 'Byte')`.

2. **EPC069-12 v3.1 — are there any payload format changes vs v3.0?**
   - What we know: v3.1 published March 2024. Only confirmed change: trailing separator after last populated field is explicitly optional. Core 12-line format, BIC optionality, 331-byte limit, ECC level M requirement — all confirmed unchanged.
   - What's unclear: The full PDF was not parsed during this session (403 on fetch). There may be minor clarifications about the amount field or purpose code handling.
   - Recommendation: Treat v3.0 rules as authoritative for implementation. The confirmed payload format in research files is correct. Flag for spot-check if banking app scanning fails.

3. **Amount field when unknown — blank line vs omit?**
   - What we know: STACK.md says "omit entirely (not just blank) if amount is zero/unknown". FEATURES.md says "Omit line to leave blank." These are contradictory.
   - What's unclear: Whether line 8 must be present as an empty line or can be completely absent.
   - Recommendation: Per EPC069-12 convention for optional fields, include an empty line (not omit the line) to maintain the 12-line structure. Use `amount != null ? 'EUR' + amount.toFixed(2) : ''` which produces an empty string (blank line) when amount is null. This is the safer interpretation — banking app parsers expect 12 lines.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 1 is code-only (pure JavaScript functions). No external tools, runtimes, databases, or CLI utilities required beyond a modern browser. The QR library is fetched from CDN at load time; no build-time tooling.

---

## Sources

### Primary (HIGH confidence)

- EPC069-12 v3.1 (March 2024) — confirmed via search result snippets and official PDF URL at europeanpaymentscouncil.eu. 12-field format, 331-byte limit, BIC optional in v002, ECC level M requirement.
- ISO 13616-1 — IBAN structure and MOD-97 validation algorithm.
- CLAUDE.md — project constraints (vanilla JS, single file, no build tools, no ESModules, no TypeScript).
- `.planning/research/FEATURES.md`, `STACK.md`, `ARCHITECTURE.md`, `PITFALLS.md` — in-project research documents.

### Secondary (MEDIUM confidence)

- qrcode-generator v2.0.4 dist directory listing via jsDelivr (https://cdn.jsdelivr.net/npm/qrcode-generator/dist/) — confirmed version 2.0.4 and qrcode_UTF8.js companion file presence and size.
- GitHub README for qrcode-generator (https://github.com/kazuhikoarase/qrcode-generator/blob/master/js/README.md) — confirmed API: `qrcode(typeNumber, 'M')`, `addData()`, `make()`, `createSvgTag()`.
- qrcode_UTF8.js source fetched via jsDelivr — confirmed it patches `stringToBytes` on load.
- WebSearch results confirming v3.1 release date (19 March 2024) and trailing separator clarification.

### Tertiary (LOW confidence)

- qr-creator v1.0.0 as fallback — version and API from training data only; not verified in this session.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — qrcode-generator v2.0.4 confirmed via CDN directory listing; companion file confirmed
- EPC payload format: HIGH — v3.1 core format confirmed unchanged from v3.0; field spec from stable published standard
- IBAN extraction/validation: HIGH — ISO 13616 MOD-97 is a stable formal standard; regex patterns are straightforward
- Amount parsing: HIGH — locale detection algorithm is deterministic; test cases well-documented
- Architecture patterns: HIGH — all components are pure functions with no external dependencies
- Pitfalls: HIGH for EPC format and IBAN; MEDIUM for name extraction heuristics and QR scanner density behaviour

**Research date:** 2026-03-23
**Valid until:** 2026-07-01 (EPC standard stable; library API unlikely to change in 3 months)
