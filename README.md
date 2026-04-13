# Financial Data Normalizer Agent

A production-ready financial data normalization agent built for AI pipelines. Callable by any autonomous agent via MCP, A2A, or x402 — no signup, no API keys required for crypto payments.

**Live endpoint:** `https://niche-agent-production.up.railway.app`

---

## What It Does

Raw financial data is messy. Trade records arrive with inconsistent date formats, missing currency codes, unverified counterparty identifiers, and no regulatory compliance checks. This agent fixes all of that automatically.

It exposes four tools:

| Tool | Data Source | Cost |
|---|---|---|
| `normalize_financial_record` | Claude Haiku | $0.002/call |
| `validate_regulatory_fields` | Claude Haiku | $0.002/call |
| `enrich_counterparty` | GLEIF Global LEI Index (live) | $0.002/call |
| `lookup_instrument` | OpenFIGI (live) | $0.002/call |

The counterparty enrichment and instrument lookup tools call real verified databases — not AI inference — returning the same data used by regulators, central banks, and financial institutions worldwide.

---

## Quick Start

### Discover available tools
```bash
curl https://niche-agent-production.up.railway.app/tools
```

### Normalize a trade record
```bash
curl -X POST https://niche-agent-production.up.railway.app/invoke \
  -H "x-api-key: cus_YOUR_ID.YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "normalize_financial_record",
    "input": {
      "raw_record": "{\"symbol\": \"AAPL US\", \"qty\": \"500\", \"px\": \"198.42 USD\", \"date\": \"04/10/26\"}",
      "record_type": "trade"
    }
  }'
```

### Look up a verified LEI code
```bash
curl -X POST https://niche-agent-production.up.railway.app/invoke \
  -H "x-api-key: cus_YOUR_ID.YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "enrich_counterparty",
    "input": {
      "counterparty_name": "Goldman Sachs International"
    }
  }'
```

Response:
```json
{
  "tool": "enrich_counterparty",
  "output": {
    "lei": "549300R83Q2VTICWNV15",
    "legal_name": "GOLDMAN SACHS INTERNATIONAL",
    "jurisdiction": "GB",
    "status": "ACTIVE",
    "verified": true,
    "source": "GLEIF"
  },
  "model": "GLEIF-API",
  "latency_ms": 1253
}
```

### Look up a verified FIGI
```bash
curl -X POST https://niche-agent-production.up.railway.app/invoke \
  -H "x-api-key: cus_YOUR_ID.YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "lookup_instrument",
    "input": {
      "identifier": "US0378331005"
    }
  }'
```

---

## Payment Options

### Option A — Stripe (for developers)
Get an API key by signing up at [your-signup-url]. Billed monthly via Stripe metered billing.

Pricing:
- 0–1,000 calls: $0.002/call
- 1,001–10,000 calls: $0.0015/call
- 10,001+ calls: $0.001/call

### Option B — x402 (for autonomous agents)
Any agent with a funded Base wallet can call this endpoint with zero signup. The server responds with HTTP 402 + USDC payment instructions, the agent pays, and the data is returned — no humans involved.

```
POST https://niche-agent-production.up.railway.app/invoke
```

Discovery manifest: `/.well-known/x402`
A2A manifest: `/.well-known/agent.json`

Network: Base (eip155:8453)
Token: USDC
Price: $0.002/call

---

## Tool Reference

### normalize_financial_record
Converts raw financial records into structured, validated JSON.

Input:
- `raw_record` (string) — raw data in any format: JSON, CSV row, or free text
- `record_type` (enum) — `transaction`, `trade`, `position`, `price_tick`, `corporate_action`
- `target_schema` (optional string) — e.g. `FIX`, `SWIFT`, `ISO20022`

Output: normalized JSON with standardized dates (ISO 8601), currency codes (ISO 4217), inferred asset class, and a `flags` array for data quality issues.

---

### validate_regulatory_fields
Checks a financial record for regulatory completeness.

Input:
- `record` (object) — financial record as JSON
- `regulation` (enum) — `MiFID_II`, `Dodd-Frank`, `EMIR`, `SEC_Rule_605`

Output: list of missing or invalid fields with remediation guidance.

---

### enrich_counterparty
Returns verified entity data from the GLEIF Global LEI Index.

Input:
- `counterparty_name` (string) — legal entity name

Output: LEI code, verified legal name, jurisdiction, entity status, registered address. Falls back to AI inference for entities not in the LEI registry.

Data source: [GLEIF Global LEI Index](https://www.gleif.org) — the same database used by regulators and central banks worldwide.

---

### lookup_instrument
Returns verified instrument data from OpenFIGI.

Input:
- `identifier` (string) — ISIN, CUSIP, SEDOL, or ticker
- `id_type` (optional enum) — `TICKER`, `ID_ISIN`, `ID_CUSIP`, `ID_SEDOL`

Auto-detects identifier type if not specified.

Output: FIGI code, composite FIGI, instrument name, exchange code, security type, market sector.

Data source: [OpenFIGI](https://www.openfigi.com) — Bloomberg's open financial instrument identifier database.

---

## Protocol Support

| Protocol | Status | Endpoint |
|---|---|---|
| MCP (Model Context Protocol) | Active | `GET /tools` |
| Google A2A | Active | `GET /.well-known/agent.json` |
| x402 | Active | `POST /invoke` |
| HTTP REST | Active | `POST /invoke` |

---

## Integration Examples

### LangChain
```python
from langchain.tools import Tool
import httpx

def call_normalizer(raw_record: str) -> dict:
    response = httpx.post(
        "https://niche-agent-production.up.railway.app/invoke",
        headers={"x-api-key": "cus_YOUR_ID.YOUR_TOKEN"},
        json={
            "tool": "normalize_financial_record",
            "input": {"raw_record": raw_record, "record_type": "trade"}
        }
    )
    return response.json()["output"]

normalizer_tool = Tool(
    name="financial_normalizer",
    description="Normalize raw financial trade records to structured JSON with verified identifiers",
    func=call_normalizer
)
```

### Python (direct)
```python
import httpx

client = httpx.Client(
    base_url="https://niche-agent-production.up.railway.app",
    headers={"x-api-key": "cus_YOUR_ID.YOUR_TOKEN"}
)

# Normalize a trade
result = client.post("/invoke", json={
    "tool": "normalize_financial_record",
    "input": {
        "raw_record": '{"symbol": "AAPL", "qty": 500, "price": 198.42}',
        "record_type": "trade"
    }
})

# Verify a counterparty LEI
lei = client.post("/invoke", json={
    "tool": "enrich_counterparty",
    "input": {"counterparty_name": "JPMorgan Chase"}
})
```

---

## Health Check
```bash
curl https://niche-agent-production.up.railway.app/health
```

```json
{
  "status": "ok",
  "agent": "financial-data-normalizer",
  "payment_rails": ["stripe", "x402"]
}
```

---

## Tags
`mcp` `a2a` `x402` `fintech` `finance` `lei` `gleif` `figi` `openfigi` `compliance` `mifid` `dodd-frank` `emir` `normalization` `financial-data` `trading` `usdc` `base`
