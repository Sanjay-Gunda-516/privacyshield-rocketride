# PrivacyShield

PrivacyShield is a small RocketRide project I built to test a practical document-privacy workflow: take in text or an image, extract the readable content, detect sensitive information, and return a redacted version.

I wanted the project to stay simple enough to understand quickly, while still combining several RocketRide nodes in one end-to-end flow instead of stopping at a basic webhook or chat example.

## What it does

The pipeline handles two main cases:

- plain text sent to the webhook
- images or scanned content that need OCR first

From there, the extracted text is passed through PII detection and sensitive values are masked before the result is returned.

```text
                           ┌───────────────┐
                         ┌→│ NER (BERT)    │
                         │ └───────────────┘
                         │
Webhook → Parser ─────────┼→ PII Anonymizer → Response
    │                    │
    └────────→ OCR ──────┘
```

The NER node is kept as a separate analysis branch. The redaction path uses parsed or OCR text directly.

## Why I structured it this way

My first version sent NER output directly into the anonymizer. While testing, I noticed the response text was being duplicated.

I used RocketRide's Trace view to narrow it down. The NER input was 153 characters, but the anonymizer was sending 306 characters to the response. I changed the flow so Parser and OCR feed the anonymizer directly and kept NER as a separate analysis branch. After that, the duplicate output disappeared.

That debugging step ended up being one of the most useful parts of the project because it forced me to understand how data was actually moving through the pipeline instead of just connecting nodes until it worked.

## Main nodes

- **Webhook** — receives the incoming request
- **Parser** — extracts content from incoming data
- **OCR** — reads text from images and scanned content
- **Named Entity Recognition** — runs BERT-based entity analysis
- **PII Anonymizer** — masks sensitive information
- **Response** — returns the redacted text to the caller

For the working version I used:

- BERT Base for NER
- GLiNER Multi PII for PII detection
- EasyOCR for English OCR
- `█` as the masking character

## Running it locally

1. Open `privacyshield.pipe` in VS Code.
2. Make sure the RocketRide local engine is connected.
3. Start the Webhook node.
4. Open **Endpoint Info**.
5. Copy the current webhook URL and public authorization key.
6. Use those values in one of the test commands below.

Do not commit or share generated endpoint credentials.

### Plain-text test

```bash
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Authorization: Bearer YOUR_PUBLIC_AUTH_KEY" \
  -H "Content-Type: text/plain" \
  --data-binary 'John Smith lives at 123 Main Street, Jersey City, NJ. His email is john.smith@example.com, his phone number is 201-555-0198, and his SSN is 123-45-6789.'
```

The response should return `status: OK` and a `redacted_text` field with the detected PII masked.

### OCR test

A fictional patient-intake image is included here:

```text
demo/privacyshield-ocr-test.png
```

Run:

```bash
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Authorization: Bearer YOUR_PUBLIC_AUTH_KEY" \
  -H "Content-Type: image/png" \
  --data-binary "@demo/privacyshield-ocr-test.png"
```

This sends the image through OCR first, then through the PII redaction flow.

## What I tested

I tested both of these locally:

```text
Plain text → Webhook → Parser → PII Anonymizer → Response
Image → Webhook → OCR → PII Anonymizer → Response
```

Both returned successful responses and masked sensitive values in the output.

The OCR test uses fictional data only.

## Open-source contribution

Alongside building PrivacyShield, I also picked up [rocketride-server issue #1911](https://github.com/rocketride-org/rocketride-server/issues/1911).

The issue is in the Anthropic LLM integration. The current thinking configuration can fall back to an older `budget_tokens` request shape for newer Claude profiles, which causes an HTTP 400 when the node runs.

I am working through the existing implementation and tests so the fix can support the newer thinking configuration without changing expected behavior for older models.

Once the pull request is submitted, I will add the PR link here.

## Demo

The screenshots below show the working pipeline, the fictional OCR test document, and the redacted output returned by the pipeline.

### Pipeline

![PrivacyShield pipeline](screenshots/pipeline.png)

### OCR test input

![OCR test input](demo/privacyshield-ocr-test.png)

### Redacted output

![Redacted output](screenshots/redacted-output.png)

### Video walkthrough

[Watch the PrivacyShield demo on YouTube](https://youtu.be/JusZOfXtvbo)

## Things I would improve next

The current version is a proof of concept, not something I would use as-is for production data.

A few next steps I would take:

- add deterministic checks for formats like email addresses and SSNs
- build a larger evaluation set with different document layouts
- measure missed detections and false positives
- test lower-quality scans and rotated images
- add a small frontend so a user can upload a file and preview the redacted result
- deploy the pipeline behind a secured endpoint

One thing I noticed during testing is that model-based redaction can sometimes mask only part of an email address. That is exactly the kind of case where I would add a deterministic fallback rule in a production version.
