# 🧠 Hackathon Cortex Code Experience — Madrid 2026

> **Snowflake Hackathon** | 27 May 2026 | 09:30 – 13:30 | Snowflake Madrid offices
> Register: https://www.snowflake.com/event/hackathon-cortex-code-experience-madrid-20260527/

---

## What is this?

A technical hackathon where teams build an **industry-specific AI copilot** on Snowflake from scratch, using real data and CoCo (Cortex Code) as the AI assistant. The challenge is a guided sprint across 5 phases:

1. **Bronze** — Git Integration + JSON data ingestion
2. **Silver + Gold** — data transformations with dbt
3. **AI** — AI views with Cortex COMPLETE (LLaMA 3.1-70B)
4. **Semantic View + Cortex Agent** — conversational AI over your data
5. **Streamlit in Snowflake** — interactive analytics app

> **Special industry:** The Insurance use case includes a **FASE 4 Machine Learning** phase (churn detection with `SNOWFLAKE.ML.CLASSIFICATION`) and **5 AI use cases** on real 5–10 minute Spanish call transcripts.

---

## Industries (10 total)

| # | Industry | Database | English Guide | Data folder |
|---|----------|----------|---------------|-------------|
| 1 | 📡 **Telecommunications** | TELCO_COPILOT | guias/en/guia_telco.md | data/telco/ |
| 2 | 🏦 **Financial Services** | FSI_COPILOT | guias/en/guia_fsi.md | data/fsi/ |
| 3 | 🛍️ **Retail & Consumer Goods** | RETAIL_COPILOT | guias/en/guia_retail.md | data/retail/ |
| 4 | 🏭 **Manufacturing & Industry** | MANUFACTURING_COPILOT | guias/en/guia_manufacturing.md | data/manufacturing/ |
| 5 | 🏥 **Healthcare & Life Sciences** | HEALTHCARE_COPILOT | guias/en/guia_healthcare.md | data/healthcare/ |
| 6 | 📺 **Media, Entertainment & Ads** | MEDIA_COPILOT | guias/en/guia_media.md | data/media/ |
| 7 | 🚚 **Transport, Logistics & Travel** | LOGISTICS_COPILOT | guias/en/guia_logistics.md | data/logistics/ |
| 8 | 🌐 **Public Sector** | PUBLIC_SECTOR_COPILOT | guias/en/guia_public_sector.md | data/public_sector/ |
| 9 | 💧 **Utilities & Water Management** | UTILITIES_COPILOT | guias/en/guia_utilities.md | data/utilities/ |
| 10 | 🛡️ **Insurance — Call Center AI** | INSURANCE_COPILOT | guias/en/guia_insurance.md | data/insurance/ |

---

## Architecture

### Standard industries (1–9)

```
[Git Repo]
    │
    ├─ JSON data
    │
    ▼
[BRONZE]  ──────────────────── RAW tables (VARIANT)
    │
    ▼  (dbt staging)
[SILVER]  ──────────────────── Flattened/typed tables (STG_*)
    │
    ▼  (dbt marts)
[GOLD]    ──────────────────── Business models (CUSTOMER_360, RISK, HEALTH...)
    │                                   │
    ▼ Cortex COMPLETE                   ▼ Semantic View
[AI]      ──── Views (VW_*)        SV_*_COPILOT
                                        │
                                   Cortex Agent
                                        │
                              Snowflake Intelligence (chat)
                                        │
                              Streamlit in Snowflake (app)
```

### Insurance + ML (industry 10)

```
[Git Repo]
    │
    ├─ JSON data (customers + call_transcripts + claims + agents)
    │
    ▼
[BRONZE]  ──────────────────── RAW tables (VARIANT)
    │
    ▼  (dbt)
[SILVER]  ──────────────────── STG_CUSTOMERS, STG_CALLS, STG_CLAIMS, STG_AGENTS
    │
    ▼  (dbt)
[GOLD]    ──────────────────── CUSTOMER_RISK_360, CALL_INTELLIGENCE_360, AGENT_PERFORMANCE
    │                                   │
    ▼ Cortex COMPLETE                   ├── Semantic View SV_INSURANCE_COPILOT
[AI]      ──── 5 views                  │
    │  (summaries, routing,             ▼
    │   QA, CX, churn)            Cortex Agent ──► Snowflake Intelligence
    │
    ▼ SNOWFLAKE.ML.CLASSIFICATION
[ML]      ──────────────────── CHURN_FEATURES → CHURN_MODEL → CHURN_PREDICTIONS
                                        │
                              Streamlit in Snowflake (INSURANCE_APP)
```

---

## Datasets

### Standard industries (1–9): 4 files per industry

| File | Approximate records | Description |
|------|---------------------|-------------|
| `customers.json` / `clients.json` / `plants.json` | ~350 | Main entities |
| `events.json` / `transactions.json` / `orders.json` | ~10,000 | Transactional activity |
| `tickets.json` / `alerts.json` / `incidents.json` | ~750 | Issues / alerts |
| `billing.json` / `accounts.json` / `batches.json` | ~2,000 | Billing / secondary data |

### Insurance (industry 10)

| File | Records | Description |
|------|---------|-------------|
| `customers.json` | 350 | Insurance customers (auto/home), premium, NPS, years as customer |
| `call_transcripts.json` | 500 | **Full Spanish call transcripts** (avg ~430 words each). 8 templates: claim auto, claim home, cancellation intent, complaint, coverage query, renewal, vehicle change, roadside assistance |
| `claims.json` | 1,000 | Claims (type, amounts, franchise, resolution days) |
| `agents.json` | 50 | Call center agents (team, QA score, FCR rate, avg handle time) |

