# Betaal QR

Paste invoice or email text, get a scannable EPC payment QR code — no manual data entry into your bank app.

## How it works

1. Paste any invoice or email text into the box
2. The app extracts the IBAN, recipient name, and amount automatically
3. A QR code is generated that any Dutch/European banking app can scan to pre-fill a payment

All processing happens in your browser. Nothing is sent to a server.

## Usage

Download `index.html` and open it in your browser — no installation, no server, no build step.

Or open it directly from the [GitHub Pages link](https://leterhaar.github.io/betaal-qr) (if enabled).

## Compatibility

Works with any banking app that supports EPC QR codes (SEPA Credit Transfer), including bunq, Rabobank, ING, and most other European banking apps.

## Tech

Single HTML file — vanilla JavaScript, no frameworks, no dependencies except a QR code library loaded from CDN.
