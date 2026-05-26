# Anzu Agent Pipeline — Technical Architecture

This document describes the design of the Anzu invoice automation pipeline in detail. It covers each processing stage from document ingestion to ERP push and audit logging. No proprietary implementation code is included.

---

## Overview

The pipeline is a sequential, stateful agent that processes one invoice event at a time. Each stage produces a structured artifact consumed by the next. Side effects (ERP writes, audit log entries) are deferred to the final two stages so that all validation failures can be surfaced before any external system is mutated.

```
┌─────────────────────────────────────────────────────────────────────┐
│  INPUT LAYER          Email (IMAP) or Direct Upload                 │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  raw bytes + metadata
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CLASSIFICATION       PDF / XML  ·  Invoice / Non-invoice           │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  document type + routing decision
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  EXTRACTION           OCR (PDF)  ·  lxml parse (CFDI XML)           │
│                       Anthropic Claude vision  ·  OpenAI vision     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  raw extracted text / fields
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  NLP FIELD PARSING    vendor · RFC · folio · amounts · dates        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  normalized invoice object
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  VALIDATION           duplicates · procurement rules · exceptions   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  validated invoice object + flags
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ERP ADAPTER          SAP B1 · NetSuite · CONTPAQi · Siigo · SINCO  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  ERP response (success / error)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  AUDIT LOG            append-only event record per invoice          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Input Handling

### 1.1 Email ingestion (IMAP)

The platform monitors one or more client-configured mailboxes using the IMAP protocol. The connection is kept persistent with periodic IDLE polling so new messages are picked up within seconds of arrival.

- **Multi-server support.** Each client can configure separate inboxes for different departments or subsidiaries. Each connection is independently managed with its own credentials and TLS settings.
- **Attachment extraction.** The agent filters messages for attachments with `.pdf` or `.xml` MIME types. Non-invoice emails (no qualifying attachments) are skipped and logged.
- **Deduplication at ingestion.** A message-ID fingerprint is computed on arrival. Messages already seen are dropped before any further processing.
- **Envelope metadata.** Sender address, subject line, and received timestamp are captured alongside the attachment and passed downstream as document metadata.

### 1.2 Direct upload

Clients with internal tooling or ERP systems that can trigger webhooks can push documents directly via a REST endpoint. The endpoint accepts multipart form data or base64-encoded payloads. Documents uploaded this way bypass the mailbox monitor and enter the pipeline at the classification stage with equivalent metadata fields.

---

## 2. Document Classification

Before extraction begins, the pipeline must determine:

1. **Format:** Is the document a PDF or an XML file?
2. **Type:** Is this an invoice, or something else (purchase order, remittance advice, contract, etc.)?

### 2.1 Format routing

Format is detected by MIME type and, as a fallback, magic-byte inspection of the raw file header. PDFs and CFDIs (Mexican XML invoices) take distinct processing paths.

| Format | Detection | Processing path |
|--------|-----------|------------------|
| PDF | `application/pdf` / `%PDF` header | OCR → vision model |
| CFDI XML | `text/xml` + SAT namespace | lxml structural parse |
| Other | — | Flagged and routed to exception queue |

### 2.2 Invoice vs. non-invoice classification

Not every document in a vendor mailbox is an invoice. The classifier uses a lightweight heuristic model trained on labeled documents from clients in the target verticals. Features include:

- Presence of fiscal fields (RFC, folio, subtotal, IVA) for CFDIs
- Keyword density for terms associated with invoices, purchase orders, or contracts
- Document structure signals (tables with line-item patterns, total fields near the bottom)

Documents that score below the invoice-confidence threshold are moved to the exception queue rather than processed. Human reviewers can reclassify and resubmit.

---

## 3. OCR and Vision Model Extraction

### 3.1 PDF path

PDFs arrive in varying quality: clean digital exports from modern ERPs, scanned paper documents, and everything in between. The extraction strategy is tiered:

1. **Text layer extraction.** If the PDF contains a parseable text layer, it is extracted directly without invoking OCR. This is the fastest and most accurate path.
2. **OCR fallback.** If the text layer is absent or sparse (common for scanned invoices), the document is rasterized page-by-page and passed through an OCR engine to produce a text representation.
3. **Vision model pass.** The rasterized pages are sent to a vision model (Anthropic Claude vision API or OpenAI vision API) with a structured extraction prompt. The prompt instructs the model to return a JSON object matching the normalized invoice schema. Vision models handle layout variation, multi-column tables, and non-standard templates that rule-based parsers cannot.

Model selection between Anthropic and OpenAI is configurable per client deployment. In practice, both are available as fallback options; if one model returns a low-confidence extraction, the other is invoked and results are compared.

### 3.2 CFDI XML path

Mexican CFDIs are structured XML documents published under a SAT-defined namespace schema. Parsing is deterministic:

- The document tree is traversed using `lxml`
- Fields are read directly from known XPath locations (e.g., `Comprobante/@Total`, `Receptor/@Rfc`)
- Line items are extracted from `Conceptos/Concepto` nodes
- The digital seal (`TimbreFiscalDigital`) is verified against the SAT public key to confirm document authenticity

No OCR or vision model is needed for valid CFDIs. The vision model is only invoked if the XML is malformed or fails schema validation.

---

## 4. NLP Field Parsing

The extraction stage produces raw text or a preliminary field map. The NLP parsing stage normalizes this output into a typed invoice object with the following fields:

| Field | Type | Notes |
|-------|------|-------|
| `vendor_name` | string | Matched against the vendor master catalog |
| `vendor_rfc` | string | Mexican tax ID; validated against SAT RFC format |
| `folio` | string | Invoice number per the issuer's sequence |
| `uuid` | string | CFDI fiscal UUID (XML invoices only) |
| `issue_date` | date | ISO 8601 |
| `due_date` | date | Derived from payment terms if not explicit |
| `currency` | string | ISO 4217; defaults to MXN or COP per client config |
| `subtotal` | decimal | Pre-tax amount |
| `tax_amount` | decimal | IVA or equivalent |
| `total` | decimal | Final amount due |
| `line_items` | array | Each item: description, quantity, unit price, amount |
| `cost_center` | string | Resolved from vendor mapping or email routing rules |
| `payment_method` | string | CFDI `FormaPago` code or parsed equivalent |

**Vendor resolution.** Vendor names on invoices vary — abbreviations, legal suffixes, typos. The parser normalizes the extracted name against a per-client vendor master using fuzzy string matching. RFC is used as the authoritative key when available; name matching is the fallback.

**Amount reconciliation.** The parser independently sums line-item amounts and compares the result against the stated subtotal and total. A mismatch beyond a configurable tolerance threshold is flagged as an exception before reaching the validation layer.

---

## 5. Validation Layer

The validation layer applies business rules to the normalized invoice object before any ERP write is attempted. Rules are configured per client and may vary by vendor category, cost center, or invoice value.

### 5.1 Duplicate detection

Each processed invoice is fingerprinted using a combination of vendor RFC, folio number, total amount, and issue date. The fingerprint is checked against a per-client processed-invoice index. An exact match triggers a duplicate exception. A near-match (same RFC and folio, different amount) triggers a discrepancy flag for human review.

### 5.2 Procurement rules

Rule categories applied in sequence:

| Rule | Description |
|------|-------------|
| **Vendor allowlist** | Invoice must come from a vendor in the approved vendor master. Unknown vendors are held for approval. |
| **PO matching** | If the client uses purchase orders, the invoice folio or line items must match an open PO. Unmatched invoices are flagged. |
| **Amount thresholds** | Invoices above a configured value require secondary approval before ERP entry. |
| **Cost center validation** | The resolved cost center must exist in the ERP chart of accounts. |
| **Tax rate validation** | IVA rate must match the rate expected for the vendor's fiscal category. |
| **Due date sanity** | Due date must be after issue date and within a configured maximum payment window. |

### 5.3 Exception flagging

Any rule violation, low-confidence extraction, or structural anomaly sets an exception flag on the invoice object. Flagged invoices are routed to the human review queue rather than pushed to the ERP. The reviewer sees the original document, the parsed fields, and the specific exception reason. Once resolved, the invoice re-enters the pipeline from the validation stage.

---

## 6. ERP Adapter Pattern

The ERP push layer is built as a collection of modular adapters. The pipeline produces a single normalized invoice object; the adapter translates that object into the API call, file format, or SDK method the target ERP expects.

### 6.1 Adapter interface

Each adapter implements a common interface with the following methods:

- `authenticate()` — establish and cache a session or token
- `find_vendor(rfc, name)` → vendor ID — look up or create a vendor record
- `find_cost_center(code)` → cost center ID
- `push_invoice(normalized_invoice)` → ERP document ID
- `handle_error(response)` — classify ERP errors into retriable vs. terminal

The core pipeline never calls ERP-specific APIs directly. It calls the interface; the adapter handles everything else.

### 6.2 Adapter implementations

| ERP | Integration method | Auth |
|-----|--------------------|------|
| **SAP Business One** | Service Layer REST API | HTTP Basic over HTTPS; session cookie |
| **NetSuite** | SuiteQL + REST Record API | OAuth 2.0 (TBA tokens) |
| **CONTPAQi** | COM/DLL interop or local REST bridge | Windows session (on-premise) |
| **Siigo** | Siigo REST API | API key per client |
| **SINCO** | SINCO Web Services (SOAP/REST) | Username + token |

### 6.3 Error handling and retry

ERP APIs can fail transiently (rate limits, session expiry, network timeouts). Each adapter distinguishes:

- **Retriable errors** (HTTP 429, 503, session timeout): retried with exponential backoff, up to a configurable maximum.
- **Terminal errors** (invalid vendor ID, schema violation, duplicate document in ERP): logged immediately and the invoice is moved to the exception queue with the ERP error detail attached.

---

## 7. Audit Logging

Every invoice processed by the pipeline produces an immutable audit log entry. The log is append-only and scoped per client.

### 7.1 Log entry structure

Each entry records:

| Field | Description |
|-------|-------------|
| `event_id` | UUID generated at pipeline entry |
| `timestamp` | UTC timestamp at each stage transition |
| `stage` | Pipeline stage that generated the event |
| `document_fingerprint` | Hash of the original document bytes |
| `invoice_fields` | Snapshot of the normalized invoice object at the time of logging |
| `outcome` | `success`, `exception`, `duplicate`, `skipped` |
| `erp_document_id` | ERP-assigned ID if push succeeded |
| `exception_reason` | Structured reason code if flagged |
| `operator_id` | ID of the human reviewer if the invoice went through manual resolution |

### 7.2 Retention and access

Audit logs are retained for the period required by the client's fiscal jurisdiction (7 years for Mexico under SAT requirements, 5 years for Colombia under DIAN requirements). Logs are queryable by invoice UUID, vendor RFC, date range, and outcome. They are not modifiable after creation.

---

## Design decisions

**Why sequential rather than parallel?** Each invoice is a discrete fiscal document. Processing invoices in strict sequence per client makes it trivial to enforce per-client rate limits against ERP APIs and avoids race conditions in duplicate detection without distributed locking.

**Why support two vision model providers?** No single model performs optimally across all invoice templates. Having both available allows per-client tuning and provides a fallback if one provider has an outage or latency spike.

**Why a modular adapter over a unified ERP connector?** ERP APIs vary so widely in authentication model, data schema, and error semantics that a unified abstraction layer would be leaky and fragile. Isolated adapters are easier to test, deploy independently, and replace without affecting the core pipeline.

---

*Last updated: May 2026. For implementation questions, open a [Technical Inquiry](../.github/ISSUE_TEMPLATE/technical-inquiry.md) issue.*
