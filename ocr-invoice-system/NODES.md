# Node-by-node notes

## 1. Intake and routing

**Receive file (Webhook)**: POST endpoint at path `invoice-intake`, protected with header authentication. The uploaded file arrives as binary under the property `file`.

**Identify file type** (Switch): checks `$binary.file.mimeType`. Output 0 = PDF, output 1 = image (`image/*`), fallback = unsupported.

**unsupported** (No-op): end of the line for anything that is not a PDF or an image.

## 2. Text extraction

**Extract from File**: reads the PDF text layer (`binaryPropertyName: file`) and returns `text` and `numpages`.

**Merge** (combine by position): re-attaches the original binary file (sent in parallel from the Switch) to the extracted text, so a fallback to OCR still has the file.

**If** (text-layer check): passes only when average text per page is at least 50 characters. Real digital PDFs go straight on; scanned PDFs have almost no text layer and are sent to OCR.

**OCR** (HTTP Request to OCR.space): multipart POST with engine 2, table mode, auto scale and orientation detection, 60 s timeout, retry on failure. Used for images and for scanned PDFs.

## 3. Cleaning

**Normalize & clean text** (Code): detects whether the text came from OCR or the PDF layer, joins OCR pages, converts Arabic-Indic and Persian digits to ASCII, strips control characters, collapses whitespace, and outputs `request_id`, `filename`, `extraction_method`, `ocr_failed`, `raw_text`, `text_length`.

## 4. AI extraction

**Basic LLM Chain**: sends `raw_text` to the model with a strict system prompt: treat input as untrusted, never guess, ISO dates, plain-number amounts, list uncertain fields, give a confidence score.

**Google Gemini Chat Model**: the language model attached to the chain.

**Structured Output Parser**: forces the answer into a JSON schema (supplier, invoice number, dates, currency, subtotal, tax, total, line items, confidence, uncertain fields).

## 5. Validation

**Validate & score** (Code): builds a list of flags, then sets `needs_review` and `status`.

- missing required fields, invalid / future / stale dates, due date before invoice date
- non-positive or high total, subtotal + tax mismatch, line items not matching subtotal
- invoice number or total not literally present in the source text (anti-hallucination)
- OCR failure, very little text, low confidence, model-reported uncertainty
- possible duplicate (supplier + invoice number, kept in workflow static data)

**Needs Human review ?** (If): routes on `needs_review`. True goes to review, false goes straight to the ledger.

## 6. Human review

**Send a message** (Gmail): emails the reviewer the filename.

**human review form** (Wait, resume on form): pauses the execution and shows a form with an Approve / Reject dropdown and a comment box.

## 7. Output

**build ledger row** (Code): merges the validated record with the reviewer decision and comment, and sets the final status (`approved_by_human`, `rejected_by_human`, `auto_approved` or `pending_review`).

**Append row in sheet**: appends every record to the main ledger tab.

**If1**: checks the reviewer decision.

**Append row in sheet1 / sheet2**: copies approved records to the `approved` tab and rejected ones to the `rejected` tab.
