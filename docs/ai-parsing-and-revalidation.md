# AI Parsing, Danish Tax Heuristics & Mathematical Revalidation

[← Main README](../README.md) · [← Documentation Index](README.md)

## Danish Accounting Domain & Regulatory Context

Operating under the Danish Bookkeeping Act (*Bogføringsloven*) and compliance guidelines established by the Danish Tax Agency (**SKAT**) requires exact financial precision. Unlike generalized receipt processors, the Win-Man AI pipeline incorporates deterministic Danish tax heuristics, strict date precedence hierarchies, and currency normalization rules.

```mermaid
flowchart TD
    INPUT[Raw OCR Stream + Downsampled Base64 Image] --> PROMPT_ENGINE[Structured Multimodal Prompts (temp=0)]
    PROMPT_ENGINE --> DUAL_LLM["Parallel GPT-4o Invocations"]

    subgraph CoreParsing["Core Extraction (ReceiptData)"]
        DUAL_LLM --> DATE_RULE["Date Hierarchy Resolution"]
        DUAL_LLM --> CURR_RULE["Currency & Handwritten Override"]
        DUAL_LLM --> AMOUNT_RULE["Gross Amount & Moms Extraction"]
    end

    subgraph ReconParsing["Bank Audit (ReconciliationData)"]
        DUAL_LLM --> CVR_RULE["8-Digit CVR Commercial ID"]
        DUAL_LLM --> IBAN_RULE["IBAN & Bank Account Details"]
        DUAL_LLM --> LINE_ITEMS["Itemized Product Breakdown"]
    end

    DATE_RULE --> MERGE[Merge & Validate Payload]
    CURR_RULE --> MERGE
    AMOUNT_RULE --> MERGE
    CVR_RULE --> MERGE
    IBAN_RULE --> MERGE
    LINE_ITEMS --> MERGE

    MERGE --> MATH_CHECK{"Statutory 25% VAT Verification:<br/>Brutto == Netto * 1.25?"}
    MATH_CHECK -->|Passed| STAGE_PERSIST[Stage Validated Payload for DB Session]
    MATH_CHECK -->|Discrepancy / Incomplete| INVALIDATE_MATCH["Wipe matching_information (Set to None)<br/>Flag for Auditor Review"]
    INVALIDATE_MATCH --> STAGE_PERSIST
```

---

## 1. Danish VAT (*Moms*) Statutory Mathematics

In Denmark, standard commercial transactions are subject to a statutory **25% VAT rate** (*Moms*). Financial ledger entries require exact separation of gross (*Brutto*), net (*Netto*), and tax (*Moms*) figures:

$$\text{Brutto} = \text{Netto} \times 1.25$$
$$\text{Netto} = \text{Brutto} - \text{MOMS}$$
$$\text{MOMS} = \text{Brutto} \times 0.20 = \text{Netto} \times 0.25$$

### Deterministic Cross-Validation Mechanics
1. **Explicit VAT Printed**: If the receipt prints both gross total and Moms, the system validates the arithmetic relation:
   $$|\text{Brutto} - (\text{Netto} + \text{MOMS})| \le 0.05 \text{ DKK}$$
   A tolerance of $0.05$ DKK accommodates commercial cent rounding rules.
2. **Missing or Blurred VAT**: If the receipt omits a separate VAT line (common on thermal supermarket tickets and handwritten slips), the engine derives the tax dynamically:
   $$\text{Derived VAT} = \text{Total Amount} \times 0.20$$
   This calculated value is populated into `VatAmount`, enabling seamless downstream automated double-entry booking using Danish tax codes `I25` (incoming domestic VAT) or `U25` (outgoing VAT).

---

## 2. Strict Date Priority Hierarchy

Receipts and invoices frequently feature multiple contradictory dates: issue date, order placement date, payment date, billing interval dates, and print timestamps. Under Danish tax audits, registering the wrong date causes ledger misallocations across quarterly VAT reporting periods.

The system enforces a strict priority hierarchy:

```
Priority 1: Factura Dato / Issue Date / Purchase Date / Billing Date
    ↓
Priority 2: Order Dato (Order placement date)
    ↓
Priority 3: Customer Handwritten Date (Pen annotation on physical slip)
    ↓
Priority 4: Betalingsdato (Payment / bank clearance date)
    ↓
Priority 5: Document Printing Date (Last resort timestamp)
```

