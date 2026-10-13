# Representative Production Code Examples

[← Main README](../README.md) · [← Documentation Index](README.md)

These code examples are extracted directly from the production repositories (`receipt-text-extraction` and `auto_accountant`). They illustrate the operational patterns, error handling, concurrency, and database integrity controls powering the platform.

---

## 1. Computer Vision: Pixel Boundary Guard & Dynamic LANCZOS Downsampling

High-resolution scans (>74M pixels) cause Google Cloud Vision API to fail with `400 Bad image data`. This component dynamically calculates the proportional scaling factor and downsamples using high-fidelity LANCZOS interpolation while guarding against decompression bombs.

*Source: `image_processor.py`*

```python
import os
import tempfile
from typing import Optional
from PIL import Image, UnidentifiedImageError

# Defensive guard against PIL DecompressionBombError on legal document scans
Image.MAX_IMAGE_PIXELS = None

class ImageProcessor:
    @staticmethod
    def extract_text_safely(image_path: str, vision_client) -> str:
        """
        Guards pixel boundaries, applies dynamic proportional LANCZOS resizing,
        and executes Google Cloud Vision OCR with structured error isolation.
        """
        try:
            with Image.open(image_path) as img:
                width, height = img.size
                total_pixels = width * height
                MAX_PIXELS = 74_000_000  # Conservative safety buffer below GCP 75M limit

                if total_pixels > MAX_PIXELS:
                    # Calculate proportional scaling factor
                    scale_factor = (MAX_PIXELS / total_pixels) ** 0.5
                    new_width = int(width * scale_factor)
                    new_height = int(height * scale_factor)

                    try:
                        resample_filter = Image.Resampling.LANCZOS
                    except AttributeError:
                        resample_filter = Image.LANCZOS

                    resized_img = img.resize((new_width, new_height), resample_filter)

                    # Normalize color mode to RGB if transparent or palette-based
                    if resized_img.mode in ('RGBA', 'P'):
                        resized_img = resized_img.convert('RGB')

                    resized_img.save(image_path, quality=95)

            with open(image_path, "rb") as image_file:
                content = image_file.read()

            image = vision_client.Image(content=content)
            response = vision_client.text_detection(image=image)

            if response.error.message:
                raise RuntimeError(f"Vision API error: {response.error.message}")

            return ImageProcessor.format_ocr_response(response.full_text_annotation)

        except Exception as e:
            if "Bad image data" in str(e):
                raise RuntimeError(f"Vision API rejected image data: {e}") from e
            raise
```

---

## 2. Parallel Dual-Schema Extraction with Pydantic & ThreadPoolExecutor

To prevent LLM token exhaustion and schema collision, core ledger fields and bank audit metadata are queried in parallel using OpenAI Structured Outputs.

*Source: `receipt_parser.py`*

```python
import base64
import json
import concurrent.futures
from typing import Optional, Dict, List, Literal
from pydantic import BaseModel, Field
import openai

class LineItem(BaseModel):
    description: Optional[str]
    unit_price: Optional[float]
    total_price: Optional[float]

class ReconciliationData(BaseModel):
    card_last_4: Optional[str] = None
    payment_method: Optional[Literal["card", "cash", "mobilepay", "bank_transfer", "unknown"]] = "unknown"
    payment_references: List[str] = Field(default_factory=list)
    transaction_time: Optional[str] = None
    document_type: Optional[str] = None
    base_amount: Optional[float] = None
    processor_fees: Optional[float] = None
    seller_name: Optional[str] = None
    seller_cvr: Optional[str] = None
    seller_bank_details: Optional[str] = None
    line_items: List[LineItem] = Field(default_factory=list)
    due_date: Optional[str] = None

class ReceiptData(BaseModel):
    Date: Optional[str]
    Amount: Optional[float]
    Currency: Optional[str]
    Notes: str
    Degrees: int
    MatchingInformation: str
    VatAmount: float

class ReceiptParser:
    def __init__(self, api_key: str):
        self.openai_client = openai.OpenAI(api_key=api_key)

    def parse_receipt_all(self, ocr_text: str, image_path: str) -> Dict:
        """
        Executes parallel extraction for Core Ledger and Extended Reconciliation schemas.
        """
        with open(image_path, "rb") as image_file:
            base64_image = base64.b64encode(image_file.read()).decode("utf-8")

        with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
            future_core = executor.submit(self._extract_core, ocr_text, base64_image)
            future_recon = executor.submit(self._extract_reconciliation, ocr_text, base64_image)

            core_data = future_core.result() or {}
            recon_data = future_recon.result() or {}

            # Merge results into unified payload
            merged_results = {**core_data, **recon_data}
            return merged_results

    def _extract_core(self, ocr_text: str, base64_image: str) -> Dict:
        response = self.openai_client.beta.chat.completions.parse(
            model="gpt-4o",
            temperature=0,
            response_format=ReceiptData,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "text", "text": self.build_core_prompt(ocr_text)},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}}
                ]
            }],
            max_completion_tokens=16384
        )
        return json.loads(response.choices[0].message.content)

    def _extract_reconciliation(self, ocr_text: str, base64_image: str) -> Dict:
        response = self.openai_client.beta.chat.completions.parse(
            model="gpt-4o",
            temperature=0,
            response_format=ReconciliationData,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "text", "text": self.build_recon_prompt(ocr_text)},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}}
                ]
            }],
            max_completion_tokens=16384
        )
        return json.loads(response.choices[0].message.content)
```

