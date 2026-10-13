# Documentation Index

[← Main README](../README.md)

This documentation provides an architectural breakdown, domain boundaries, production code examples, and technical deep-dives for the **Win-Man / Revisor1 AI Receipt Automation & Auto-Accountant Suite**.

## Guides & Deep Dives

- [System Architecture](architecture.md) — High-throughput event-driven SQS/Lambda pipeline, request lifecycle, action modes, and fault isolation.
- [Domain Map & Boundaries](domain-map.md) — System decomposition across Vision pre-processing, LLM extraction engine, Danish VAT/currency validation, Win-Man ERP connector, and bank reconciliation engine.
- [Representative Code Examples](code-examples.md) — Production Python implementations extracted directly from the deployed codebase (dynamic LANCZOS downsampler, Pydantic schemas, null-byte cleaner, session-scoped SQL executor).
- [Computer Vision Pipeline](computer-vision-pipeline.md) — Deep dive on handling 74M+ pixel scans, decompression bomb defenses (`MAX_IMAGE_PIXELS`), rotation angle normalization, and Google Cloud Vision optimization.
- [AI Parsing & Revalidation](ai-parsing-and-revalidation.md) — Deep dive on Danish accounting heuristics (*Moms* 25% mathematical cross-validation), multi-currency resolution (DKK vs. EUR/USD), handwritten annotations, and prompt engineering for zero hallucination.
- [ERP Reconciliation & Safety](erp-reconciliation-and-safety.md) — PostgreSQL session-scoped audit context (`SET session.username = 'da@revisor1.dk'`), atomic integrity invalidation, and `o3-mini` automated bank reconciliation rule synthesis.

## Architectural Decision Records (ADRs)

- [ADR Index](decisions/README.md) — Index of architectural decisions.
- [ADR 001: Containerized Lambda Architecture](decisions/001-containerized-lambda-architecture.md) — Rationale for choosing 10GB containerized AWS Lambda over EC2 or ECS.
- [ADR 002: Dual-Engine Vision OCR & Decoupled LLM Parsing](decisions/002-dual-engine-ocr-llm-pipeline.md) — Rationale for combining Google Cloud Vision with parallel GPT-4o Pydantic schemas.
- [ADR 003: Session-Scoped ERP Auditing](decisions/003-session-scoped-erp-auditing.md) — Database session tracking for regulatory compliance under Danish statutory law.

## Visual Diagrams

- [Diagram Sources](../diagrams/README.md) — Version-controlled Mermaid diagram definitions for system context, image processing pipeline, sequence workflows, state machines, and relational schemas.
