# ADR 002: Dual-Engine Vision OCR and Decoupled LLM Parsing

[← Main README](../../README.md) · [← Documentation Index](../README.md)

- **Status:** Accepted
- **Context:** Extracting structured financial records from messy, multi-lingual receipts requires both character-level optical accuracy (reading low-contrast receipt numbers, comma vs. dot separators) and high-level semantic reasoning (Danish tax rules, handwritten amount overrides, date hierarchies). End-to-end multimodal models (e.g., pure GPT-4o Vision directly on raw images without OCR tokens) can hallucinate numbers, misread tiny receipt fonts, or fail to extract long itemized tables due to context-length and resolution constraints.
- **Decision:** Implement a dual-engine processing pipeline combining Google Cloud Vision for deterministic optical character recognition (OCR) with OpenAI GPT-4o for structured semantic parsing and rule adherence. The OCR text is passed alongside downsampled base64 image data into concurrent, decoupled Pydantic schemas (`ReceiptData` and `ReconciliationData`) executed via `ThreadPoolExecutor(max_workers=2)`.
- **Benefits:**
  - **Character-Level Grounding:** Google Cloud Vision produces spatial token bounding boxes and hierarchical text structures (Page → Block → Paragraph → Word → Symbol), preventing digit hallucination on low-contrast thermal receipts.
  - **Decoupled Schema Execution:** Splitting the core ledger schema (`ReceiptData`) from the detailed bank audit schema (`ReconciliationData`) halves the risk of token truncation and avoids schema collision.
  - **Cost-Effective OCR Caching:** Raw OCR text is persisted to `double_entry.receipt_ocrs`. Downstream re-parsing tasks (`backfill_reconciliation`) can re-parse receipts using cached text without re-incurring Google Cloud Vision billing ($1.50 per 1,000 units saved).
  - **Structured Pydantic Guarantees:** Strict output adherence via OpenAI Structured Outputs (`response_format=ReceiptData`) ensures zero schema parsing errors in Python.
- **Trade-offs:**
  - Two external API roundtrips (GCP Vision then OpenAI GPT-4o) introduce cumulative network latency (~3.5 seconds total end-to-end).
  - Multi-cloud dependency requires managing both GCP service account secrets and OpenAI API tokens.
- **Alternatives:**
  - *End-to-End Pure LLM Vision (GPT-4o Vision only):* Higher risk of digit hallucination on small receipt fonts; no persistent text cache for fast batch reprocessing.
  - *Self-Hosted Tesseract OCR:* High error rate on crumpled, skewed, or thermal receipts, requiring extensive manual image morphological operations.
  - *Amazon Textract:* Slower turnaround and significantly higher cost per page compared to Google Cloud Vision API for multilingual European receipts.
