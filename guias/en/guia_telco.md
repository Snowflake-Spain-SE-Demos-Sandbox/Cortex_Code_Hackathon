# 📡 Telco Operations Copilot
**DB:** `TELCO_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called TELCO_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE TELCO_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in TELCO_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/;
```

**[CoCo]** Bronze Tables
```
In TELCO_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS from data/telco/customers.json
- RAW_NETWORK_EVENTS from data/telco/network_events.json
- RAW_TICKETS from data/telco/tickets.json
- RAW_BILLING_USAGE from data/telco/billing_usage.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_CUSTOMERS; -- expected: 350
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_NETWORK_EVENTS; -- expected: 10,000
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_TICKETS; -- expected: 750
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_BILLING_USAGE; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called telco_copilot connected to TELCO_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_CUSTOMERS, RAW_NETWORK_EVENTS, RAW_TICKETS, RAW_BILLING_USAGE
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Create models/staging/stg_customers.sql to flatten RAW_CUSTOMERS.
Fields: telecom customers (customer_id, company_name, plan, region, segment, monthly_revenue_eur, nps_score, status)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_NETWORK_EVENTS
```
Create models/staging/stg_network_events.sql to flatten RAW_NETWORK_EVENTS.
Fields: network events (event_id, customer_id, event_type, severity, cell_tower_id, duration_ms, metadata)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_TICKETS
```
Create models/staging/stg_tickets.sql to flatten RAW_TICKETS.
Fields: support tickets (ticket_id, customer_id, category, priority, description, sla_breached, satisfaction_score)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_BILLING_USAGE
```
Create models/staging/stg_billing_usage.sql to flatten RAW_BILLING_USAGE.
Fields: billing records (billing_id, customer_id, service_type, amount_eur, payment_status)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_CUSTOMERS; -- expected: 350
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_NETWORK_EVENTS; -- expected: 10,000
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_TICKETS; -- expected: 750
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_BILLING_USAGE; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_360
```
Create models/marts/customer_360.sql:
360-degree customer view: base data + total_tickets + open_tickets + avg_satisfaction + sla_breach_rate + total_network_events + critical_events + total_billed_eur. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: NETWORK_HEALTH
```
Create models/marts/network_health.sql:
Network health per cell_tower_id and day: total_events, critical_events, high_events, avg_duration_ms, distinct_customers_affected, top_event_type.
Materialize as table in GOLD.
```

**[CoCo]** Gold: REVENUE_AT_RISK
```
Create models/marts/revenue_at_risk.sql:
Revenue at risk per customer: monthly_revenue_eur + total_tickets_last_90d + sla_breaches_last_90d + unpaid_amount + risk_score (0-100 based on status, SLA, tickets, unpaid bills, NPS).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.GOLD.CUSTOMER_360;
SELECT COUNT(*) FROM TELCO_COPILOT.GOLD.NETWORK_HEALTH;
SELECT COUNT(*) FROM TELCO_COPILOT.GOLD.REVENUE_AT_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Automatic ticket classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table TELCO_COPILOT.SILVER.STG_TICKETS.
Analyze the "description" field and return a JSON with: technical_category (connectivity|hardware|software|coverage|billing|commercial), estimated_severity (critical|high|medium|low), responsible_department (network_engineering|level2_support|billing|commercial|field), brief_summary (max 15 words)
System prompt: "You are a support ticket classification system for a telecommunications company."
Create view TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Account executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table TELCO_COPILOT.GOLD.CUSTOMER_360.
Generate a 3-4 sentence executive summary in English using fields: company_name, plan, region, segment, monthly_revenue_eur, total_tickets, open_tickets, avg_satisfaction, sla_breach_rate, critical_events, total_billed_eur
System prompt: "You are an account analyst at a telecommunications company. Generate a 3-4 sentence executive summary in English about this customer."
Create view TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Retention action recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table TELCO_COPILOT.GOLD.REVENUE_AT_RISK.
Return a JSON with: urgency_level (high|medium|low), main_action, complementary_actions (array 2-3), suggested_offer, contact_channel (phone|email|in_person_visit), personalized_message
System prompt: "You are a customer retention expert in telecommunications. Analyze this at-risk customer and return ONLY valid JSON."
Filter WHERE risk_score >= 30.
Create view TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in TELCO_COPILOT.GOLD called SV_TELCO_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers customers, support tickets, network events (technical incidents) and billing. Regions are Spanish cities.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Telecommunications.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent TELCO_COPILOT_AGENT connected to Semantic View TELCO_COPILOT.GOLD.SV_TELCO_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: telecommunications company in Spain
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT
  COMMENT = 'Copilot for Telecommunications'
  PROFILE = '{"display_name": "Telco Operations Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for telecommunications company in Spain. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "telco_analyst",
        "description": "Data covers customers, support tickets, network events (technical incidents) and billing. Regions are Spanish cities."
      }
    }],
    "tool_resources": {
      "telco_analyst": {
        "semantic_view": "TELCO_COPILOT.GOLD.SV_TELCO_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "What are the 5 customers with the highest churn risk?"
- "Which network towers have the most critical events this week?"
- "How much total revenue is at risk due to unpaid bills?"
- "Give me a summary of tickets with breached SLA still unresolved."

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database TELCO_COPILOT, schema AI, named TELCO_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from CUSTOMER_360
2. Distribution — bar chart (st.bar_chart) by category/region from NETWORK_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from REVENUE_AT_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