---

## Insurance AI Use Cases (FASE 3 + FASE 4)

| # | View | Description |
|---|------|-------------|
| 1 | `VW_CALL_SUMMARY` | Structured executive summary per call: intent, resolution, CSAT, key actions |
| 2 | `VW_CALL_ROUTING` | Intelligent routing classification: department, urgency, confidence |
| 3 | `VW_AGENT_QA` | Automatic agent QA: communication quality, protocol compliance, empathy score |
| 4 | `VW_CX_INSIGHTS` | CX insights: sentiment, emotion, NPS drivers, verbatim topics |
| 5 | `VW_CHURN_SIGNALS` | Churn signals: detected intent, urgency, estimated revenue at risk |
| ML | `CHURN_MODEL` | `SNOWFLAKE.ML.CLASSIFICATION` churn prediction with 12 features from Gold + AI |

---

## Agenda

| Time | Activity |
|------|----------|
| 09:30 | Welcome & setup |
| 09:45 | FASE 0: Database + Git Integration (10 min) |
| 10:00 | FASE 1: Bronze data ingestion (20 min) |
| 10:20 | FASE 2: Silver + Gold with dbt (30 min) |
| 10:50 | FASE 3: AI views with Cortex COMPLETE (30 min) |
| 11:20 | *Coffee break (15 min)* |
| 11:35 | FASE 4: Machine Learning — churn detection (**Insurance only**, 10 min) |
| 11:45 | FASE 4/5: Semantic View (20 min) |
| 12:05 | FASE 5/6: Cortex Agent + Snowflake Intelligence (20 min) |
| 12:25 | FASE FINAL: Streamlit in Snowflake (20 min) |
| 12:45 | Demo and evaluation (30 min) |
| 13:15 | Awards & closing |

---

## Prerequisites

- **Snowflake Trial** (Enterprise, AWS US-West-2) — $400 credits, 30 days
  - Account setup: `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US_WEST_2';`
- **CoCo (Cortex Code)** — Snowflake's AI-powered IDE
- **Warehouse:** `COMPUTE_WH` (already created)
- **Role:** `ACCOUNTADMIN` for the initial setup, then `SYSADMIN` / `PUBLIC`

---

## Repository Structure

```
hackaton/
├── guias/                    # Spanish guides (main)
│   ├── guia_telco.md
│   ├── guia_fsi.md
│   ├── guia_retail.md
│   ├── guia_manufacturing.md
│   ├── guia_healthcare.md
│   ├── guia_media.md
│   ├── guia_logistics.md
│   ├── guia_public_sector.md
│   ├── guia_utilities.md
│   ├── guia_insurance.md    ← special: ML + 5 AI cases
│   ├── en/                   # English guides
│   │   ├── guia_telco.md
│   │   ├── ... (9 standard)
│   │   └── guia_insurance.md
│   └── it/                   # Italian guides
│       ├── guia_telco.md
│       ├── ... (9 standard)
│       └── guia_insurance.md
├── data/
│   ├── telco/               # 4 JSON files per industry
│   ├── fsi/
│   ├── retail/
│   ├── manufacturing/
│   ├── healthcare/
│   ├── media/
│   ├── logistics/
│   ├── public_sector/
│   ├── utilities/
│   └── insurance/           # 4 JSON files including call transcripts
├── Readme.md                 # Spanish README
├── Readme_it.md              # Italian README
├── Readme_en.md              # This file
├── build_guides.py           # Generates Spanish guides
├── build_guides_it.py        # Generates Italian guides
├── build_guides_en.py        # Generates English guides
├── build_guide_insurance.py  # Generates insurance guide (ES + IT + EN)
└── generate_insurance_data.py # Generates insurance synthetic data
```

---

## Evaluation Criteria

| Criterion | Weight |
|-----------|--------|
| Working Bronze + Silver + Gold pipeline | 25% |
| Quality and creativity of AI use cases (and ML for Insurance) | 30% |
| Functional Cortex Agent (answers natural language questions) | 20% |
| Streamlit in Snowflake app quality | 15% |
| Live demo presentation | 10% |

---

## CoCo Usage Tips

CoCo (Cortex Code) is your AI assistant throughout the hackathon. Use it to:

- **Generate SQL** — "Create a dbt model that joins STG_CUSTOMERS with STG_TICKETS"
- **Write dbt models** — "Create models/staging/stg_customers.sql flattening SRC VARIANT"
- **Build Cortex queries** — "Write a SNOWFLAKE.CORTEX.COMPLETE query to classify tickets"
- **Create Semantic Views** — "Create SV_TELCO_COPILOT over my GOLD tables"
- **Create Agents** — "Create a Cortex Agent connected to SV_TELCO_COPILOT"
- **Build Streamlit apps** — "Create a 4-tab SiS app with KPIs, charts and a data table"

> **Pro tip:** Always include the database and schema name in your CoCo prompts so it generates the right objects from the start.

---

## Snowflake Costs (reference)

```sql
-- Credits used in the last 24h
SELECT warehouse_name, SUM(credits_used) AS credits
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD('day', -1, CURRENT_TIMESTAMP())
GROUP BY 1 ORDER BY 2 DESC;

-- Storage used (MB)
SELECT TABLE_SCHEMA, SUM(ACTIVE_BYTES)/1e6 AS mb
FROM information_schema.table_storage_metrics
GROUP BY 1 ORDER BY 2 DESC;
```

---

*Good luck — and may the best copilot win! 🚀*
