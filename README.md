# CircularFlow — Circular Economy Intelligence Agent

A production-grade multi-agent system that classifies returned IKEA products, estimates residual value, and routes them to the optimal circular channel (resell, repair, donate, recycle). Built as a demonstration of agentic AI applied to circular economy operations using publicly available IKEA product data and Buy Back pricing.

---

## What it does

IKEA's goal is to be fully circular by 2030. The operational challenge: every returned or end-of-life product needs a routing decision — resell in the As-Is section, send for repair, donate to a charity partner, or recycle. Today that decision is largely rule-based and manual. This system makes it data-driven and automatic.

```
Product return event
        ↓
🔍  Condition Assessment Agent   — classifies damage level from product description
        ↓
💶  Value Estimation Agent       — estimates residual value using age, category, condition
        ↓
📊  Demand Signal Agent          — checks whether refurbished demand exists for this SKU × region
        ↓
🔀  Channel Routing Agent        — recommends: resell / repair / donate / recycle
        ↓
✅  Evaluation + HITL Agent      — quality gate + human approval before action
        ↓
📄  Routing Decision             — channel, confidence score, reasoning trace, audit trail
```

---

## Architecture

### Multi-agent design

Agents communicate through typed message contracts. Each agent produces a structured output that the next agent consumes — no agent calls another directly. This means any agent can be swapped, retested, or extended independently.

```
ReturnEvent → ConditionReport → ValuationReport → DemandReport → RoutingRecommendation → ApprovedDecision
```

### Agents

| Agent | Role | Key output |
|---|---|---|
| `ConditionAssessmentAgent` | Classifies damage level (none / cosmetic / functional / severe) from product description and metadata | `ConditionReport` with damage class and confidence |
| `ValueEstimationAgent` | Estimates residual EUR value using IKEA Buy Back pricing tables, product age, and condition class | `ValuationReport` with estimated value and depreciation curve |
| `DemandSignalAgent` | Checks whether refurbished demand exists for this SKU in the relevant region, adjusted for condition | `DemandReport` with demand score and regional context |
| `ChannelRoutingAgent` | Recommends the optimal circular channel based on condition, value, and demand signal | `RoutingRecommendation` with primary channel and fallback |
| `EvaluationAgent` | Scores the routing recommendation on four dimensions before delivery; escalates to HITL if below threshold | Quality-gated `ApprovedDecision` |
| `Orchestrator` | Coordinates all agents, owns the message bus, writes audit trail | `PipelineResult` with full reasoning trace |

### Routing channels

| Channel | Condition | Value | Demand |
|---|---|---|---|
| **Resell (As-Is)** | Cosmetic only | High | Strong |
| **Repair** | Functional damage | Medium | Moderate+ |
| **Donate** | Any | Low | Weak or zero |
| **Recycle** | Severe / unrepairable | None | N/A |

### Evaluation scoring

Routing recommendations are scored on four dimensions before any human sees them:

| Dimension | What it checks |
|---|---|
| Factual grounding | All claims traceable to condition, valuation, and demand data |
| Causal validity | Channel recommendation logically follows from the input signals |
| Consistency | No internal contradictions across condition, value, and demand |
| Confidence calibration | Stated confidence reflects actual signal strength |

Recommendations scoring below the threshold (default 3.5/5) are automatically escalated to a human reviewer rather than auto-approved.

---

## Data sources

All data is publicly available — no IKEA proprietary data required.

| Source | What it provides | How it's used |
|---|---|---|
| IKEA Product API | Product category, dimensions, materials, current retail price | Baseline for condition classification and value estimation |
| IKEA Buy Back price tables | Published buyback prices per category and condition tier | Ground truth for residual value model |
| Synthetic return records | Generated product return events (SKU, age, condition description) | Test and demonstration data |
| EU circular economy statistics | Furniture return and recycling rates by category | Prior probabilities for routing model |

### Generating synthetic return data

```bash
python3 generate_returns.py --n 500 --categories sofa,bookcase,chair,desk,lamp
```

Generates realistic return events with product age distributions, condition descriptions, and damage types calibrated to IKEA's published sustainability data.

---

## Tech stack

- **Orchestration** — LangGraph (multi-agent coordination, typed message bus)
- **LLM** — Anthropic Claude (claude-sonnet-4-6) for condition classification and routing narrative
- **Data** — DuckDB for product catalogue and return records
- **Embeddings** — ChromaDB for semantic similarity on condition descriptions
- **Dashboard** — Streamlit + Plotly
- **Language** — Python 3.9+

---

## Project structure

