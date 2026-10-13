# Architecture Decision Records (ADRs)

[← Main README](../../README.md) · [← Documentation Index](../README.md)

This directory documents the key architectural and design decisions made for the **Enterprise AI Receipt Extraction & Ledger Reconciliation Platform**.

---

## Decision Log

| ID | Title | Status | Decision Summary |
|---|---|---|---|
| [ADR 001](001-containerized-lambda-architecture.md) | **Containerized AWS Lambda Architecture** | Accepted | Package Python image processing pipeline into containerized Lambda workers to handle dynamic LANCZOS downsampling of 74M+ pixel scans with zero idle cost. |
| [ADR 002](002-dual-engine-ocr-llm-pipeline.md) | **Dual-Engine Vision OCR and Decoupled LLM Parsing** | Accepted | Combine Google Cloud Vision hierarchical OCR with parallel OpenAI GPT-4o Pydantic schemas (`ReceiptData` & `ReconciliationData`) to prevent hallucination. |
| [ADR 003](003-session-scoped-erp-auditing.md) | **Session-Scoped ERP Auditing and Regulatory Traceability** | Accepted | Enforce transaction-local session identity (`SET session.username = 'ai-service@accounting.internal'`) to comply with Danish statutory accounting law (*Bogføringsloven*). |
