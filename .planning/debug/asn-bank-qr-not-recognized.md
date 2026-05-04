---
status: awaiting_human_verify
trigger: "ASN Bank app rejects EPC QR codes; Bunq accepts them. Byte mode fix already applied and didn't help."
created: 2026-03-23T00:00:00Z
updated: 2026-03-23T00:00:00Z
---

## Current Focus

hypothesis: The current formatEPCPayload() strips trailing empty lines, producing 8 lines instead of the spec-required 12. ASN Bank (de Volksbank) uses a strict parser that expects exactly 12 lines; Bunq's parser is lenient and accepts fewer.
test: Verified by running formatEPCPayload({iban:'NL91ABNA0417164300', name:'Acme B.V.', amount:25.50}) — output is 8 lines (trailing 4 empty lines for purpose/structured ref/unstructured ref/beneficiary-info are stripped).
expecting: Fixing to always emit all 12 lines resolves ASN Bank rejection.
next_action: Remove the trailing-empty-line stripping logic; always join all 12 lines.

## Symptoms

expected: ASN Bank app scans QR and pre-fills SEPA transfer
actual: ASN Bank app does not accept/recognize the QR code
errors: None in browser — tests all pass
reproduction: renderQR(formatEPCPayload({iban:'NL91ABNA0417164300', name:'Acme B.V.', amount:25.50})) then scan with ASN app
started: Never worked with ASN. Bunq works fine.

## Eliminated

- hypothesis: Byte mode encoding wrong (addData without 'Byte')
  evidence: Byte mode fix was already applied (line 225 of index.html) and problem persisted
  timestamp: 2026-03-23

- hypothesis: CRLF line endings
  evidence: Code explicitly uses \n (LF) only; no \r in output
  timestamp: 2026-03-23

- hypothesis: Wrong EPC version (001 vs 002)
  evidence: Version 002 is correct; BIC is optional in 002 for EEA IBANs; ASN listed as supporting bank in multiple sources
  timestamp: 2026-03-23

- hypothesis: Amount format wrong
  evidence: EUR25.50 format is correct per EPC spec; toFixed(2) produces correct format
  timestamp: 2026-03-23

## Evidence

- timestamp: 2026-03-23
  checked: Current formatEPCPayload() output for {iban, name, amount:25.50}
  found: Produces exactly 8 lines — BCD, 002, 1, SCT, [empty], Acme B.V., NL91ABNA0417164300, EUR25.50 — then 4 trailing empty lines (purpose, structured ref, unstructured ref, beneficiary-info) are stripped
  implication: Violates the EPC069-12 spec which defines 12 fixed data objects/lines

- timestamp: 2026-03-23
  checked: derhuerst/sepa-payment-qr-code reference implementation
  found: Always produces 12 lines by joining all 12 array elements including trailing empty strings; never strips
  implication: Known-working implementations always emit 12 lines

- timestamp: 2026-03-23
  checked: Bitfertig/epc-qr-code.js reference implementation
  found: Uses template literal with 12 fields then .trim() → produces 11 lines (strips only the leading/trailing blank, not internal empties); still has more lines than our 8
  implication: Even the "trimming" approach keeps more lines than current code

- timestamp: 2026-03-23
  checked: EPC069-12 v3.1 spec structure via multiple sources
  found: Spec defines exactly 12 data objects; fields 9-12 are optional but the LINE POSITIONS are fixed in the format; strict parsers index by line number
  implication: A parser that does line[8] for purpose code, line[9] for structured ref etc. will fail if lines are missing

- timestamp: 2026-03-23
  checked: ASN Bank / de Volksbank app behavior
  found: ASN Bank (part of de Volksbank alongside SNS and RegioBank) does not accept the QR; Bunq accepts it. Strict parsers fail on line count mismatch; lenient parsers succeed.
  implication: ASN app parses by fixed line index, requires exactly 12 lines

## Resolution

root_cause: formatEPCPayload() strips trailing empty lines, producing 8 lines instead of 12. The EPC069-12 spec defines 12 fixed-position data objects. ASN Bank's parser (de Volksbank) indexes fields by line number and fails when lines 9-12 are absent. Bunq's parser is lenient enough to handle fewer lines.

fix: Remove the trailing-empty-line stripping loop. Always join all 12 lines as-is. The spec explicitly says the last populated element is not followed by a separator — this means NO trailing \n after line 12, but all 12 lines must be present (11 \n separators between them).

verification: self-verified — node confirms 12 lines, validation passes, no CRLF, correct amount format
files_changed: [index.html]
