# Anzu Dynamics

AI-powered invoice automation and ERP integration for mid-market companies in Mexico and Colombia.

This repository contains no proprietary source code. It documents the system architecture, agent pipeline design, and technical decisions behind the Anzu platform. The core codebase is private.

---

## The problem

Mid-market finance teams in Latin America process invoices manually. A vendor sends a PDF or XML by email. Someone opens it, reads the fields, and re-types that data into an ERP — one invoice at a time. At 200+ invoices a month across multiple vendors and cost centers, that is several hours of work per week with no tolerance for error. The ERPs exist. The invoices exist. Nothing connects them automatically.

## What Anzu does

Anzu is a vertical AI agent that ingests invoices from email or direct upload, extracts structured data using OCR and NLP, validates it against procurement rules, and pushes the result directly into the client's ERP — without manual input.

The agent handles both PDF and XML formats (including Mexican CFDIs), resolves vendor mapping, flags exceptions for human review, and logs every transaction for audit purposes.

## Agent pipeline

```
Invoice arrives (email ingestion or direct upload)
        │
        ▼
Document classification
(PDF vs XML, invoice vs non-invoice)
        │
        ▼
OCR + vision model extraction
(Anthropic / OpenAI vision APIs)
        │
        ▼
NLP field parsing
(vendor, RFC, folio, amounts, line items, dates)
        │
        ▼
Validation layer
(procurement rules, duplicate detection, exception flagging)
        │
        ▼
ERP push
(SAP B1 / NetSuite / CONTPAQi / Siigo / SINCO)
        │
        ▼
Audit log + confirmation
```

For a detailed breakdown of each stage, see [`architecture/pipeline.md`](architecture/pipeline.md).

## Stack

| Layer | Technology |
|---|---|
| Language | Python |
| Vision / extraction | Anthropic Claude vision API, OpenAI vision API |
| Document parsing | OCR pipeline, `lxml` for CFDI XML parsing |
| ERP integrations | SAP Business One, NetSuite, CONTPAQi, Siigo, SINCO |
| Email ingestion | IMAP (multi-server) |
| Validation | Custom rules engine per client configuration |

## ERP integration approach

Each ERP integration is built as a modular adapter. The core pipeline produces a normalized invoice object; the adapter layer translates that object into the format each ERP expects and handles authentication, rate limits, and error responses independently.

This means adding a new ERP requires building one adapter without touching the extraction pipeline.

## Results

- Deployed to mid-market clients in Mexico and Colombia
- Automates the majority of manual touchpoints in the invoice procurement cycle
- Selected for the **Polsky Center EIP 2025** cohort at the University of Chicago
- Recognized in the **Chicago Booth New Venture Challenge 2026**

## Why the code is private

The extraction pipeline, ERP adapters, and validation logic represent the core commercial IP of the product. Clients share sensitive financial data with the platform under NDA. Publishing the source code is not something we can do at this stage.

If you are a technical evaluator and want to understand specific implementation decisions, open a [Technical Inquiry](../../issues/new?template=technical-inquiry.md) or reach out directly.

---

## Built by

**Emilio Montemayor** — [github.com/emiliomt](https://github.com/emiliomt)  
Co-Founder, Anzu Dynamics  
Chicago Booth MBA '26