```
circularflow/
├── run.py                      # CLI entry point
├── dashboard.py                # Streamlit dashboard
├── generate_returns.py         # Synthetic return data generator
├── agents/
│   ├── __init__.py
│   ├── messages.py             # Typed message contracts
│   ├── condition_agent.py      # Damage classification
│   ├── value_agent.py          # Residual value estimation
│   ├── demand_agent.py         # Demand signal retrieval
│   ├── routing_agent.py        # Channel recommendation
│   ├── eval_agent.py           # Quality gate + HITL escalation
│   └── orchestrator.py         # Pipeline coordinator
├── data/
│   ├── ikea_products.db        # DuckDB product catalogue
│   ├── buyback_prices.json     # Scraped Buy Back pricing tables
│   └── returns.db              # Return event records
└── reports/                    # Auto-created timestamped JSON audit trails
```

---

## Setup

### Prerequisites

- Python 3.9+
- Anthropic API key

### Installation

```bash
git clone https://github.com/sudiptamandal1983-web/circularflow.git
cd circularflow

python3 -m venv .venv
source .venv/activate

pip install duckdb pandas anthropic streamlit plotly langchain langgraph chromadb
```

### Environment variables

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Load IKEA product data

```bash
python3 -c "
import json, duckdb
con = duckdb.connect('data/ikea_products.db')
con.execute(\"CREATE TABLE products AS SELECT * FROM read_json_auto('data/ikea_catalogue.json')\")
con.close()
print('Done')
"
```

---

## Usage

```bash
# Basic run — single return event
python3 run.py --sku S89028673 --age 3 --condition "scratched surface on left panel, hinge intact"

# With LLM reasoning
python3 run.py --sku S89028673 --age 3 --condition "..." --llm

# Full pipeline with evaluation gate
python3 run.py --sku S89028673 --age 3 --condition "..." --llm --eval

# Batch processing from return records file
python3 run.py --batch data/returns.db --llm --eval

# Interactive dashboard
streamlit run dashboard.py
```

### CLI options

| Flag | Default | Description |
|---|---|---|
| `--sku` | required | IKEA product SKU |
| `--age` | required | Product age in years |
| `--condition` | required | Free-text condition description |
| `--llm` | off | Use Claude for reasoning and narrative |
| `--eval` | off | Score recommendations, escalate if below threshold |
| `--eval-threshold` | 3.5 | Minimum score (1–5) to auto-approve |
| `--batch` | off | Process all records from a DuckDB file |
| `--region` | NL | Region for demand signal lookup |

---

## Sample output

**Resell recommendation — bookcase in good condition**

```json
{
  "sku": "S89028673",
  "product": "BILLY Bookcase",
  "condition_class": "cosmetic",
  "condition_confidence": 0.91,
  "residual_value_eur": 28.50,
  "buyback_reference_eur": 30.00,
  "demand_score": 0.78,
  "region": "NL",
  "recommended_channel": "resell",
  "fallback_channel": "donate",
  "reasoning": "Product shows cosmetic damage only (scratched surface, intact structure). Residual value EUR 28.50 vs Buy Back reference EUR 30.00. Demand for second-hand BILLY bookcases in NL is strong (score 0.78). Recommend resell in As-Is section at EUR 25-28.",
  "eval_score": 4.3,
  "approved": true,
  "audit_trail": "reports/2026-06-01T14:23:11.json"
}
```

**Escalated to HITL — ambiguous damage description**

```json
{
  "condition_class": "functional",
  "condition_confidence": 0.54,
  "eval_score": 2.9,
  "approved": false,
  "escalation_reason": "Condition confidence below threshold. Human review required before routing decision.",
  "audit_trail": "reports/2026-06-01T14:31:07.json"
}
```

---

## Why this matters for IKEA

IKEA processes millions of returns annually across 60+ markets. The current routing process relies on in-store staff making manual condition assessments against fixed rules. This system:

- Makes the routing decision consistent and data-driven
- Captures residual value more accurately using real-time Buy Back pricing
- Surfaces demand signals before routing — avoiding donating items that could be resold
- Produces a full audit trail for IKEA's ESG reporting requirements
- Escalates ambiguous cases to human review rather than defaulting to a low-value channel

The architecture is designed to be modular — new channels (repair partners, material recovery) can be added as new agents without touching existing pipeline logic.

---

## Roadmap

- [ ] Image-based condition assessment (CLIP embeddings on product photos)
- [ ] Multi-region demand aggregation across IKEA store network
- [ ] Integration with IKEA's circular hub inventory system
- [ ] Carbon impact scoring per routing decision
- [ ] Repair cost estimation agent (repair vs resell economics)

---

## Relevance to circular economy

This project was built to explore a real operational problem in IKEA's circular economy strategy. The routing decision — where a returned product goes next — is where circular economy commitments either succeed or fail in practice. An item that ends up in landfill because the condition assessment was too conservative, or donated when it could have been resold, is a failure of the system, not of the intention.

The AI problem here is genuinely hard: condition classification from free-text descriptions is noisy, residual value estimation requires calibrated pricing models, and demand signals vary by region, season, and product category. Getting it right requires the same rigour as any production ML system — evaluation frameworks, confidence scoring, human oversight for edge cases, and audit trails for accountability.

---

## License

MIT
