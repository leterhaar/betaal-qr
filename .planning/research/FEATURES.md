# Feature Landscape

**Domain:** Payment QR code generator (EPC / SEPA Credit Transfer)
**Researched:** 2026-03-23
**Confidence note:** Web access unavailable this session. EPC QR standard details drawn from training knowledge of EPC069-12 v3.0 (published 2022) — HIGH confidence on standard spec, MEDIUM on competitive landscape.

---

## EPC QR Code Standard: Required Fields

The EPC QR code (defined in EPC069-12, current version 3.0) encodes a SEPA Credit Transfer initiation. The payload is plain text, line-separated, with a strict field order.

### Field Specification

| Line | Field | Mandatory | Max Length | Format / Constraints |
|------|-------|-----------|------------|----------------------|
| 1 | Service Tag | YES | 3 chars | Always `BCD` |
| 2 | Version | YES | 3 chars | `001` or `002` (v002 adds instant payment) |
| 3 | Character Set | YES | 1 char | `1` = UTF-8, `2` = ISO 8859-1, etc. (1–8) |
| 4 | Identification | YES | 3 chars | Always `SCT` (SEPA Credit Transfer) |
| 5 | BIC of Beneficiary Bank | Conditional | 11 chars | BIC/SWIFT code. Mandatory in v001; optional in v002 |
| 6 | Beneficiary Name | YES | 70 chars | Free text |
| 7 | Beneficiary IBAN | YES | 34 chars | Valid IBAN (e.g. `NL91ABNA0417164300`) |
| 8 | Amount | Optional | 12 chars | `EUR` followed by amount, e.g. `EUR123.45`. Omit line to leave blank. |
| 9 | Purpose Code | Optional | 4 chars | ISO 20022 purpose code (e.g. `GDDS` for goods) |
| 10 | Remittance Reference | Optional | 35 chars | Structured: ISO 11649 creditor reference or legacy reference. Mutually exclusive with line 11. |
| 11 | Remittance Information | Optional | 140 chars | Unstructured free-text remittance. Mutually exclusive with line 10. |
| 12 | Originator Information | Optional | 70 chars | Information for the originator (payer). Not forwarded in the transaction. |

### Key Constraints
- Total payload must not exceed **331 bytes** (to stay within QR code capacity at error correction level M)
- QR code version must support the payload size — typically QR version 13–15 at ECC level M
- Amount field: if present, minimum `EUR0.01`, maximum `EUR999999999.99`. Value must have exactly 2 decimal places.
- BIC field: optional only in v002 (used in the EU since 2016 when BIC-free SEPA transfers became mandatory within EU)
- Lines must be present in order; optional fields omitted by leaving the line blank (empty line), not by removing it

### Practical Minimum for a Working QR Code
For Dutch banking apps (ING, Rabobank, ABN AMRO), the practical minimum is:
1. `BCD` (service tag)
2. `002` (version — BIC-optional)
3. `1` (UTF-8)
4. `SCT`
5. *(BIC — can be blank)*
6. Beneficiary name
7. Beneficiary IBAN
8. Amount (can be blank, but including it eliminates manual entry)

---

## Table Stakes

Features users expect in any payment QR code tool. Missing these makes the product feel broken or untrustworthy.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| IBAN input field | Core data — no QR without it | Low | Must validate IBAN checksum (mod97 algorithm) |
| Beneficiary name input | Required EPC field | Low | 70-char limit must be enforced |
| Amount input | Without it the payer must type the amount manually — defeats the purpose | Low | Must accept common formats (€1.234,56 and 1234.56 and EUR1234.56) and normalise to EURxxx.xx |
| QR code display | The actual deliverable | Low | Use qrcode.js or similar; must render at sufficient size (min 200x200px) |
| Input validation with clear errors | Payment data must be exact — wrong IBAN = lost money | Medium | IBAN checksum, amount range, name length. Errors shown inline, not alert dialogs |
| Review step before generating | Users need to confirm auto-extracted data is correct | Low | Editable fields; generate button only active when valid |
| Works in modern browsers without install | The entire value prop of a static HTML app | Low | No build step, no server, just open index.html |

---

## Differentiators

