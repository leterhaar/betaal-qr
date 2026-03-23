# Architecture Patterns

**Domain:** Static client-side payment QR code generator
**Researched:** 2026-03-23

---

## Recommended Architecture

A single HTML file with a linear pipeline: raw text in, structured payment data out, EPC QR code rendered.

```
[Paste Area] --> [Text Parser] --> [Field Editor] --> [EPC Formatter] --> [QR Renderer]
                     |                  |
              (IBAN, amount,     (user confirms /
               name extracted)    corrects fields)
```

No routing, no state management library, no build step. The entire app is one file with four logical layers embedded in `<script>` tags.

---

## Component Boundaries

| Component | Responsibility | Inputs | Outputs | Communicates With |
|-----------|---------------|--------|---------|-------------------|
| **Paste Area** | Accept raw text from user | Keyboard/clipboard events | Raw string | Text Parser (on input or button click) |
| **Text Parser** | Extract IBAN, amount, name from raw string | Raw string | ParsedPayment object `{iban, amount, name, currency}` | Field Editor |
| **Field Editor** | Display extracted fields; let user correct them | ParsedPayment object | ConfirmedPayment object | EPC Formatter (on confirm) |
| **EPC Formatter** | Assemble EPC QR payload string from confirmed fields | ConfirmedPayment object | EPC payload string | QR Renderer |
| **QR Renderer** | Generate and display QR code image | EPC payload string | `<canvas>` or `<img>` element | (terminal — user scans) |

---

## Data Flow

### Forward Path (happy path)

```
1. User pastes text into <textarea>
2. Text Parser runs synchronously on paste/button-click
   - IBAN regex: /\b[A-Z]{2}[0-9]{2}[A-Z0-9]{4}[0-9]{7}([A-Z0-9]?){0,16}\b/
   - Amount: multiple locale formats (€1.234,56 | 1,234.56 | EUR 1234.56)
   - Name: proximity heuristic — labeled field ("t.n.v.", "beneficiary:") or
           text block adjacent to IBAN
3. Parsed fields populate editable <input> elements
4. User reviews — corrects any wrong field — clicks "Generate"
5. EPC Formatter assembles payload per EPC069-12 v1.0:
     BCD\n001\n1\nSCT\n\n{name}\n{iban}\nEUR{amount}\n\n\n{remittance}\n
6. QR Renderer passes payload string to qrcode library, displays result
7. User scans QR with banking app
```

### Backward / Error Path

```
- Parser finds no IBAN → show inline warning, keep inputs empty, do not block
- Parser finds multiple IBANs → show all candidates, let user pick
- Amount ambiguous → leave amount field empty, user types manually
- EPC Formatter detects invalid field (IBAN checksum fail, amount > 999999999.99) → show field-level error, block generation
- QR Renderer failure → show error message below canvas
```

---

## Patterns to Follow

### Pattern 1: Pure Function Parser

**What:** `parsePaymentText(rawString) -> ParsedPayment` — no side effects, no DOM access.

**When:** Always. Parser logic must be independently testable and replaceable.

**Example:**
```javascript
function parsePaymentText(text) {
  return {
    iban:     extractIBAN(text),    // string | null
    amount:   extractAmount(text),  // number | null  (in euros, e.g. 1234.56)
    name:     extractName(text),    // string | null
    currency: 'EUR'
  };
}
```

Keeping the parser as a pure function means it can be unit-tested without a browser, and swapped or extended without touching the UI layer.

### Pattern 2: Strict EPC Payload Assembly

**What:** `formatEPCPayload(confirmed) -> string` — a single function that owns the EPC069-12 format.

**When:** Always. The EPC payload format has strict rules (line order, field lengths, encoding). Centralise this to a single function so any format fix applies everywhere.

**Example:**
```javascript
function formatEPCPayload({ iban, name, amount, remittance }) {
  // EPC QR Version 1, UTF-8, SEPA Credit Transfer
  return [
    'BCD',
    '001',
    '1',
    'SCT',
    '',                              // BIC — optional, omit for v1
    name.substring(0, 70),
    iban.replace(/\s/g, ''),
    'EUR' + amount.toFixed(2),
    '',                              // purpose
    '',                              // creditor reference
    remittance.substring(0, 140),   // unstructured remittance info
    ''                               // beneficiary info
  ].join('\n');
}
```

### Pattern 3: Validate Before Render

**What:** Run field validation (IBAN checksum, amount range, name length) before passing to QR Renderer.

**When:** Always — before every `Generate` click. Validation belongs between Field Editor and EPC Formatter, not inside either.

