# 🏭 Smart Factory Copilot
**DB:** `MANUFACTURING_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called MANUFACTURING_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE MANUFACTURING_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in MANUFACTURING_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/;
```

**[CoCo]** Bronze Tables
```
In MANUFACTURING_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_EQUIPMENT from data/manufacturing/equipment.json
- RAW_SENSOR_READINGS from data/manufacturing/sensor_readings.json
- RAW_WORK_ORDERS from data/manufacturing/work_orders.json
- RAW_PRODUCTION_BATCHES from data/manufacturing/production_batches.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_EQUIPMENT; -- expected: 350
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_SENSOR_READINGS; -- expected: 10,000
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_WORK_ORDERS; -- expected: 750
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_PRODUCTION_BATCHES; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called manufacturing_copilot connected to MANUFACTURING_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_EQUIPMENT, RAW_SENSOR_READINGS, RAW_WORK_ORDERS, RAW_PRODUCTION_BATCHES
```

**[CoCo]** Silver: STG_EQUIPMENT
```
Create models/staging/stg_equipment.sql to flatten RAW_EQUIPMENT.
Fields: equipment/machines (equipment_id, name, type, plant, production_line, manufacturer, status, criticality, hours_operated)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_SENSOR_READINGS
```
Create models/staging/stg_sensor_readings.sql to flatten RAW_SENSOR_READINGS.
Fields: IoT sensor readings (reading_id, equipment_id, sensor_type, value, threshold_min, threshold_max, is_anomaly)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_WORK_ORDERS
```
Create models/staging/stg_work_orders.sql to flatten RAW_WORK_ORDERS.
Fields: work orders (work_order_id, equipment_id, category, priority, description, estimated_hours, spare_parts_cost_eur)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_PRODUCTION_BATCHES
```
Create models/staging/stg_production_batches.sql to flatten RAW_PRODUCTION_BATCHES.
Fields: production batches (batch_id, product_name, production_line, planned_quantity, actual_quantity, defect_count, material_cost_eur, quality_grade)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_EQUIPMENT; -- expected: 350
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_SENSOR_READINGS; -- expected: 10,000
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS; -- expected: 750
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_PRODUCTION_BATCHES; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: EQUIPMENT_360
```
Create models/marts/equipment_360.sql:
360-degree equipment view: base data + total_sensor_anomalies + total_work_orders + open_work_orders + total_maintenance_cost_eur + avg_production_quality + hours_operated + days_since_last_maintenance. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: PRODUCTION_HEALTH
```
Create models/marts/production_health.sql:
Production health by production_line, plant and day: total_batches, total_produced, total_defects, defect_rate_pct, avg_quality_grade, total_material_cost, total_energy_cost.
Materialize as table in GOLD.
```

**[CoCo]** Gold: MAINTENANCE_RISK
```
Create models/marts/maintenance_risk.sql:
Maintenance risk per equipment: hours_operated + anomaly_count_last_90d + pending_work_orders + avg_sensor_deviation + maintenance_risk_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360;
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.GOLD.PRODUCTION_HEALTH;
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Work order classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS.
Analyze the "description" field and return a JSON with: maintenance_type (corrective|preventive|predictive|improvement|inspection|calibration), urgency (critical|high|medium|low), required_specialization (mechanical|electrical|instrumentation|automation|general), production_impact (high|medium|low|none), brief_summary (max 15 words)
System prompt: "You are a work order classification system for an industrial plant."
Create view MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Equipment executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360.
Generate a 3-4 sentence executive summary in English using fields: name, type, plant, production_line, status, criticality, hours_operated, total_sensor_anomalies, total_work_orders, total_maintenance_cost_eur
System prompt: "You are an industrial reliability engineer. Generate a 3-4 sentence executive summary in English about the status of this equipment."
Create view MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Predictive maintenance recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK.
Return a JSON with: failure_risk_level (high|medium|low), recommended_action (replacement|repair|inspection|monitoring), suspect_component, maximum_deadline_days (number), impact_if_failure (line_down|reduced_production|degraded_quality), estimated_cost_eur
System prompt: "You are an industrial predictive maintenance expert. Analyze this equipment and return ONLY valid JSON."
Filter WHERE maintenance_risk_score >= 30.
Create view MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in MANUFACTURING_COPILOT.GOLD called SV_MANUFACTURING_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers industrial equipment, IoT sensor readings, work/maintenance orders and production batches. Industrial plant in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Manufacturing & Industry.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent MANUFACTURING_COPILOT_AGENT connected to Semantic View MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: manufacturing/industrial company in Spain
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT
  COMMENT = 'Copilot for Manufacturing & Industry'
  PROFILE = '{"display_name": "Smart Factory Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for manufacturing/industrial company in Spain. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "manufacturing_analyst",
        "description": "Data covers industrial equipment, IoT sensor readings, work/maintenance orders and production batches. Industrial plant in Spain."
      }
    }],
    "tool_resources": {
      "manufacturing_analyst": {
        "semantic_view": "MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which equipment has the highest failure risk in the next 7 days?"
- "How many critical work orders are pending per plant?"
- "What is the defect rate by production line this week?"
- "Which IoT sensors are generating the most anomalies per unit?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database MANUFACTURING_COPILOT, schema AI, named MANUFACTURING_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from EQUIPMENT_360
2. Distribution — bar chart (st.bar_chart) by category/region from PRODUCTION_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from MAINTENANCE_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
