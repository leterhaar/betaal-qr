# Domain Pitfalls

**Domain:** EPC QR code generator with IBAN/payment text extraction
**Project:** Betaal QR
**Researched:** 2026-03-23
**Confidence:** MEDIUM — EPC069-12 spec and IBAN ISO 13616 are stable published standards covered in training data. Web verification was unavailable during research session; flag for spot-check before implementation.

---

## Critical Pitfalls

Mistakes that cause silent wrong payments, scannable-but-rejected QR codes, or fundamental rewrites.

---

### Pitfall 1: Wrong EPC QR Payload Encoding (UTF-8 vs Latin-1)

**What goes wrong:** The EPC069-12 specification requires the QR code payload to be encoded in UTF-8. Implementations that use the default JS string encoding without explicitly setting UTF-8 on the QR library call may produce codes that scan to garbled text for beneficiary names containing accented characters (e.g., "Société", "Müller GmbH").

**Why it happens:** Most QR code libraries default to whatever encoding is convenient. The developer tests with ASCII-only names ("Test BV") and never catches the problem.

**Consequences:** QR code scans to a payment with a corrupted beneficiary name. Some banking apps reject the payment; others silently accept it with a wrong name — a serious user trust issue.

**Prevention:**
- Explicitly configure the QR library to use UTF-8 byte encoding, not Latin-1 or ISO-8859-1.
- Test with at least one beneficiary name containing non-ASCII characters (e.g., "Société Générale", "Müller & Söhne").
- Verify the scanned text in a QR reader shows the exact characters entered.

**Detection:** Scan the generated QR with a plain QR reader (not a banking app) and compare raw text to the input. Garbled special characters are immediately visible.

**Phase:** Address in the QR generation implementation phase, before any UX work.

---

### Pitfall 2: EPC Payload Field Order and Line Termination Are Mandatory

**What goes wrong:** The EPC QR payload is newline-delimited with a strict field order (Service Tag, Version, Encoding, Function, BIC, Beneficiary Name, IBAN, Amount, Purpose, Remittance Reference, Remittance Text, Beneficiary to Originator Info). Swapping fields, omitting blank lines for optional fields, or using `\r\n` instead of `\n` causes banking apps to reject the QR entirely (no scan, or scan with error).

**Why it happens:** Developers reconstruct the format from blog posts or incomplete examples that omit the "empty line for missing optional field" rule.

**Consequences:** QR code is unscannable by ING/Rabobank/ABN AMRO. Only discovered during device testing, potentially late in development.

**Prevention:**
- Read EPC069-12 directly, not third-party summaries.
- The payload MUST be exactly 12 lines (some versions) — blank lines are required placeholders for omitted optional fields.
- Hardcode the template with all 12 positions and fill blanks explicitly.
- Use `\n` (LF only) not `\r\n` as the line separator.
- The Amount field must be formatted as `EUR` followed by a decimal with exactly 2 decimal places, no thousand separators (e.g., `EUR1234.56`, NOT `EUR1.234,56` or `EUR 1234.56`).

**Detection:** Paste the raw QR payload string into a SEPA QR validator tool. The European Payments Council and several Dutch bank developer portals provide these.

**Phase:** Address in QR generation implementation. Write a unit test that asserts exact payload string output character-by-character.

---

### Pitfall 3: Amount Parsing Locale Ambiguity (Dutch vs English Number Format)

**What goes wrong:** Dutch invoices use `1.234,56` (period as thousands separator, comma as decimal). English invoices and many software systems use `1,234.56`. A naive regex that grabs "the number near EUR/€" will misparse `€1.234,56` as `1234` (drops the decimal entirely) or as `1.234` (treats comma as decimal separator but loses the cents).

**Why it happens:** The developer tests with round amounts (`€500`, `€1200`) or English-format amounts and never tests Dutch locale invoices.

**Consequences:** User sends payment for wrong amount. This is a critical trust/correctness failure — the entire app's value is destroyed if it silently sends €1 instead of €1,234.56.

**Prevention:**
- Explicitly handle both formats: detect locale by checking which of `.` or `,` appears last in the number string (the last separator is the decimal separator).
- Algorithm: strip currency symbols → find all digits, commas, periods → if last separator is `,`, treat `,` as decimal; if last separator is `.`, treat `.` as decimal → convert to float.
- Test cases to cover: `1.234,56`, `1,234.56`, `1234.56`, `1.234`, `1,234`, `€ 1.234,56`, `EUR1234.56`, `1.234.567,89` (million+).
- Round to 2 decimal places before formatting for EPC payload.

**Detection:** Add a test suite with at least 8 amount format variants including Dutch locale large amounts.

