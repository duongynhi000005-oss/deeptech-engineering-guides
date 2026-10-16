# Building a Zero-Dependency FIFO/LIFO Crypto & Freelance Tax Engine in Python

**Tags:** `python`, `fintech`, `blockchain`, `accounting`

When you earn engineering revenue across GitHub Bounties (Algora), Stripe/PayPal invoices, Ko-fi digital downloads, and on-chain token payouts (`USDC`, `SOL`, `ETH`, `XLM`), standard spreadsheet templates fall apart for three reasons:

1. **Floating-Point Rounding Drift (`IEEE 754`):** Representing currency or token quantities with Python `float` introduces binary rounding errors (`0.1 + 0.2 == 0.30000000000000004`) that accumulate across thousands of swaps.
2. **Duplicate Webhook & Indexer Exports:** Merging CSV exports from Solana/Stellar RPC indexers and payment gateways inevitably introduces duplicate rows when retries occur.
3. **FIFO vs. LIFO Cost-Basis Lot Matching:** When you receive `10 SOL` from a bounty at `$140/SOL`, another `5 SOL` at `$180/SOL`, and later swap `12 SOL` for `USDC` at `$195/SOL` with a `$1.50` gas fee, your realized capital gain depends entirely on how partial tax lots are consumed and whether disposal fees reduce net proceeds.

In this tutorial, we will build a **deterministic, zero-dependency Python 3 reconciliation engine** using `decimal.Decimal` and `hashlib.sha256`.

> **Want the Ready-Made 5-Sheet Dark-Mode Excel OS + Python CLI?** Grab the complete [**Deep-Tech Freelance, Bounty & Crypto FIFO/LIFO Tax OS ($5)**](https://ko-fi.com/s/6f070d3e68) on Ko-fi (or inspect the [GitHub Release v1.0.0](https://github.com/duongynhi000005-oss/deeptech-crypto-freelance-tax-os/releases/tag/v1.0.0)).

---

## Step 1: Canonical SHA-256 Deduplication Across CSV Exports

Payment processors and blockchain indexers frequently export the same transaction twice (e.g., once from a webhook log and once from a monthly CSV statement). By normalizing the transaction ID (or hashing the canonical tuple `timestamp|source|type|asset|qty|usd_price`), we eliminate duplicate rows in $O(1)$ time:

```python
from __future__ import annotations
import hashlib
from decimal import Decimal, ROUND_HALF_UP
from dataclasses import dataclass
from typing import List, Set


def d(val: str | int | float | Decimal) -> Decimal:
    return Decimal(str(val))


def q2(val: Decimal) -> Decimal:
    return val.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)


@dataclass(frozen=True)
class LedgerRow:
    tx_id: str
    timestamp: str
    category: str  # INCOME, CRYPTO_BUY, CRYPTO_DISPOSAL, EXPENSE
    asset: str
    quantity: Decimal
    unit_price_usd: Decimal
    fee_usd: Decimal

    def canonical_key(self) -> str:
        clean_id = self.tx_id.strip().lower()
        if clean_id:
            return f"id:{clean_id}"
        raw = f"{self.timestamp}|{self.category}|{self.asset}|{self.quantity.normalize()}|{self.unit_price_usd.normalize()}"
        return "hash:" + hashlib.sha256(raw.encode("utf-8")).hexdigest()


def deduplicate_rows(rows: List[LedgerRow]) -> List[LedgerRow]:
    seen: Set[str] = set()
    unique: List[LedgerRow] = []
    for row in rows:
        key = row.canonical_key()
        if key not in seen:
            seen.add(key)
            unique.append(row)
    return unique
```

---

## Step 2: Partial-Lot FIFO & LIFO Cost-Basis Matching

Every time you receive a token via a bounty (`INCOME` in a non-USD crypto asset) or purchase a token (`CRYPTO_BUY`), it creates a **Tax Lot** whose unit cost basis includes acquisition fees:

$$\text{Unit Cost USD} = \frac{\text{Quantity} \times \text{Spot Price USD} + \text{Acquisition Fee USD}}{\text{Quantity}}$$

When a `CRYPTO_DISPOSAL` occurs, we pop lots from the **front** of the queue for **FIFO** (First-In, First-Out) or from the **back** of the queue for **LIFO** (Last-In, First-Out), splitting lots cleanly when a disposal only consumes part of a lot:

```python
@dataclass
class TaxLot:
    asset: str
    remaining_qty: Decimal
    unit_cost_usd: Decimal


def compute_realized_gains(rows: List[LedgerRow], mode: str = "FIFO") -> Decimal:
    lots_by_asset: dict[str, list[TaxLot]] = {}
    total_realized_gain = Decimal("0")

    for row in rows:
        asset = row.asset.upper()
        if row.category in ("CRYPTO_BUY", "CRYPTO_BOUNTY") and asset != "USD":
            total_cost = (row.quantity * row.unit_price_usd) + row.fee_usd
            unit_cost = total_cost / row.quantity
            lots_by_asset.setdefault(asset, []).append(
                TaxLot(asset=asset, remaining_qty=row.quantity, unit_cost_usd=unit_cost)
            )
        elif row.category == "CRYPTO_DISPOSAL" and asset != "USD":
            pool = lots_by_asset.get(asset, [])
            qty_to_dispose = row.quantity
            net_proceeds = (row.quantity * row.unit_price_usd) - row.fee_usd
            matched_cost = Decimal("0")

            while qty_to_dispose > Decimal("0") and pool:
                lot = pool[0] if mode == "FIFO" else pool[-1]
                take = min(qty_to_dispose, lot.remaining_qty)
                matched_cost += take * lot.unit_cost_usd
                lot.remaining_qty -= take
                qty_to_dispose -= take
                if lot.remaining_qty == Decimal("0"):
                    pool.pop(0 if mode == "FIFO" else -1)

            total_realized_gain += net_proceeds - matched_cost

    return q2(total_realized_gain)
```

---

## Step 3: Comparing FIFO vs. LIFO in Rising Markets

In a rising token market (e.g., acquiring `SOL` at `$120` in January and `$175` in March, then selling half your position at `$190` in April):
- **FIFO** matches against the older `$120` lot first, resulting in a **higher immediate realized capital gain** (`$70/SOL`).
- **LIFO** (where permitted under specific lot-identification rules) matches against the newer `$175` lot first, resulting in a **lower immediate realized gain** (`$15/SOL`) while preserving the `$120` cost basis on the remaining lot.

Running both calculations side-by-side gives you instant visibility into your quarterly estimated tax reserve before year-end.

---

## Get the Full 5-Sheet Dark-Mode Excel OS + Python CLI Bundle ($5)

Instead of wiring up Excel charts and quarterly tax schedules from scratch, download the complete bundle:
- **Ko-fi Instant Download ($5):** [Deep-Tech Freelance, Bounty & Crypto FIFO/LIFO Tax OS (Pro v1.0)](https://ko-fi.com/s/6f070d3e68)
- **FastAPI + WebSocket Real-Time AI Starter Kit ($5):** [Ko-fi Shop](https://ko-fi.com/nhi741323)
- **GitHub Release `v1.0.0`:** [duongynhi000005-oss/deeptech-crypto-freelance-tax-os](https://github.com/duongynhi000005-oss/deeptech-crypto-freelance-tax-os/releases/tag/v1.0.0)
