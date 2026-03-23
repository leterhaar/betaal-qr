# Requirements: Betaal QR

**Defined:** 2026-03-23
**Core Value:** Paste text, get a scannable payment QR code — no manual data entry into your bank app.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Extraction

- [ ] **EXTR-01**: User can paste invoice/email text into a text area
- [ ] **EXTR-02**: App extracts IBAN from pasted text using regex pattern matching
- [ ] **EXTR-03**: App extracts payment amount from pasted text (handles €1.234,56 / EUR 1234.56 / 1,234.56 formats)
- [ ] **EXTR-04**: App extracts counterparty name from pasted text using heuristics (proximity to IBAN, labeled fields like "t.n.v.", "begunstigde")

### Payment Fields

- [ ] **FLDS-01**: User can review and edit extracted IBAN, name, and amount in form fields
- [ ] **FLDS-02**: IBAN is validated using MOD-97 checksum with inline error display
- [ ] **FLDS-03**: Beneficiary name field enforces 70-character EPC limit
- [ ] **FLDS-04**: User can enter unstructured remittance info (free-text, max 140 chars)
- [ ] **FLDS-05**: User can optionally enter BIC/SWIFT code (hidden by default, expandable)

### QR Generation

- [ ] **QRCD-01**: App generates valid EPC QR code (v002, UTF-8, BIC optional) from confirmed fields
- [ ] **QRCD-02**: QR code displays at scannable size (min 200x200px)
- [ ] **QRCD-03**: App validates payload does not exceed 331-byte EPC limit before generating
- [ ] **QRCD-04**: QR code is scannable by Dutch banking apps (ING, Rabobank, ABN AMRO)
- [ ] **QRCD-05**: User can copy QR code image to clipboard

### General

- [ ] **GENL-01**: App runs as a single static HTML+JS file — open index.html in any modern browser
- [ ] **GENL-02**: App supports dark/light mode following OS color scheme preference
- [ ] **GENL-03**: All processing is client-side — no data leaves the browser

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Output Options

- **OUT-01**: User can download QR code as PNG image
- **OUT-02**: Print-friendly layout for payment slips

### Advanced Fields

- **ADVF-01**: Structured remittance reference support (ISO 11649 creditor reference)
- **ADVF-02**: Purpose code field (ISO 20022 dropdown — GDDS, SALA, etc.)

### Persistence

- **PERS-01**: Persist last-used values in localStorage for repeat payments

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| AI/LLM extraction | Requires API key, costs money, sends data to third party, fails offline |
| OCR / image/PDF upload | Adds significant complexity (Tesseract.js = 10MB+), poor edge case handling |
| Server-side processing | Kills zero-setup value prop; privacy risk |
| Payment execution / bank integration | Requires OAuth, PSD2, bank APIs — different product |
| Multi-payment batch mode | Complex UI, niche use case |
| User accounts / history / cloud sync | Privacy risk, hosting cost, auth complexity |
| QR code styling (logo, colors) | Reduces scan reliability; payment QR should be maximally readable |
| Multiple QR standards (iDEAL, PayPal, Tikkie) | Scope creep; EPC covers the EU-wide use case |
| Mobile-optimised scanning | Can't scan QR on own screen with own phone |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| EXTR-01 | Pending | Pending |
| EXTR-02 | Pending | Pending |
| EXTR-03 | Pending | Pending |
| EXTR-04 | Pending | Pending |
| FLDS-01 | Pending | Pending |
| FLDS-02 | Pending | Pending |
| FLDS-03 | Pending | Pending |
| FLDS-04 | Pending | Pending |
| FLDS-05 | Pending | Pending |
| QRCD-01 | Pending | Pending |
| QRCD-02 | Pending | Pending |
| QRCD-03 | Pending | Pending |
| QRCD-04 | Pending | Pending |
| QRCD-05 | Pending | Pending |
| GENL-01 | Pending | Pending |
| GENL-02 | Pending | Pending |
| GENL-03 | Pending | Pending |

**Coverage:**
- v1 requirements: 17 total
- Mapped to phases: 0
- Unmapped: 17 ⚠️

---
*Requirements defined: 2026-03-23*
*Last updated: 2026-03-23 after initial definition*
