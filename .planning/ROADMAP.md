# Roadmap: Betaal QR

## Overview

Two phases build this app inside-out. Phase 1 implements all the spec-sensitive pure logic — IBAN extraction and MOD-97 validation, amount normalisation across locale formats, name heuristics, EPC payload assembly, and field validation — as pure functions testable in the browser console. Phase 2 wires everything into a complete working app: form UI, QR rendering, dark/light mode, copy-to-clipboard, and the paste area that triggers auto-extraction. The split ensures EPC spec correctness failures surface in isolation, before UI complexity is added.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Core Logic** - Pure-function pipeline: IBAN extraction + MOD-97 validation, amount parsing, name heuristics, EPC payload assembly, field validation
- [ ] **Phase 2: Full App** - Complete working app: form UI, QR rendering, paste area with auto-extraction, dark/light mode, copy to clipboard
- [ ] **Phase 3: Wero QR** - Add Wero/iDEAL QR code support via local backend for banks that don't support EPC QR (ABN AMRO, ASN, Rabobank)

## Phase Details

### Phase 1: Core Logic
**Goal**: A browser-console-testable EPC payment pipeline — paste a payment object in, get a valid EPC string and extracted fields out
**Depends on**: Nothing (first phase)
**Requirements**: EXTR-02, EXTR-03, EXTR-04, FLDS-02, FLDS-03, QRCD-01, QRCD-03, GENL-01, GENL-03
**Success Criteria** (what must be TRUE):
  1. `parsePaymentText()` correctly extracts IBAN (including space-separated formats), amount (Dutch and English locale), and beneficiary name from a raw invoice text string when called in the browser console
  2. `formatEPCPayload()` produces an exactly-12-line string with LF line endings, version 002, amount as `EURnnn.nn`, and blank placeholders for omitted optional fields
  3. `validateFields()` rejects an IBAN that fails MOD-97 checksum and accepts one that passes, with an inline-ready error message string
  4. The single `index.html` file opens in a browser without a server and without errors in the console
**Plans:** 2 plans

Plans:
- [x] 01-01-PLAN.md — Text extraction functions (IBAN, amount, name) with MOD-97 validation and inline test harness
- [x] 01-02-PLAN.md — EPC payload assembly, byte-limit validation, QR library integration, end-to-end pipeline

### Phase 2: Full App
**Goal**: The complete product — user pastes invoice text, fields auto-populate, user confirms or corrects, generates a scannable EPC QR code
**Depends on**: Phase 1
**Requirements**: EXTR-01, FLDS-01, FLDS-04, FLDS-05, QRCD-02, QRCD-04, QRCD-05, GENL-02
**Success Criteria** (what must be TRUE):
  1. User pastes Dutch invoice text into the textarea and the IBAN, amount, and name fields populate automatically with three-state feedback (success / partial / failed extraction)
  2. User can review and edit the pre-populated IBAN, name, amount, and remittance fields before generating — inline validation errors appear for invalid IBAN or missing required fields without blocking the form
  3. A QR code appears at minimum 200x200px after clicking Generate, and it scans correctly in the Bunq mobile app to pre-fill a SEPA credit transfer with correct values
  4. User can copy the QR code image to clipboard using the copy button
  5. App follows OS dark/light mode preference automatically
**Plans:** 1/2 plans executed
**UI hint**: yes

Plans:
- [x] 02-01-PLAN.md — Form UI with paste-to-extract, editable fields, inline validation, QR generation at 200x200px
- [ ] 02-02-PLAN.md — Copy to clipboard, dark/light mode, banking app scan verification

### Phase 3: Wero QR
**Goal**: Support banks that don't accept EPC QR (ABN AMRO, ASN, Rabobank) by adding Wero/iDEAL QR generation via a local backend
**Depends on**: Phase 2
**Requirements**: TBD
**Success Criteria** (what must be TRUE):
  1. A local backend serves Wero QR codes alongside EPC QR codes
  2. ABN AMRO and/or ASN banking apps accept the generated QR code
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Core Logic | 2/2 | Complete | 2026-03-23 |
| 2. Full App | 1/2 | In Progress|  |
| 3. Wero QR | 0/? | Not started | - |
