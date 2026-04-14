# 🏦 FSI Risk & Compliance Copilot
**DB:** `FSI_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called FSI_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE FSI_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in FSI_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY FSI_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/;
```

**[CoCo]** Bronze Tables
```
In FSI_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CLIENTS from data/fsi/clients.json
- RAW_TRANSACTIONS from data/fsi/transactions.json
- RAW_RISK_ALERTS from data/fsi/risk_alerts.json
- RAW_ACCOUNTS from data/fsi/accounts.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_CLIENTS; -- expected: 350
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_TRANSACTIONS; -- expected: 10,000
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_RISK_ALERTS; -- expected: 750
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_ACCOUNTS; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called fsi_copilot connected to FSI_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_CLIENTS, RAW_TRANSACTIONS, RAW_RISK_ALERTS, RAW_ACCOUNTS
```

**[CoCo]** Silver: STG_CLIENTS
```
Create models/staging/stg_clients.sql to flatten RAW_CLIENTS.
Fields: financial clients (client_id, company_name, segment, product_type, monthly_volume_eur, risk_profile, kyc_status, aml_score)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_TRANSACTIONS
```
Create models/staging/stg_transactions.sql to flatten RAW_TRANSACTIONS.
Fields: transactions (transaction_id, client_id, transaction_type, amount_eur, channel, status, country)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_RISK_ALERTS
```
Create models/staging/stg_risk_alerts.sql to flatten RAW_RISK_ALERTS.
Fields: risk/fraud alerts (alert_id, client_id, category, severity, description, rule_triggered, regulatory_report_required)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_ACCOUNTS
```
Create models/staging/stg_accounts.sql to flatten RAW_ACCOUNTS.
Fields: accounts and products (account_id, client_id, account_type, balance_eur, interest_rate_pct, status)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_CLIENTS; -- expected: 350
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_TRANSACTIONS; -- expected: 10,000
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_RISK_ALERTS; -- expected: 750
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_ACCOUNTS; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CLIENT_360
```
Create models/marts/client_360.sql:
360-degree financial client view: base data + total_accounts + total_balance_eur + total_transactions + avg_transaction_amount + total_alerts + open_alerts + kyc_status + aml_score. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: TRANSACTION_HEALTH
```
Create models/marts/transaction_health.sql:
Transaction health by type and day: total_transactions, total_volume_eur, rejected_count, avg_amount, distinct_clients, top_channel.
Materialize as table in GOLD.
```

**[CoCo]** Gold: RISK_EXPOSURE
```
Create models/marts/risk_exposure.sql:
Risk exposure per client: monthly_volume_eur + total_alerts_last_90d + critical_alerts + accounts_in_arrears + kyc_expired + compliance_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.GOLD.CLIENT_360;
SELECT COUNT(*) FROM FSI_COPILOT.GOLD.TRANSACTION_HEALTH;
SELECT COUNT(*) FROM FSI_COPILOT.GOLD.RISK_EXPOSURE;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Automatic risk alert classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table FSI_COPILOT.SILVER.STG_RISK_ALERTS.
Analyze the "description" field and return a JSON with: risk_type (fraud|money_laundering|compliance|credit|operational|cyber), urgency (critical|high|medium|low), department (fraud|compliance|risk|cybersecurity|legal), requires_regulatory_report (yes|no), brief_summary (max 15 words)
System prompt: "You are a risk analyst at a financial institution."
Create view FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Client executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table FSI_COPILOT.GOLD.CLIENT_360.
Generate a 3-4 sentence executive summary in English using fields: company_name, segment, product_type, monthly_volume_eur, total_accounts, total_balance_eur, total_alerts, aml_score, kyc_status
System prompt: "You are a private banking analyst. Generate a 3-4 sentence executive summary in English about this client's financial profile."
Create view FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Compliance action recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table FSI_COPILOT.GOLD.RISK_EXPOSURE.
Return a JSON with: risk_level (high|medium|low), main_action, regulatory_actions (array 2-3), requires_SAR (yes|no), maximum_deadline_days (number), kyc_recommendation
System prompt: "You are a financial compliance expert (KYC/AML). Analyze this customer and return ONLY valid JSON."
Filter WHERE compliance_score >= 40.
Create view FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in FSI_COPILOT.GOLD called SV_FSI_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers financial clients, transactions, risk/fraud/AML alerts and accounts/products. Financial institution in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW FSI_COPILOT.GOLD.SV_FSI_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Financial Services.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent FSI_COPILOT_AGENT connected to Semantic View FSI_COPILOT.GOLD.SV_FSI_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: financial institution in Spain (banking, investments, insurance)
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.FSI_COPILOT_AGENT
  COMMENT = 'Copilot for Financial Services'
  PROFILE = '{"display_name": "FSI Risk & Compliance Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for financial institution in Spain (banking, investments, insurance). Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "fsi_analyst",
        "description": "Data covers financial clients, transactions, risk/fraud/AML alerts and accounts/products. Financial institution in Spain."
      }
    }],
    "tool_resources": {
      "fsi_analyst": {
        "semantic_view": "FSI_COPILOT.GOLD.SV_FSI_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.FSI_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW FSI_COPILOT.GOLD.SV_FSI_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which clients have the lowest compliance score?"
- "How many critical risk alerts are currently open?"
- "What is the total volume of rejected transactions this month?"
- "Which clients have expired or soon-to-expire KYC?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database FSI_COPILOT, schema AI, named FSI_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from CLIENT_360
2. Distribution — bar chart (st.bar_chart) by category/region from TRANSACTION_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from RISK_EXPOSURE
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
