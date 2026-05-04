# Betaal QR

## What This Is

A static web app (single HTML file) that extracts payment details from pasted invoice or email text — IBAN, counterparty name, and amount — and generates an EPC QR code scannable by any European banking app. Runs locally in the browser with no server or API keys required.

## Core Value

Paste text, get a scannable payment QR code — no manual data entry into your bank app.

## Requirements

### Validated

- [x] User can paste invoice/email text into a text area — Validated in Phase 1 (extraction) + Phase 2 (UI)
- [x] App extracts IBAN from pasted text using regex pattern matching — Validated in Phase 1
- [x] App extracts payment amount from pasted text using pattern matching — Validated in Phase 1
- [x] App extracts counterparty name from pasted text using heuristics — Validated in Phase 1
- [x] User can review and edit extracted IBAN, name, and amount before QR generation — Validated in Phase 2
- [x] App generates a valid EPC QR code from the confirmed payment details — Validated in Phase 2
- [x] QR code is scannable by Bunq banking app (confirmed by user) — Validated in Phase 2
- [x] App runs as a static HTML+JS page — just open index.html in a browser — Validated in Phase 1

### Active

(All core requirements validated. Phase 3 will add Wero QR for banks that don't support EPC QR.)

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
| Regex/pattern matching over AI | Free, offline, no API keys, fast — sufficient for structured payment data | Validated Phase 1 |
| Static HTML+JS over server app | Zero setup — just open the file in a browser | Validated Phase 1 |
| EPC QR standard | European standard supported by most EU banking apps | Validated Phase 2 (Bunq confirmed) |
| Review step before QR generation | Payment data must be correct — user confirms/edits extracted fields | Validated Phase 2 |

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
*Last updated: 2026-03-23 after Phase 2 completion — all core requirements validated, app is feature-complete*
