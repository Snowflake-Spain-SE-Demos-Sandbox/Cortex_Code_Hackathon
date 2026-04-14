# 🌐 Smart City Copilot
**DB:** `PUBLIC_SECTOR_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called PUBLIC_SECTOR_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE PUBLIC_SECTOR_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in PUBLIC_SECTOR_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/;
```

**[CoCo]** Bronze Tables
```
In PUBLIC_SECTOR_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CITIZENS from data/public_sector/citizens.json
- RAW_SERVICE_REQUESTS from data/public_sector/service_requests.json
- RAW_INSPECTIONS from data/public_sector/inspection_reports.json
- RAW_BUDGET from data/public_sector/budget_items.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_CITIZENS; -- expected: 350
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_SERVICE_REQUESTS; -- expected: 10,000
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_INSPECTIONS; -- expected: 750
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_BUDGET; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called public_sector_copilot connected to PUBLIC_SECTOR_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_CITIZENS, RAW_SERVICE_REQUESTS, RAW_INSPECTIONS, RAW_BUDGET
```

**[CoCo]** Silver: STG_CITIZENS
```
Create models/staging/stg_citizens.sql to flatten RAW_CITIZENS.
Fields: citizens (citizen_id, full_name, age, district, municipality, profile_type, preferred_channel, digital_literacy)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_SERVICE_REQUESTS
```
Create models/staging/stg_service_requests.sql to flatten RAW_SERVICE_REQUESTS.
Fields: service requests (request_id, citizen_id, request_type, department, channel, status, priority, response_days)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_INSPECTIONS
```
Create models/staging/stg_inspections.sql to flatten RAW_INSPECTIONS.
Fields: inspection reports (report_id, inspector_id, category, severity, description, fine_amount_eur, follow_up_required)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_BUDGET
```
Create models/staging/stg_budget.sql to flatten RAW_BUDGET.
Fields: budget items (item_id, department, fiscal_year, budget_line, approved_amount_eur, executed_amount_eur, execution_pct)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_CITIZENS; -- expected: 350
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_SERVICE_REQUESTS; -- expected: 10,000
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS; -- expected: 750
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_BUDGET; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CITIZEN_360
```
Create models/marts/citizen_360.sql:
360-degree citizen view: base data + total_requests + resolved_requests + avg_response_days + total_complaints + digital_channel_usage_pct + days_since_last_interaction. JOIN of Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: SERVICE_HEALTH
```
Create models/marts/service_health.sql:
Service health by department and period: total_requests, resolved_count, avg_response_days, overdue_count, citizen_satisfaction_proxy, top_request_type.
Materialize as table in GOLD.
```

**[CoCo]** Gold: BUDGET_EXECUTION
```
Create models/marts/budget_execution.sql:
Budget execution by department and quarter: approved_total_eur, executed_total_eur, execution_pct, pending_eur, overbudget_items, underexecuted_items.
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.GOLD.CITIZEN_360;
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH;
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.GOLD.BUDGET_EXECUTION;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Inspection report classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS.
Analyze the "description" field and return a JSON with: scope (urban_planning|health|environment|work_safety|accessibility|business|public_works|noise), severity (minor|serious|very_serious), responsible_authority (urban_planning|health|environment|local_police|fire_dept|court), requires_penalty (yes|no), brief_summary (max 15 words)
System prompt: "You are an inspection classification system for a public administration."
Create view PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Public service executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.
Generate a 3-4 sentence executive summary in English using fields: department, total_requests, resolved_count, avg_response_days, overdue_count, top_request_type
System prompt: "You are a public management analyst. Generate a 3-4 sentence executive summary in English about the status of this service/department."
Create view PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Service improvement recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.
Return a JSON with: efficiency_level (good|improvable|deficient), main_action (digitalization|training|resource_reallocation|process_simplification), improvement_areas (array 2-3), estimated_savings_pct, implementation_deadline_months, citizen_impact (high|medium|low)
System prompt: "You are a public sector digital transformation consultant. Analyze this department and return ONLY valid JSON."
Filter WHERE avg_response_days > 15.
Create view PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in PUBLIC_SECTOR_COPILOT.GOLD called SV_PUBLIC_SECTOR_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers citizens, service requests, inspection reports and budgets. Public administration / city council in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Public Sector.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent PUBLIC_SECTOR_COPILOT_AGENT connected to Semantic View PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: public administration / city council in Spain
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.PUBLIC_SECTOR_COPILOT_AGENT
  COMMENT = 'Copilot for Public Sector'
  PROFILE = '{"display_name": "Smart City Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for public administration / city council in Spain. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "public_sector_analyst",
        "description": "Data covers citizens, service requests, inspection reports and budgets. Public administration / city council in Spain."
      }
    }],
    "tool_resources": {
      "public_sector_analyst": {
        "semantic_view": "PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.PUBLIC_SECTOR_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which departments have the most overdue pending requests?"
- "What is the budget execution percentage by department?"
- "How many serious inspections are unresolved this quarter?"
- "Which services exceed the target response time of 15 days?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database PUBLIC_SECTOR_COPILOT, schema AI, named PUBLIC_SECTOR_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from CITIZEN_360
2. Distribution — bar chart (st.bar_chart) by category/region from SERVICE_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from BUDGET_EXECUTION
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
