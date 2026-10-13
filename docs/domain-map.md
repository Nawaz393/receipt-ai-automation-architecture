# Domain Map & System Decomposition

[← Main README](../README.md) · [← Documentation Index](README.md)

This document maps the bounded contexts, core data models, and interaction boundaries of the **Enterprise AI Receipt Extraction & Ledger Reconciliation Platform**.

---

## 1. Domain Decomposition Overview

```mermaid
flowchart TD
    subgraph VisionContext ["1. Computer Vision & Ingestion Domain"]
        RawImage["Raw Receipt Scans (JPEG, PNG, HEIC, PDF)"]
        LANCZOS["ImageProcessor (LANCZOS Resampling, RGB Norm)"]
        OCR["Google Cloud Vision Tokens & Coordinates"]
    end

    subgraph LLMContext ["2. Multimodal AI Extraction Domain"]
        Schema1["ReceiptData (Supplier, Date, Amounts, Line Items)"]
        Schema2["ReconciliationData (Payment Methods, Card IDs)"]
        Parser["ReceiptParser (GPT-4o Pydantic Enforcement)"]
    end

    subgraph ValidationContext ["3. Statutory Accounting & Validation Domain"]
        VatRule["Danish Bogføringsloven (Statutory 25% VAT Assertions)"]
        CurrencyRule["Multi-Currency Converter (DKK Priority, EUR, USD)"]
        Heuristics["Date Precedence (Factura vs. Betaling vs. Periode)"]
    end

    subgraph ERPContext ["4. Core Double-Entry ERP Domain"]
        Session["DatabaseManager (SET session.username Injection)"]
        DoubleEntry[("double_entry.typing_output")]
        TaskBots[("task_bots.typer_bot_output")]
    end

    subgraph AutoAccountantContext ["5. Autonomous Bank Reconciliation Domain"]
        BankFeeds[("bank_transaction_rules")]
        O3MiniWorker["o3-mini Rule Synthesis Agent"]
        Matching["Matching Information State Machine"]
    end

    VisionContext -->|Cached OCR & Clean Image| LLMContext
    LLMContext -->|Structured Payloads| ValidationContext
    ValidationContext -->|Validated & Mathematical Invariants| ERPContext
    ERPContext -->|Unreconciled Ledgers| AutoAccountantContext
    AutoAccountantContext -->|Synthesized Rules & Matches| ERPContext
```

---

## 2. Bounded Context Specifications

### 2.1 Computer Vision & Ingestion Domain
- **Responsibilities**: Ingests multi-format receipts from S3/FileViewer, dynamically computes scaling ratios for ultra-high-resolution files (>74M pixels), converts exotic color spaces to standard RGB, and extracts dense text tokens with spatial coordinates.
- **Key Files**: `image_processor.py`, `receipt_processor.py`.

### 2.2 Multimodal AI Extraction Domain
- **Responsibilities**: Submits OCR text and visual crops in parallel to OpenAI GPT-4o using Pydantic structured output models, eliminating JSON formatting hallucinations and field truncation.
- **Key Files**: `receipt_parser.py`, `pydantic_models.py`.

### 2.3 Statutory Accounting & Validation Domain
- **Responsibilities**: Implements Danish tax law (*Bogføringsloven*). Validates that the total gross matches the net plus 25% Moms. Resolves complex multi-currency exchange receipts and handwritten tips.
- **Key Files**: `ai_revalidation.py`, `retry_incorrect_amounts.py`.

### 2.4 Core Double-Entry ERP Domain
- **Responsibilities**: Injects regulatory user identity (`ai-service@accounting.internal`), sanitizes ASCII null-bytes from strings, executes parameterized queries against core double-entry ERP tables, and updates task processing queues.
- **Key Files**: `database_manager.py`, `sql_models.py`.

### 2.5 Autonomous Bank Reconciliation Domain
- **Responsibilities**: Matches bank statement entries against booked receipt invoices, uses `o3-mini` to synthesize regex pattern rules for recurring merchant charges, and invalidates outdated matching state.
- **Key Files**: `auto_accountant.py`, `type_and_reconcile_receipt_bot.py`.

Related: [Architecture](architecture.md) · [Code Examples](code-examples.md) · [ERP Safety](erp-reconciliation-and-safety.md)
