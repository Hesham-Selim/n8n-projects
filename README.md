# n8n Projects

A collection of n8n workflows I've built, published with all credentials, API keys, IDs and personal data removed.

## Projects

| Project | What it does | Stack |
|---|---|---|
| [OCR Invoice System](./ocr-invoice-system) | Receives invoices (PDF or image) by webhook, extracts the text, uses an LLM to pull out structured fields, validates them, and routes risky invoices to a human reviewer before logging to Google Sheets. | Webhook, OCR.space, Google Gemini, Gmail, Google Sheets |

## How to import a workflow

1. Open the project folder and download `workflow.json`.
2. In n8n: **Workflows → Import from File** (or paste into the canvas).
3. Create your own credentials and attach them to the nodes listed in the project README.
4. Replace the placeholders (`YOUR_GOOGLE_SHEET_ID`, `YOUR_REVIEWER_EMAIL@example.com`).

## Security

No credentials are included. Every exported workflow was checked for API keys, emails, spreadsheet IDs and webhook URLs before publishing.
