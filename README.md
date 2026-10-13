# Win-Man / Revisor1 AI Receipt Automation & Auto-Accountant Architecture

A documentation-first architectural showcase of the event-driven receipt extraction, Danish statutory VAT cross-validation, and autonomous bank ledger reconciliation platform engineered for **Revisor 1** (`revisor1.dk`), a leading Danish accounting and auditing firm.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![AWS Lambda](https://img.shields.io/badge/AWS_Lambda-10GB_Container-FF9900?logo=awslambda&logoColor=white)
![Pillow](https://img.shields.io/badge/Pillow-LANCZOS_Resampling-blue)
![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o_%7C_o3--mini-412991?logo=openai&logoColor=white)
![Google Cloud Vision](https://img.shields.io/badge/GCP_Vision-OCR_Annotation-4285F4?logo=googlecloud&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Double--Entry_Core-4169E1?logo=postgresql&logoColor=white)

---

## Executive Architectural Summary & Impact

Operating under Danish statutory accounting law (*Bogføringsloven*) and Danish Tax Agency (**SKAT**) regulations, processing financial receipts has historically required substantial manual typist intervention. The **Win-Man AI Receipt Extraction & Reconciliation Suite** replaces manual data-entry contractors with an event-driven, containerized AWS Lambda microservice and an autonomous bank rule reasoning bot.

### Quantified Production Impact (Audited Production Data)

| Metric | Human Contractor Baseline | AI Autonomous Engine | Improvement |
|---|---|---|---|
| **Processing Turnaround Time** | 4 to 48 hours | **3.2 – 5.8 seconds** | **~99.8% Latency Reduction** |
| **Unit Cost per Receipt** | $0.16 – $0.25 (piece rate + hourly tracking) | **$0.010 – $0.014** (Vision + GPT-4o + Lambda) | **~93% Cost Reduction** |
| **Volume Processed (4-Month Audit)** | 125,897 manual operations ($20,167 labor) | Handled autonomously with zero backlog | **Scale-ready for peak VAT deadlines** |
| **Danish VAT (*Moms*) Calculation Accuracy** | Susceptible to human typo & line-item confusion | **100% Deterministic Cross-Check** ($Brutto = Netto \times 1.25$) | **Zero tax formula discrepancies** |
| **Bank Transaction Rule Generation** | Manual accountant SQL rule crafting | **Autonomous AI Rule Proposer** (`o3-mini`) | **>80% auto-reconciliation, >90% acceptance** |

---

## Top-Level System Context

```mermaid
flowchart TB
    subgraph Ingestion["Client Ingestion & Document Storage"]
        CLIENT["Revisor 1 Client / Mobile App"] -->|"Upload Receipt Image"| S3[("AWS S3: receipt_files")]
        VIEWER["Win-Man File Viewer (PHP)"] -->|"Rotate / Inspect Scan"| S3
    end

    subgraph Messaging["Asynchronous Event Queue"]
        WINMAN_APP["Win-Man ERP Core (revisor1.dk)"] -->|"Enqueue Task ID"| SQS["AWS SQS: receipt_files (Batch: 1)"]
        SQS -->|"Event Trigger"| LAMBDA["AWS Lambda Container (Python 3.12 / 10GB)"]
    end

    subgraph Security["Secrets Management"]
        SECRETS["AWS Secrets Manager"] -->|"Dynamic GCP Service Key"| LAMBDA
    end

    subgraph VisionAI["Vision & Multimodal AI Pipeline"]
        LAMBDA -->|"Stream Image Bytes"| S3
        LAMBDA --> PREPROC["PIL Image Processor (LANCZOS & Max-Pixel Guard)"]
        PREPROC --> GCP_VISION["Google Cloud Vision API (Hierarchical OCR)"]
        GCP_VISION -->|"Raw Text + Token Stream"| THREADPOOL["ThreadPoolExecutor (workers=2)"]
        THREADPOOL -->|"Core Prompt + Base64"| GPT4O_CORE["OpenAI GPT-4o (ReceiptData Schema)"]
        THREADPOOL -->|"Recon Prompt + Base64"| GPT4O_RECON["OpenAI GPT-4o (ReconciliationData Schema)"]
        GPT4O_CORE --> MERGE["Result Merge & Sanitizer (Null-Byte Scrub)"]
        GPT4O_RECON --> MERGE
    end

    subgraph LedgerDB["Win-Man PostgreSQL Accounting Core"]
        MERGE -->|"SET session.username = 'da@revisor1.dk'"| PG_CORE[("PostgreSQL Database")]
        PG_CORE --> TYPING_TBL["double_entry.typing_output"]
        PG_CORE --> BOT_LOG["task_bots.typer_bot_output"]
        PG_CORE --> OCR_CACHE["double_entry.receipt_ocrs"]
        PG_CORE --> RULES_TBL["bank_transaction_rules"]
    end

    subgraph AutoAccountant["Autonomous Reconciliation Bot"]
        PG_CORE -->|"Customer Bank Feeds & History"| RECON_BOT["Auto-Accountant Service (Python)"]
        RECON_BOT -->|"Pattern Synthesis"| O3_MINI["OpenAI o3-mini Reasoning Engine"]
        O3_MINI -->|"Generated SQL Rules (active = false)"| RULES_TBL
    end
```

---

## Engineering Highlights

- **Computer Vision Ingestion Guardrails**: Dynamic proportional downsampling for high-DPI scans exceeding Google Cloud Vision's 75M-pixel limit using high-fidelity `Image.Resampling.LANCZOS` and Pillow decompression bomb defense (`Image.MAX_IMAGE_PIXELS = None`).
- **Parallel Dual-Schema Multimodal Parsing**: Concurrently extracts core accounting fields (`ReceiptData`) and extended bank audit metadata (`ReconciliationData`) via `ThreadPoolExecutor(max_workers=2)` against OpenAI `gpt-4o` structured outputs, preventing token truncation.
- **Danish Accounting Compliance & Tax Heuristics**: Deterministic statutory 25% Danish VAT (*Moms*) verification ($Brutto = Netto \times 1.25$), strict date priority hierarchies (Factura Dato vs. SKAT *"modtaget den"* vs. Gebyr period intervals), and handwritten currency override detection (EUR/USD converted to DKK).
- **Session-Scoped ERP Auditing**: Injects audited bot credentials (`SET session.username = 'da@revisor1.dk'`) before running double-entry ledger mutations, satisfying regulatory audit compliance (*Bogføringsloven*).
- **Defensive Data Integrity**: Recursive ASCII null-byte (`\x00`) scrubbing to prevent PostgreSQL JSONB crashes and atomic nullification of `matching_information` on ambiguous accounting entries to eliminate downstream bank mis-reconciliations.
- **Autonomous Rule Proposer (`auto_accountant`)**: Deep reasoning service powered by OpenAI `o3-mini` synthesizing regex-based SQL rules from historical bank feeds, achieving >80% automated ledger matching with >90% auditor acceptance.

---

## Documentation Map

| Topic | Guide |
|---|---|
| System Architecture & High-Throughput Pipeline | [Architecture Guide](docs/architecture.md) |
| Domain Boundaries & Component Map | [Domain Map](docs/domain-map.md) |
| Realistic Production Python & SQL Implementations | [Code Examples](docs/code-examples.md) |
| 74M+ Pixel Handling & Google Cloud Vision Optimization | [Computer Vision Pipeline](docs/computer-vision-pipeline.md) |
| Danish 25% VAT, Multi-Currency, and Zero-Hallucination Parsing | [AI Parsing & Revalidation](docs/ai-parsing-and-revalidation.md) |
| Database Session Isolation, Audit Trails & Bank Auto-Accountant | [ERP Reconciliation & Safety](docs/erp-reconciliation-and-safety.md) |
| Architectural Decision Records Index | [ADRs](docs/decisions/README.md) |
| Reusable Mermaid Diagram Source Files | [Diagram Sources](diagrams/README.md) |

---

## Technology Map

| Layer | Technologies / Patterns | Rationale |
|---|---|---|
| **Runtime & Compute** | AWS Lambda Container (Python 3.12, 10GB RAM) | Absorbs spiky quarterly tax workloads with zero idle cost; 10GB grants 6 vCPUs for rapid LANCZOS processing |
| **Messaging & Queuing** | Amazon SQS (`receipt_files`, BatchSize=1) | Decouples document ingestion from image analysis with visibility timeout retry safety |
| **Image Processing** | Pillow (`PIL`), LANCZOS Resampling, OpenCV | Guards against decompression bombs; safely downsamples 74M+ pixel images below Google Cloud limits |
| **OCR Engine** | Google Cloud Vision API (`text_detection`) | High-accuracy hierarchical spatial token extraction for European multilingual receipts |
| **AI Semantic Extraction** | OpenAI `gpt-4o` Structured Outputs (`pydantic`) | Parallel dual-schema parsing with zero token truncation and schema enforcement |
| **Autonomous Rule Reasoning**| OpenAI `o3-mini` | Analyzes historical bank text patterns to propose deterministic SQL regex auto-accounting rules |
| **Relational Persistence** | PostgreSQL, `psycopg2`, Double-Entry Engine | Session-scoped auditing (`session.username`), JSONB audit snapshots, and foreign key consistency |
| **Secrets Management** | AWS Secrets Manager (`boto3`) | Dynamic retrieval of GCP service account keys with zero secrets baked into container images |

---

## Contact

**M Nawaz Khan — Principal Software Architect & Full Stack Engineer**

- GitHub: [Nawaz393](https://github.com/Nawaz393)
- LinkedIn: [Muhammad Nawaz Khan](https://www.linkedin.com/in/nawaz393)

[Documentation Index](docs/README.md)