**Phase:** Address in extraction/parsing phase. This is the highest-risk parsing component.

---

### Pitfall 4: IBAN Regex Matches Formatting Artifacts (Spaces, Hyphens)

**What goes wrong:** IBANs in invoice text often appear with spacing for readability: `NL91 ABNA 0417 1643 00`. A regex that matches only contiguous alphanumeric characters will fail to find this. Conversely, a regex that strips all whitespace from the entire document before matching may accidentally concatenate adjacent tokens into a false IBAN match.

**Why it happens:** Developer tests with a clean machine-generated IBAN and never tests with a scanned/formatted document.

**Consequences:** Extraction fails silently — the IBAN field is left blank and the user must type it manually (defeating the app's core value), OR a false IBAN is extracted.

**Prevention:**
- Normalize before matching: in the candidate region around IBAN-looking text, strip spaces and hyphens from token sequences before running IBAN validation.
- Pattern: `[A-Z]{2}\d{2}[\s-]?(?:[A-Z0-9]{4}[\s-]?)+` then strip non-alphanumeric before validating.
- Always validate the extracted IBAN using the ISO 13616 MOD-97 check digit algorithm — do not rely on regex alone.
- The MOD-97 check: rearrange IBAN (move first 4 chars to end), convert letters to digits (A=10…Z=35), compute mod 97 — result must equal 1.

**Detection:** Test with IBANs in the following forms: no spaces, spaces every 4 chars, hyphen-separated groups, mixed case.

**Phase:** Address in IBAN extraction phase. MOD-97 is ~10 lines of JS — always implement it.

---

### Pitfall 5: IBAN Checksum Validation Skipped — Wrong IBANs Pass Through

**What goes wrong:** Regex matches a string that looks like an IBAN (correct country code prefix, right length) but has incorrect check digits. Without MOD-97 validation, this invalid IBAN is silently used to generate the QR code.

**Why it happens:** Developers assume format-correct = valid. IBAN check digits catch transcription errors.

**Consequences:** User scans QR, banking app rejects payment with cryptic error ("invalid IBAN"), or worse — the IBAN happens to belong to a different account.

**Prevention:**
- Implement MOD-97 check digit validation as a hard requirement, not an optional feature.
- If multiple IBAN candidates are found, the one that passes MOD-97 wins.
- If none pass, show extraction failure rather than a bad IBAN.

**Detection:** Feed a deliberately wrong IBAN (`NL00ABNA0417164300` — wrong check digits) and verify the app rejects it.

**Phase:** Core implementation, not a polish item. Implement alongside IBAN regex.

---

### Pitfall 6: Beneficiary Name Extraction Corrupts or Truncates

**What goes wrong:** The EPC QR standard limits the beneficiary name field to 70 characters. Invoice text often contains full legal names with suffixes ("B.V.", "N.V.", "GmbH & Co. KG") that may be truncated. Worse, heuristic extraction may grab the wrong line — e.g., the sender's name instead of the recipient's.

**Why it happens:** Name extraction is heuristic (proximity to IBAN, labeled fields) and inherently ambiguous. The 70-char limit is easy to overlook.

**Consequences:** Banking app rejects QR (name too long), or payment is made with wrong beneficiary name (which may fail bank verification if name-check is enabled in the receiving bank).

**Prevention:**
- Always enforce the 70-character limit before including name in the QR payload — truncate at word boundary, not mid-character.
- Show the extracted name prominently in the review UI so users can spot wrong extraction.
- Support common Dutch label patterns: "t.n.v.", "ten name van", "Naam:", "Begunstigde:", "Ontvanger:".
- Support common English patterns: "Beneficiary:", "Pay to:", "Account Name:", "In favour of:".
- When label-based extraction fails, fall back to "line immediately before IBAN in document" — this is often the beneficiary name on Dutch invoices.
- Never silently truncate — if name exceeds 70 chars, show a warning so the user can edit.

**Detection:** Test with a beneficiary name of 85 characters. Verify the app warns and allows editing.

**Phase:** Extraction phase (heuristics), UI phase (review/edit step + truncation warning).

---

### Pitfall 7: BIC Field Handling — Optional in Version 002, Required in Version 001

**What goes wrong:** The EPC QR standard has two versions. Version 001 requires a BIC (SWIFT code); Version 002 (introduced in 2016 and now standard) makes BIC optional. Implementations that hardcode Version 001 require users to enter a BIC, or that omit the BIC line entirely from a Version 001 payload, produce invalid QR codes.

**Why it happens:** Many blog posts and Stack Overflow answers still document Version 001 format.

**Consequences:** QR codes rejected by modern banking apps, or extra unnecessary friction asking users for a BIC they don't have.

**Prevention:**
- Use Version 002 (`002` in the version field) — BIC is optional, leave the field blank.
- The Version 002 encoding value is `1` for UTF-8.
- Do not require or prompt for BIC from users.

**Detection:** Check the second line of the generated payload string — it should read `002`.

**Phase:** QR generation implementation. One-line fix but easy to get wrong from outdated docs.

---

### Pitfall 8: QR Code Density Too High for Banking App Scanners

**What goes wrong:** Long remittance references or long beneficiary names increase QR code density. Banking app QR scanners (phone cameras in a scanning UI) are not as forgiving as dedicated barcode scanners. Very dense QR codes fail to scan reliably.

**Why it happens:** Desktop QR readers scan everything fine. Mobile banking app scanners under real conditions (poor lighting, slight camera shake) fail on dense codes.

**Consequences:** The QR code is technically correct but unusable in practice — the entire app value is destroyed.

**Prevention:**
- Target QR error correction level M (15% recovery) not H (30%). Higher correction = more dense = harder to scan at a given size.
- Enforce the 70-char beneficiary name limit strictly — it has a direct impact on QR density.
- Limit the remittance reference/text fields in UI (the EPC standard caps these at 140 chars for structured reference, 140 chars for unstructured text — but aim for shorter).
- Render QR at minimum 200x200px, recommend 300x300px for on-screen display.
- Test by actually scanning with ING/Rabobank/ABN AMRO apps on a phone.

**Detection:** Physical device testing is the only reliable detection. Do this early, not at the end.

**Phase:** QR generation + integration testing. Set rendering parameters once; do device testing before declaring done.

---

## Moderate Pitfalls

---

### Pitfall 9: Multiple IBANs in Document — Wrong One Selected

**What goes wrong:** An invoice may contain both the sender's IBAN (for reference or return payments) and the beneficiary IBAN (the one to pay). Regex finds both and picks the first one, which may be the sender's.

**Prevention:**
- When multiple IBANs are found, highlight all candidates in the review UI rather than silently picking one.
- Prefer IBANs that appear near payment-trigger labels: "betaling", "betalen", "pay to", "IBAN:", "rekeningnummer".
- Let the user select the correct IBAN from discovered candidates if ambiguous.

**Phase:** Extraction phase (ranking logic) and UI phase (ambiguity resolution).

---

### Pitfall 10: Amount Regex Matches Non-Payment Numbers

**What goes wrong:** Documents contain many numbers — invoice numbers, dates, phone numbers, VAT rates (21%), line item quantities. The amount regex may match `21` (VAT percentage) or `2024` (year) as the payment amount.

**Prevention:**
- Anchor amount extraction to currency indicators: only match numbers preceded or followed within ~20 characters by `€`, `EUR`, `euro`, `te betalen`, `total`, `totaal`, `amount due`, `bedrag`.
- Prefer the last occurrence of a currency-anchored amount (invoice totals typically appear at the bottom).
- Exclude obviously wrong values: amounts below €0.01 or above €999,999,999.99 should be flagged.

**Phase:** Extraction phase.

---

### Pitfall 11: Copy-Paste Introduces Non-Breaking Spaces and Unicode Hyphens

**What goes wrong:** Text pasted from PDF viewers, email clients, or web pages often contains non-breaking spaces (`\u00A0`), soft hyphens (`\u00AD`), en-dashes (`\u2013`), or zero-width joiners. These characters break regex patterns that assume normal ASCII whitespace and hyphens.

**Prevention:**
- Normalize pasted input before any regex processing: replace non-breaking spaces with regular spaces, strip zero-width characters, normalize dashes to ASCII hyphen.
- One-liner normalization: `text.replace(/\u00A0/g, ' ').replace(/\u00AD/g, '').replace(/[\u2013\u2014]/g, '-')`.

**Phase:** First step in the extraction pipeline, before any pattern matching.

---

### Pitfall 12: EPC Amount Format — Trailing Zeros Required

**What goes wrong:** The EPC payload amount field must always include two decimal places: `EUR100.00`, not `EUR100` or `EUR100.0`. Some implementations use `parseFloat().toString()` which drops trailing zeros.

**Prevention:**
- Always format the amount using `.toFixed(2)` before constructing the payload.
- `"EUR" + amount.toFixed(2)` — never string-interpolate the raw float.

**Phase:** QR generation implementation. Write a unit test for round-number amounts (€100, €50, €1000).

---

## Minor Pitfalls

---

### Pitfall 13: Single-File HTML Breaks on Some Browser Security Policies

**What goes wrong:** When opened as a local `file://` URL, some browser Content Security Policy defaults or extensions may block inline `<script>` tags or CDN-loaded libraries. The app silently fails to load the QR library.

**Prevention:**
- Test opening `index.html` directly from the filesystem in Chrome, Firefox, and Edge.
- Prefer a CDN with a reliable SRI hash, or bundle the QR library inline in the HTML file (removes CDN dependency entirely).
- Add a visible fallback: if the QR library fails to load, show an error message rather than a blank output area.

**Phase:** Deployment/packaging phase.

---

### Pitfall 14: QR Code Not Rendered as Vector — Blurry When Scaled

**What goes wrong:** QR codes rendered to `<canvas>` at a fixed pixel size look blurry when the user zooms their browser or displays on high-DPI screens, making them harder for banking apps to scan.

**Prevention:**
- Render to an `<svg>` element if the chosen library supports it (qrcode.js and qr-creator both support SVG output).
- If using canvas, set canvas dimensions to `2x` the CSS size and use `devicePixelRatio` for retina rendering.

**Phase:** QR generation implementation.

---

### Pitfall 15: No User Feedback When Extraction Finds Nothing

**What goes wrong:** Regex finds no IBAN or amount. The form fields are left blank. The user doesn't understand whether the app failed or they need to type manually.

**Prevention:**
- Distinguish three states clearly in the UI: (a) extraction succeeded, (b) partial extraction (found IBAN but not amount), (c) extraction failed entirely.
- For (b) and (c), show explicit messages: "Could not find an amount — please enter it manually."
- Always allow manual entry/editing regardless of extraction outcome.

**Phase:** UI phase.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|----------------|------------|
| IBAN regex implementation | Formatted IBANs with spaces fail to match | Strip spaces before MOD-97 validation |
| IBAN regex implementation | No checksum validation — wrong IBANs silently pass | Always implement MOD-97 |
| Amount parsing | Dutch locale `1.234,56` misread as `1.234` | Detect last separator as decimal indicator |
| Amount parsing | Non-payment numbers matched (VAT, dates) | Anchor to currency labels |
| Beneficiary name extraction | Wrong name selected (sender vs recipient) | Proximity-to-IBAN heuristic + user review |
| Beneficiary name extraction | Name exceeds 70-char EPC limit | Enforce limit + warn user |
| QR payload construction | Wrong EPC version (001 vs 002) requiring BIC | Use Version 002, make BIC optional |
| QR payload construction | Wrong line terminator (`\r\n` vs `\n`) | Use `\n` only |
| QR payload construction | Amount not formatted as `EUR1234.56` | Use `.toFixed(2)` + `EUR` prefix |
| QR payload construction | UTF-8 encoding not set — accented names garbled | Explicitly set UTF-8 on QR library call |
| QR rendering | Dense QR fails mobile banking app scanners | Use error correction level M; test on device |
| Single-file deployment | CDN-loaded library blocked by browser CSP | Bundle library inline or add CSP fallback |
| Input normalization | Non-breaking spaces / Unicode dashes break regex | Normalize pasted text before all parsing |

---

## Confidence Notes

All pitfalls above are derived from:
- The EPC069-12 standard (stable, published by European Payments Council — version 002 finalized 2016)
- ISO 13616 IBAN standard (MOD-97 algorithm, country code/length table)
- General knowledge of Dutch invoice formatting conventions and banking app behavior

**HIGH confidence:** EPC payload format rules (field order, line terminators, amount format, version 002 BIC optionality, 70-char name limit, UTF-8 encoding requirement) — these are formally specified.

**HIGH confidence:** IBAN MOD-97 validation — formally specified in ISO 13616.

**MEDIUM confidence:** Dutch locale number format ambiguity, beneficiary name label patterns, QR scanner density limits on mobile banking apps — based on domain knowledge of Dutch business invoicing, not verified against current banking app behavior during this research session.

**LOW confidence:** Specific QR library behavior (UTF-8 defaults, SVG support) — verify against the chosen library's current documentation before implementing.

## Sources

- EPC069-12: "Quick Response Code — Guidelines to Enable the Data Capture for the Initiation of a SEPA Credit Transfer" (European Payments Council) — stable published standard, Version 6.0 (2019). Verify at: https://www.europeanpaymentscouncil.eu/document-library/guidance-documents
- ISO 13616-1: International Bank Account Number (IBAN) — MOD-97 algorithm specification
- Note: External URLs were not fetched during this research session (tool permissions unavailable). Findings are based on training data of the above standards. Spot-check the EPC069-12 specification PDF directly before implementing the payload builder.