Features that set a tool apart. Not expected as baseline, but meaningfully valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Auto-extraction from pasted text | Core differentiator of this product — eliminates manual entry entirely | High | Regex for IBAN (high confidence), amount (medium confidence), name (low confidence — hardest field) |
| Structured remittance reference support | Invoice numbers as ISO 11649 creditor reference are machine-readable by bank systems | Medium | ISO 11649 check digit calculation is non-trivial; enables structured reconciliation |
| Unstructured remittance / payment description | "For invoice #123" in free text — most invoices include this | Low | Simple 140-char text field; very practical |
| Copy QR to clipboard | Faster than screenshot for digital workflows | Low | Canvas `toBlob()` + Clipboard API; not all browsers support it equally |
| Download QR as PNG | Lets users embed QR in reply emails or print it | Low | Canvas `toDataURL()` — straightforward |
| Persist last-used values in localStorage | Repeat payments to same beneficiary become near-instant | Low | Opt-in suggested for privacy; useful for accountants paying same suppliers |
| BIC field (visible but optional) | Some older Dutch bank apps or non-EU banks may still need it | Low | Hidden by default, expandable — don't clutter the main flow |
| Purpose code field | Needed for some B2B payment categories (e.g. salaries = SALA) | Low | Dropdown of common ISO 20022 purpose codes; niche but useful for payroll use |
| Dark/light mode | Personal tool — matches OS preference | Low | `prefers-color-scheme` media query |
| Print-friendly layout | Some users want a printed payment slip with QR | Low | CSS `@media print` |

---

## Anti-Features

Features to deliberately NOT build for this tool.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| AI/LLM extraction | Requires API key, costs money, sends payment data to third party, fails offline | Regex + heuristics only — sufficient for structured payment data |
| OCR / image/PDF upload | Adds significant complexity (Tesseract.js = 10MB+), handles edge cases poorly, out of scope for MVP | Text paste only; user copies text from PDF viewer |
| Server-side processing | Kills the zero-setup value prop; introduces privacy risk (payment data in transit) | All logic in browser |
| Payment execution / bank integration | Requires OAuth, PSD2 compliance, bank APIs — completely different product category | QR only; user's bank app handles the payment |
| Multi-payment batch mode | Complex UI, niche use case, distracts from core flow | Single payment per session; user can reload for next invoice |
| User accounts / history / cloud sync | Privacy risk, hosting cost, authentication complexity | localStorage for last-used values only (optional) |
| QR code styling (logo, colors, rounded corners) | "Fancy" QR codes can reduce scan reliability; payment QR should be maximally readable | Standard black-on-white QR only |
| Multiple QR standards (iDEAL, PayPal, Tikkie) | Scope creep; EPC covers the EU-wide use case | EPC QR only; other standards are separate products |
| Internationalisation / multiple languages | Dutch personal tool; English labels are fine | Dutch/English hybrid is acceptable |
| Mobile-optimised QR scanning from own screen | You can't scan a QR on your own screen with your own phone | Desktop-first; mobile is secondary use case |

---

## Feature Dependencies

```
IBAN input → IBAN validation → QR generation
Amount input → Amount normalisation → QR generation
Name input → Name length validation → QR generation
QR generation → QR display
QR display → Copy to clipboard (depends on Canvas API)
QR display → Download as PNG (depends on Canvas API)

Auto-extraction (paste) → IBAN regex → pre-fills IBAN input
Auto-extraction (paste) → Amount regex → pre-fills Amount input
Auto-extraction (paste) → Name heuristic → pre-fills Name input (lower confidence)

Remittance reference (structured) → ISO 11649 check digit calculation
Remittance reference (structured, unstructured) → mutually exclusive — only one per QR
```

---

## MVP Recommendation

The MVP is tightly scoped. Ship these:

1. **IBAN input + validation** — table stakes, required field
2. **Name input** — table stakes, required field
3. **Amount input + normalisation** — table stakes; without it the QR is barely useful
4. **QR code generation and display** — the deliverable
5. **Auto-extraction from pasted text** — the core differentiator; this is what makes the tool worth building
6. **Unstructured remittance field** — Low complexity, high practical value (invoice reference)
7. **Download QR as PNG** — Low complexity, makes the QR usable outside the browser

**Defer:**
- Structured remittance (ISO 11649): useful but adds non-trivial complexity; post-MVP
- Copy to clipboard: nice-to-have; fallback is screenshot
- BIC field: v002 makes it optional for all Dutch banks; hide unless needed
- localStorage persistence: post-MVP; validate core flow first
- Purpose code: niche; post-MVP

---

## Sources

- EPC069-12 "Quick Response Code: Guidelines to Enable the Data Capture for the Initiation of a SEPA Credit Transfer" v3.0 (2022) — training knowledge, HIGH confidence on field spec
- Dutch banking app EPC QR support: ING, Rabobank, ABN AMRO all support EPC QR v002 — HIGH confidence (well-established since 2016 BIC-free SEPA mandate)
- Competitive feature landscape (payment QR tools): MEDIUM confidence — derived from training knowledge, unverified against live tools as web access unavailable