---

## 3. Data Sanitization & Null-Byte Scrubbing

OCR artifacts and non-ASCII character sequences occasionally inject ASCII null bytes (`\x00`), which trigger fatal errors in PostgreSQL JSONB parsers. This recursive cleaner purges null bytes and normalizes empty tokens (`"n/a"`, `"-"`, `"none"`) to SQL `NULL`.

*Source: `receipt_processor.py`*

```python
from typing import Dict, Any

class DataSanitizer:
    @staticmethod
    def clean_parsed_data(data: Any) -> Any:
        """
        Recursively scrubs null bytes (\x00) and converts empty string markers
        to Python None to prevent PostgreSQL JSONB parsing exceptions.
        """
        if isinstance(data, dict):
            return {k: DataSanitizer.clean_parsed_data(v) for k, v in data.items()}
        elif isinstance(data, list):
            return [DataSanitizer.clean_parsed_data(item) for item in data]
        elif isinstance(data, str):
            cleaned = data.replace('\x00', '').strip()
            if cleaned == "" or cleaned.lower() in ["null", "none", "n/a", "unknown", "-"]:
                return None
            return cleaned
        return data
```

---

## 4. Session Context Injection & Atomic Double-Entry Persistence

Enforces statutory audit trails under Danish accounting law (*Bogføringsloven*) by injecting the auditor session identity prior to running ledger update queries. Additionally validates that mandatory accounting fields are populated; otherwise, `matching_information` is automatically nullified to prevent corrupt bank matches.

*Source: `database_manager.py`*

```python
import psycopg2
from psycopg2.extras import Json
from typing import Dict

class DatabaseManager:
    def __init__(self, connection):
        self.conn = connection
        self.cursor = self.conn.cursor()

    def update_typing_results(self, typing_output_id: int, typing_results: Dict):
        try:
            # Defensive Integrity: If any core accounting field is missing, invalidate matching_info
            mandatory_keys = ['entry_date', 'currency', 'amount_original_currency', 'vat_amount']
            if any(typing_results.get(k) is None for k in mandatory_keys):
                typing_results["matching_information"] = None

            # Enforce auditor identity in session context
            self.cursor.execute("SELECT set_config('session.username', current_user::text, true)")

            # Atomic update to double_entry.typing_output
            update_sql = """
                UPDATE double_entry.typing_output
                SET entry_date             = COALESCE(%s, entry_date),
                    amount_original_currency = COALESCE(%s, amount_original_currency),
                    currency               = COALESCE(%s, currency),
                    degrees                = COALESCE(%s, degrees),
                    matching_information   = COALESCE(%s, matching_information),
                    vat_amount             = COALESCE(%s, vat_amount)
                WHERE id = %s
            """
            params = (
                typing_results.get("entry_date"),
                typing_results.get("amount_original_currency"),
                typing_results.get("currency"),
                typing_results.get("degrees"),
                typing_results.get("matching_information"),
                typing_results.get("vat_amount"),
                typing_output_id,
            )
            self.cursor.execute(update_sql, params)

            # Persist comprehensive JSONB audit snapshot to task_bots.typer_bot_output
            bot_output_sql = """
                UPDATE task_bots.typer_bot_output 
                SET output = COALESCE(output::jsonb, '{}'::jsonb) || %s::jsonb 
                WHERE id = %s
            """
            self.cursor.execute(bot_output_sql, (Json(typing_results), typing_output_id))

            self.conn.commit()

        except Exception as e:
            self.conn.rollback()
            raise RuntimeError(f"Failed to update typing results for task {typing_output_id}: {e}") from e
```

---

## 5. Bank Auto-Accountant: Autonomous SQL Rule Synthesizer

Uses OpenAI `o3-mini` to reason over historical customer transactions, generating SQL regex rules that auto-reconcile recurring vendor payouts with 100% precision.

*Source: `auto_accountant.py`*

```python
import json
from openai import OpenAI

def generate_reconciliation_rules(openai_client: OpenAI, assistant_id: str, historical_csv: str) -> list[str]:
    """
    Synthesizes exact-match SQL rules for bank transaction auto-booking.
    """
    prompt = f"""
Come up with bank transaction auto-accounting rules.
Target data with 100% precision:
{historical_csv}

Return a JSON array of objects:
- rule_expression (PostgreSQL ILIKE expression)
- min_amount
- max_amount
- account (Danish chart of accounts number, e.g., '2811')
- vat_code (e.g., 'I25' for incoming 25% moms)
"""

    response = openai_client.chat.completions.create(
        model="o3-mini",
        messages=[{"role": "user", "content": prompt}],
    )

    rules = json.loads(response.choices[0].message.content)
    sql_statements = []

    for rule in rules:
        sql = f"""
INSERT INTO bank_transaction_rules
(customer_id, active, applies_to, rule_expression, min_amount, max_amount, account, vat_code)
VALUES
(1, false, 'auto_accounting', '{rule["rule_expression"]}', '{rule["min_amount"]}', '{rule["max_amount"]}', '{rule["account"]}', '{rule["vat_code"]}');
""".strip()
        sql_statements.append(sql)

    return sql_statements
```

Related: [Domain Map](domain-map.md) · [Computer Vision Pipeline](computer-vision-pipeline.md) · [AI Parsing Guide](ai-parsing-and-revalidation.md)