### Domain-Specific Date Edge Cases
- **SKAT Official Documents**: Invoices and notifications issued by the Danish Tax Agency (*SKAT*) strictly extract the date designated by the phrase **`"modtaget den"`** (received date), disregarding issuance dates.
- **Utility & Telephony Bills (*Forbrugerperiode*)**: Periodic service bills (e.g., electricity, mobile carriers) feature date intervals (e.g., `Forbrugerperiode 11.11.2024 - 10.12.2024`). The engine strictly selects the **last/closing date of the interval** (`2024-12-10`).
- **Telephony Bills**: For telecommunication invoices, extracts the date explicitly marked with **`"til"`** (until).
- **Format Normalization**: All dates are standardized to ISO-8601 (`YYYY-MM-DD`). Ambiguous expressions (e.g., `04-05-06`) are rejected as `null` rather than misattributed.

---

## 3. Handwritten Overrides & Multi-Currency Handling

Danish entrepreneurs traveling abroad frequently incur expenses in foreign currencies (EUR, USD, GBP, SEK). Upon returning, company directors or accountants write the actual converted DKK amount deducted from their Danish corporate credit card directly onto the slip with a pen:

$$\text{Printed on Receipt:} \quad \mathbf{77.00 \text{ EUR}} \quad \longrightarrow \quad \text{Handwritten Annotation:} \quad \mathbf{\text{"(575)"}}$$

### Prioritization Rules
1. **Handwritten Precedence**: If any amount on a foreign receipt is manually edited or circled in pen, the engine prioritizes the handwritten value over the printed machine text.
2. **Currency Normalization**: When a handwritten DKK amount accompanies a printed non-DKK figure, the currency is strictly normalized to `DKK`.
3. **Implicit Currency Resolution**: If currency markers (`kr.`) are ambiguous, the engine scans the vendor address:
   - Danish municipalities (Copenhagen, Aarhus, Odense, Aalborg, Frederiksberg, etc.) default to `DKK`.
   - Swedish/Norwegian postal addresses resolve to `SEK` or `NOK`.

---

## 4. Special Document Routing & Deletion Criteria

Not every scanned document is a standard commercial receipt. The pipeline routes special document types through tailored accounting semantics:

| Document Type | Extraction Behavior | Accounting Ledger Impact |
|---|---|---|
| **Kreditnota (Credit Note)** | Extracts parent invoice reference (`kreditnota fakt <nr>`) | Offsets existing accounts payable/receivable with negative ledger entries |
| **Gebyr (Payment Processing Fees)** | Extracts gateway fee totals (Stripe, PayPal, Nets) | Books fees to finance expense account (`3600`) separate from gross revenue |
| **LØN Documents (Payroll)** | Flags `Notes = "LON"`, leaves amounts blank | Prevents accidental posting of payroll records to vendor expense ledgers |
| **Booking Cancellations** | Flags `Notes = "booking annulation"`, leaves blank | Suppresses transaction booking |
| **Blurred / Unreadable Scans** | Leaves fields blank, sets `expense = "no"` | Diverts task to manual accountant resolution pool |

---

## 5. Prompt Engineering for Zero Hallucination

To ensure the LLM never fabricates financial data, prompts are engineered under zero-temperature constraints with explicit negative assertions:

1. **`temperature = 0`**: Enforces fully deterministic token sampling across all API invocations.
2. **Explicit Null Enforcement**: The system prompt instructs:
   > *"Do NOT assume, infer, or guess any values. If you are not 100% absolutely certain, you MUST return `null`. False positives or hallucinated data are strictly forbidden. There is zero margin for error."*
3. **Structured Outputs (`response_format`)**: Enforces JSON schema adherence at the OpenAI API kernel level, guaranteeing that output payloads always conform to the Pydantic schema types.
4. **Context Length Protection**: High-line-item invoices (e.g., 500-line wholesale supply sheets) risk generation timeouts. The prompt instructs the model to truncate line item arrays for statements while preserving gross header totals.

Related: [Domain Map](domain-map.md) · [Code Examples](code-examples.md) · [ERP Reconciliation](erp-reconciliation-and-safety.md)
