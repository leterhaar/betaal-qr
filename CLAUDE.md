<!-- GSD:project-start source:PROJECT.md -->
## Project

**Betaal QR**

A static web app (single HTML file) that extracts payment details from pasted invoice or email text — IBAN, counterparty name, and amount — and generates an EPC QR code scannable by any European banking app. Runs locally in the browser with no server or API keys required.

**Core Value:** Paste text, get a scannable payment QR code — no manual data entry into your bank app.

### Constraints

- **Tech stack**: Static HTML + vanilla JavaScript — no build tools, no frameworks, no server
- **Dependencies**: Minimal — QR code generation library loaded via CDN
- **Privacy**: All processing happens client-side; no data leaves the browser
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Runtime
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Vanilla JavaScript (ES2020+) | N/A | App logic, extraction, QR wiring | No framework overhead; project fits in ~200 LOC; runs without build tools; PROJECT.md explicitly requires it |
| HTML5 | N/A | Single-file shell | `<textarea>`, `<canvas>`, `<input>` cover all UI needs; no templating required |
| CSS (vanilla, no preprocessor) | N/A | Styling | Single-file constraint rules out Tailwind CDN noise; plain CSS is sufficient |
### QR Code Generation
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **qrcode-generator** (by kazuhikoarase) | 1.4.4 | Render EPC payload as QR image | Pure JS, no dependencies, ships as a UMD module loadable from jsDelivr CDN with a single `<script>` tag; actively maintained; supports error correction level M which EPC069-12 requires |
### EPC QR Payload Construction
- Max payload: 331 characters (after URL-encoding is NOT applied — this is raw bytes)
- Error correction level: M (15% recovery)
- QR version: auto (library handles this)
- Line separator: `\n` (LF only, not CRLF)
- BIC field: optional in EPC version 002 (most Dutch banks accept without BIC)
- Amount field: omit entirely (not just blank) if amount is zero/unknown
### IBAN Extraction
### Amount Extraction
- `€1.234,56` (Dutch/German locale: dot as thousands, comma as decimal)
- `1,234.56` (English locale: comma as thousands, dot as decimal)
- `EUR 1234.56` (plain)
- `1234` (integer, no decimals)
### No Build Tooling
| Decision | Rationale |
|----------|-----------|
| No npm/Node | PROJECT.md constraint: "just open index.html in a browser" |
| No Webpack/Vite/Rollup | Build tools require Node; violates single-file static constraint |
| No TypeScript | No transpilation step allowed; use JSDoc comments for type hints if desired |
| No ESModules with import | `import` requires either a bundler or a server (CORS blocks local file:// imports in some browsers); use classic `<script>` tags instead |
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| QR library | qrcode-generator (kazuhikoarase) | **QRCode.js** (davidshimjs) | Last commit 2013; unmaintained; no error-correction-level control |
| QR library | qrcode-generator | **qrcode** (soldair/node-qrcode) | Node-first; browser bundle exists but is 80 KB+ and requires a bundler for clean tree-shaking; overkill for single-file use |
| QR library | qrcode-generator | **jsQR** | Decoder, not encoder — wrong direction |
| QR library | qrcode-generator | **qr-creator** | Smaller (6 KB), good CDN support, renders to canvas/SVG; viable alternative if qrcode-generator has any issues — use this as fallback |
| Styling | Vanilla CSS | Bootstrap CDN | Adds 30 KB for a UI with ~5 elements; unnecessary |
| Styling | Vanilla CSS | Tailwind CDN | Play CDN is development-only; not suitable for a file:// loaded page |
| Extraction | Regex | LLM/AI API | Explicitly out of scope (PROJECT.md) |
| Extraction | Regex | IBAN validation library (ibantools) | Correct checksums are nice-to-have; adds a CDN dependency for marginal benefit; implement mod-97 inline instead |
## Fallback QR Library: qr-creator
## Installation
## EPC QR Code Spec Reference
| Parameter | Value | Notes |
|-----------|-------|-------|
| Service Tag | `BCD` | Hardcoded literal |
| Version | `002` | Use v002 (allows omitting BIC) |
| Character encoding | `1` (UTF-8) | Always use 1 |
| Identification | `SCT` | SEPA Credit Transfer |
| Max beneficiary name | 70 chars | Truncate if longer |
| Max IBAN length | 34 chars | ISO 13616 |
| Amount format | `EURnnn.nn` | e.g., `EUR12.50`; omit field entirely if unknown |
| Max unstructured reference | 140 chars | Invoice number / omschrijving goes here |
| Error correction | Level M | Required by spec |
| Max QR payload | 331 bytes | Keep payload short; Dutch invoices rarely approach this |
| Line ending | `\n` (LF) | Not CRLF |
## Sources
| Claim | Source | Confidence |
|-------|--------|------------|
| EPC QR field structure | EPC069-12 spec (training data from official EPC document) | HIGH |
| qrcode-generator API | npm package documentation (training data) | MEDIUM — verify version |
| qr-creator API | npm package documentation (training data) | MEDIUM — verify version |
| IBAN regex structure | ISO 13616 standard (training data) | HIGH |
| Amount format variations | Domain knowledge from Dutch invoicing conventions | MEDIUM |
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
