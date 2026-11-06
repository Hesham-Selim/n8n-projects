# OCR Invoice System

An n8n workflow that turns incoming invoice files (PDF or image) into validated, structured records, and sends anything suspicious to a human reviewer before it is logged.

## Flow

```
Webhook (POST file)
   └─ Identify file type ── PDF ───► Extract text ─► Enough text? ── yes ─┐
                         │                                │ no (scanned)  │
                         ├─ Image ──────────────► OCR.space ◄────────────┘
                         └─ Other ──► rejected (no-op)
                                         │
                              Normalize & clean text
                                         │
                       Gemini extracts structured invoice fields
                                         │
                                  Validate & score
                                         │
                          Needs human review? ── yes ─► Email + review form (Approve / Reject)
                                         │ no                         │
                                         └──────────► Build ledger row ◄┘
                                                              │
                                       Google Sheets: full ledger + approved / rejected tabs
```

## Highlights

- **Hybrid extraction**: uses the PDF text layer when it exists, falls back to OCR for scans and images.
- **Arabic + English friendly**: converts Arabic-Indic and Persian digits to ASCII so dates and amounts parse correctly.
- **LLM with guardrails**: strict extraction prompt (no guessing, nulls for missing values) and a structured JSON schema output parser.
- **Rule-based validation** on top of the LLM: required fields, date sanity, subtotal + tax = total, line items sum, high-amount threshold, duplicate detection, and a grounding check that extracted values literally appear in the source text.
- **Human-in-the-loop**: flagged invoices pause on a review form before being recorded.

## Setup

Import `workflow.json`, then configure:

| Node | Credential / value needed |
|---|---|
| Receive file (Webhook) | Header Auth credential (header `X-Webhook-Secret`) |
| OCR | Header Auth credential for [OCR.space](https://ocr.space/ocrapi) (header `apikey`) |
| Google Gemini Chat Model | Google Gemini (PaLM) API credential |
| Send a message | Gmail OAuth2 credential, and set `YOUR_REVIEWER_EMAIL@example.com` |
| Append row in sheet / sheet1 / sheet2 | Google Sheets OAuth2 credential, and set `YOUR_GOOGLE_SHEET_ID` |

Create a Google Sheet with three tabs named `Sheet1`, `approved` and `rejected`.

## Try it

```bash
curl -X POST "https://<your-n8n-host>/webhook/invoice-intake" \
  -H "X-Webhook-Secret: <your-secret>" \
  -F "file=@invoice.pdf"
```

## Tunable thresholds

In the **Validate & score** node: `HIGH_AMOUNT`, `MIN_CONFIDENCE`, `MAX_AGE_DAYS`, `MIN_TEXT_LEN`.

See [NODES.md](./NODES.md) for a node-by-node explanation.
