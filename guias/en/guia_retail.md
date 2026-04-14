# 🛍️ Retail Customer Intelligence Copilot
**DB:** `RETAIL_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called RETAIL_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE RETAIL_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in RETAIL_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY RETAIL_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @RETAIL_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/retail/;
```

**[CoCo]** Bronze Tables
```
In RETAIL_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS from data/retail/customers.json
- RAW_ORDERS from data/retail/orders.json
- RAW_SUPPORT_CASES from data/retail/support_cases.json
- RAW_INVENTORY from data/retail/inventory_movements.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_CUSTOMERS; -- expected: 350
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_ORDERS; -- expected: 10,000
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_SUPPORT_CASES; -- expected: 750
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_INVENTORY; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called retail_copilot connected to RETAIL_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_CUSTOMERS, RAW_ORDERS, RAW_SUPPORT_CASES, RAW_INVENTORY
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Create models/staging/stg_customers.sql to flatten RAW_CUSTOMERS.
Fields: customers (customer_id, full_name, region, loyalty_tier, channel_preference, avg_order_value_eur, total_lifetime_value_eur, preferred_category, status)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_ORDERS
```
Create models/staging/stg_orders.sql to flatten RAW_ORDERS.
Fields: orders (order_id, customer_id, order_type, total_amount_eur, items_count, store_id, payment_method, product_category)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_SUPPORT_CASES
```
Create models/staging/stg_support_cases.sql to flatten RAW_SUPPORT_CASES.
Fields: support cases (case_id, customer_id, category, priority, description, sla_breached, satisfaction_score)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_INVENTORY
```
Create models/staging/stg_inventory.sql to flatten RAW_INVENTORY.
Fields: inventory movements (movement_id, product_id, store_id, movement_type, quantity, unit_cost_eur, product_category)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_CUSTOMERS; -- expected: 350
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_ORDERS; -- expected: 10,000
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_SUPPORT_CASES; -- expected: 750
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_INVENTORY; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_360
```
Create models/marts/customer_360.sql:
360-degree retail customer view: base data + total_orders + total_spent_eur + avg_order_value + total_returns + open_cases + avg_satisfaction + loyalty_tier + days_since_last_purchase. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: SALES_PERFORMANCE
```
Create models/marts/sales_performance.sql:
Sales performance by store_id, product_category and day: total_orders, total_revenue_eur, avg_basket_size, return_rate, distinct_customers, top_payment_method.
Materialize as table in GOLD.
```

**[CoCo]** Gold: CHURN_RISK
```
Create models/marts/churn_risk.sql:
Churn risk per customer: total_lifetime_value_eur + days_since_last_purchase + support_cases_last_90d + sla_breaches + return_rate + churn_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.GOLD.CUSTOMER_360;
SELECT COUNT(*) FROM RETAIL_COPILOT.GOLD.SALES_PERFORMANCE;
SELECT COUNT(*) FROM RETAIL_COPILOT.GOLD.CHURN_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Support case classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table RETAIL_COPILOT.SILVER.STG_SUPPORT_CASES.
Analyze the "description" field and return a JSON with: incident_type (return|shipping|product|billing|loyalty|account|technology), urgency (critical|high|medium|low), department (logistics|customer_service|billing|marketing|IT), customer_impact (high|medium|low), brief_summary (max 15 words)
System prompt: "You are an incident classification system for a retail company."
Create view RETAIL_COPILOT.AI.VW_CASE_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_CASE_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Customer executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table RETAIL_COPILOT.GOLD.CUSTOMER_360.
Generate a 3-4 sentence executive summary in English using fields: full_name, loyalty_tier, region, total_orders, total_spent_eur, avg_order_value, preferred_category, days_since_last_purchase, open_cases
System prompt: "You are a CRM analyst at a retail company. Generate a 3-4 sentence executive summary in English about this customer's purchase profile."
Create view RETAIL_COPILOT.AI.VW_CUSTOMER_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_CUSTOMER_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Personalized offer recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table RETAIL_COPILOT.GOLD.CHURN_RISK.
Return a JSON with: behavioral_segment (loyal|at_risk|dormant|new), main_offer (discount or product), contact_channel (email|push|sms|mail), optimal_time (day/hour), personalized_message
System prompt: "You are a retail personalized marketing expert. Analyze this customer and return ONLY valid JSON."
Filter WHERE churn_score >= 30.
Create view RETAIL_COPILOT.AI.VW_PERSONALIZATION_RECOMMENDATIONS. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_PERSONALIZATION_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in RETAIL_COPILOT.GOLD called SV_RETAIL_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers retail customers, orders/purchases, support/return cases and inventory. Retail chain in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Retail & Consumer Goods.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent RETAIL_COPILOT_AGENT connected to Semantic View RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: retail company in Spain
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.RETAIL_COPILOT_AGENT
  COMMENT = 'Copilot for Retail & Consumer Goods'
  PROFILE = '{"display_name": "Retail Customer Intelligence Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for retail company in Spain. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "retail_analyst",
        "description": "Data covers retail customers, orders/purchases, support/return cases and inventory. Retail chain in Spain."
      }
    }],
    "tool_resources": {
      "retail_analyst": {
        "semantic_view": "RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.RETAIL_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "What are the 10 customers with the highest churn risk?"
- "Which product categories have the most returns this month?"
- "What is the average basket size by loyalty tier?"
- "Which customers haven't purchased in more than 90 days?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database RETAIL_COPILOT, schema AI, named RETAIL_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from CUSTOMER_360
2. Distribution — bar chart (st.bar_chart) by category/region from SALES_PERFORMANCE
3. Top 20 — interactive table (st.dataframe) sorted by risk score from CHURN_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