**Example:**
```javascript
function validateFields({ iban, name, amount }) {
  const errors = {};
  if (!isValidIBAN(iban))          errors.iban   = 'Ongeldige IBAN';
  if (!amount || amount <= 0)      errors.amount = 'Bedrag ongeldig';
  if (!name || name.trim() === '') errors.name   = 'Naam ontbreekt';
  return errors; // empty = valid
}
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Parsing Inside Event Handlers

**What:** Putting regex logic directly inside `textarea.addEventListener('input', ...)`.

**Why bad:** Untestable inline logic. Any change to extraction rules requires careful surgery inside event wiring. Parser and UI become one blob.

**Instead:** Event handler calls `parsePaymentText(textarea.value)` and applies the result to the DOM. Parser is a separate pure function.

### Anti-Pattern 2: Generating QR on Every Keystroke

**What:** Re-rendering the QR code every time any input field changes.

**Why bad:** QR library calls are synchronous and moderately expensive. Also produces flickering and confuses the user about when the code is "final".

**Instead:** QR is generated only when the user explicitly clicks "Generate" / "Maak QR". Field changes update input state only.

### Anti-Pattern 3: Embedding EPC Format Rules in the Renderer

**What:** Building the `BCD\n001\n...` string inside the click handler or QR renderer call.

**Why bad:** The EPC format is where bugs live. If it is scattered or duplicated, one wrong edit breaks compliance with banking apps silently.

**Instead:** `formatEPCPayload()` owns all format knowledge. Renderer receives only the final string.

### Anti-Pattern 4: Trusting Parser Output Directly

**What:** Piping `parsePaymentText()` output straight to `formatEPCPayload()` without a user review step.

**Why bad:** Payment amounts and IBANs extracted from free text will sometimes be wrong. Wrong QR = wrong payment. The review step is a safety gate, not a UX nicety.

**Instead:** Always route through the Field Editor (editable inputs) so the user confirms before generation.

---

## Component Build Order

Dependencies flow left-to-right. Build in this sequence:

```
1. EPC Formatter          (no deps — pure function, testable immediately)
2. Text Parser            (no deps — pure function, testable immediately)
3. QR Renderer            (depends on QR library via CDN only)
4. Field Editor UI        (depends on nothing except DOM)
5. Paste Area + wiring    (wires Parser → Field Editor)
6. Generate button wiring (wires Validator + Formatter → Renderer)
```

Steps 1-2 can be developed in any order. Steps 5-6 require 1-4 to exist.

**Why this order:**
- Formatter and Parser have no UI dependencies — develop and verify logic in isolation first.
- QR Renderer is a thin wrapper around the CDN library — confirm it works before building the form around it.
- Wiring (steps 5-6) is the last step because it only assembles already-working parts.

---

## Data Structures

### ParsedPayment (parser output, before user review)
```javascript
{
  iban:     string | null,   // raw extracted string, spaces included
  amount:   number | null,   // decimal euros, e.g. 1234.56
  name:     string | null,   // extracted candidate name
  currency: 'EUR'
}
```

### ConfirmedPayment (after user review, before generation)
```javascript
{
  iban:       string,   // normalised: uppercase, no spaces
  amount:     number,   // decimal euros
  name:       string,   // max 70 chars per EPC spec
  remittance: string,   // free text reference, max 140 chars (may be empty)
  currency:   'EUR'
}
```

### EPC Payload (string, passed to QR library)
```
BCD
001
1
SCT

{name}
{iban}
EUR{amount}


{remittance}

```
12 lines, `\n` separated, UTF-8 encoded. Line 5 (BIC) intentionally empty for v1.

---

## Scalability Considerations

This is a personal local tool — scale is not a concern. The relevant dimension is maintainability.

| Concern | Approach |
|---------|----------|
| Adding new extraction rules | Pure parser function — add regex/heuristic, return same shape |
| Supporting other currencies | ConfirmedPayment.currency field already present; EPC formatter line 8 already uses it |
| Handling edge cases (multiple IBANs) | Parser returns array of candidates; Field Editor renders a picker — no architectural change |
| Internationalisation | Labels are in Dutch strings in HTML only — swap string constants, no logic change |

---

## File Structure

Single file is acceptable and recommended given the constraints. Internal structure within `index.html`:

```
index.html
  <style>          — layout and visual feedback styles
  <body>
    #paste-area    — <textarea> for raw input
    #fields        — <input> elements for IBAN, name, amount, reference
    #generate-btn  — trigger button
    #qr-output     — <canvas> or <div> for QR image
    #error-display — inline validation messages
  <script src="qrcode.min.js">    — QR library (CDN or local fallback)
  <script>
    // 1. parsePaymentText(text) -> ParsedPayment
    // 2. extractIBAN / extractAmount / extractName helpers
    // 3. validateIBAN(iban) -> boolean
    // 4. formatEPCPayload(confirmed) -> string
    // 5. validateFields(confirmed) -> errors object
    // 6. renderQR(payload) -> void
    // 7. DOM wiring — event listeners only, no logic
```

Keep functions in this order so each function is defined before the wiring that calls it.

---

## Sources

- EPC069-12 "Quick Response Code — Guidelines to Enable the Data Capture for the Initiation of a SEPA Credit Transfer" (European Payments Council) — defines payload format, field lengths, encoding requirements. Version 1.0 is the widely deployed version supported by Dutch banking apps.
- IBAN structure: ISO 13616-1 — 2-letter country code + 2 check digits + BBAN, max 34 chars total.
- QR code rendering: qrcode.js / qrcodejs (MIT) or qr-creator — both well-established vanilla JS libraries loadable from CDN with no build step required.
- Confidence: HIGH for EPC payload format (stable standard, unchanged since 2016). HIGH for IBAN regex structure. MEDIUM for name extraction heuristics (no formal standard — project-specific judgment).
