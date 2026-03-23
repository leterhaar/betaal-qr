# Betaal QR

## What This Is

A static web app (single HTML file) that extracts payment details from pasted invoice or email text — IBAN, counterparty name, and amount — and generates an EPC QR code scannable by any European banking app. Runs locally in the browser with no server or API keys required.

## Core Value

Paste text, get a scannable payment QR code — no manual data entry into your bank app.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] User can paste invoice/email text into a text area
- [ ] App extracts IBAN from pasted text using regex pattern matching
- [ ] App extracts payment amount from pasted text using pattern matching
- [ ] App extracts counterparty name from pasted text using heuristics (text near IBAN, labeled fields)
- [ ] User can review and edit extracted IBAN, name, and amount before QR generation
- [ ] App generates a valid EPC QR code from the confirmed payment details
- [ ] QR code is scannable by standard European banking apps (ING, Rabobank, ABN AMRO, etc.)
- [ ] App runs as a static HTML+JS page — just open index.html in a browser

### Out of Scope

- AI/LLM-based extraction — using regex/rules only, no API keys or costs
- OCR / image upload — text input only, no screenshot or PDF parsing
- Server-side processing — everything runs client-side in the browser
- Multi-user / authentication — personal local tool
- Payment execution — only generates QR codes, doesn't send money

## Context

- EPC QR code standard (EPC069-12) defines the format for SEPA credit transfer QR codes
- IBANs have a predictable format: 2 letter country code + 2 check digits + up to 30 alphanumeric chars (e.g., NL91ABNA0417164300)
- Amounts in invoices appear in various formats: €1.234,56 or 1,234.56 or EUR 1234.56
- Counterparty name extraction is the hardest part — will rely on proximity to IBAN and common label patterns ("t.n.v.", "ten name van", "beneficiary", etc.)
- Target banking apps: Dutch banks (ING, Rabobank, ABN AMRO) but EPC QR is EU-wide

## Constraints

- **Tech stack**: Static HTML + vanilla JavaScript — no build tools, no frameworks, no server
- **Dependencies**: Minimal — QR code generation library loaded via CDN
- **Privacy**: All processing happens client-side; no data leaves the browser

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Regex/pattern matching over AI | Free, offline, no API keys, fast — sufficient for structured payment data | — Pending |
| Static HTML+JS over server app | Zero setup — just open the file in a browser | — Pending |
| EPC QR standard | European standard supported by most EU banking apps | — Pending |
| Review step before QR generation | Payment data must be correct — user confirms/edits extracted fields | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-23 after initialization*
