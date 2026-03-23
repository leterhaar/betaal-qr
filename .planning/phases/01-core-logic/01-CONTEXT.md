# Phase 1: Core Logic - Context

**Gathered:** 2026-03-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Pure-function pipeline for payment text extraction and EPC QR payload assembly. No UI — functions testable in browser console. Includes: IBAN extraction + MOD-97 validation, amount parsing with Dutch/English locale support, counterparty name heuristics, EPC payload formatting (v002), field validation.

</domain>

<decisions>
## Implementation Decisions

### Claude's Discretion

User chose to skip detailed discussion — Claude has full discretion on all implementation decisions for this phase. Use research findings from `.planning/research/` to guide choices. Key defaults:

- **Amount parsing:** Support both Dutch (€1.234,56) and English (1,234.56) formats. Treat ambiguous amounts conservatively — prefer Dutch locale given target user.
- **Name extraction:** Use Dutch label heuristics (t.n.v., begunstigde, ten name van, naam) + proximity to IBAN. Fall back to empty string if nothing found — user edits in Phase 2.
- **IBAN handling:** If multiple IBANs found, pick the first one. Accept spaces in IBANs (normalize by stripping). Validate with MOD-97.
- **EPC payload:** Use v002 (BIC optional), UTF-8 (charset 1), include blank lines for omitted optional fields per spec. Amount as `EURnnn.nn` with exactly 2 decimal places.
- **Special characters:** Preserve UTF-8 characters in beneficiary names. Enforce 70-char limit by truncating with warning.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### EPC QR Standard
- `.planning/research/FEATURES.md` — EPC069-12 field specification (lines 9-48), mandatory vs optional fields, 331-byte payload limit
- `.planning/research/STACK.md` — QR library recommendation (qrcode-generator), vendoring strategy

### Architecture
- `.planning/research/ARCHITECTURE.md` — Component boundaries, data flow, pure function pipeline design
- `.planning/research/PITFALLS.md` — Amount parsing risks, EPC payload format sensitivity, IBAN validation requirements

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield project, no existing code

### Established Patterns
- None — first phase establishes patterns

### Integration Points
- Functions created here will be wired into the UI in Phase 2

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches based on research findings.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-core-logic*
*Context gathered: 2026-03-23*
