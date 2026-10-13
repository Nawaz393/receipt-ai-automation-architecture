# Diagram Sources

[← Main README](../README.md)

- [System Context](system-context.mmd) — High-level architecture connecting ingestion, SQS, containerized AWS Lambda, Vision OCR, GPT-4o, and PostgreSQL Double-Entry ERP.
- [Image Processing Pipeline](image-processing-pipeline.mmd) — Pixel boundary guarding, dynamic LANCZOS downsampling for 74M-pixel scans, format normalization, and Google Cloud Vision integration.
- [AI Revalidation Sequence](ai-revalidation-sequence.mmd) — End-to-end request lifecycle: OCR extraction, parallel GPT-4o multimodal parsing, Danish VAT 25% mathematical validation, and `o3-mini` auto-accountant rule synthesis.
- [Reconciliation State Machine](reconciliation-state-machine.mmd) — Receipt document lifecycle states from upload, queue validation, OCR extraction, parallel parsing, audit session injection, to automated ledger booking.
- [Database Integration](database-integration.mmd) — Session context injection (`SET session.username = 'ai-service@accounting.internal'`), double-entry table schema interactions, and audit triggers.

GitHub renders Mermaid diagrams embedded in Markdown. These standalone `.mmd` source files maintain portable, version-controlled architecture definitions.
