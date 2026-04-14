# 💧 Water & Utilities Plant Copilot — Talk to your Plant
**DB:** `UTILITIES_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called UTILITIES_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE UTILITIES_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in UTILITIES_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/;
```

**[CoCo]** Bronze Tables
```
In UTILITIES_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_PLANTS from data/utilities/plants.json
- RAW_SENSOR_EVENTS from data/utilities/sensor_events.json
- RAW_INCIDENTS from data/utilities/incidents.json
- RAW_WATER_QUALITY from data/utilities/water_quality.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_PLANTS; -- expected: 350
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_SENSOR_EVENTS; -- expected: 10,000
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_INCIDENTS; -- expected: 750
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_WATER_QUALITY; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called utilities_copilot connected to UTILITIES_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_PLANTS, RAW_SENSOR_EVENTS, RAW_INCIDENTS, RAW_WATER_QUALITY
```

**[CoCo]** Silver: STG_PLANTS
```
Create models/staging/stg_plants.sql to flatten RAW_PLANTS.
Fields: plants/facilities (plant_id, plant_name, plant_type, operator, region, zone, capacity_m3_day, current_flow_m3_day, status, criticality, annual_budget_eur)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_SENSOR_EVENTS
```
Create models/staging/stg_sensor_events.sql to flatten RAW_SENSOR_EVENTS.
Fields: SCADA sensor events (event_id, plant_id, sensor_type, value, alarm_min, alarm_max, is_alarm, subsystem)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_INCIDENTS
```
Create models/staging/stg_incidents.sql to flatten RAW_INCIDENTS.
Fields: operational incidents (incident_id, plant_id, category, priority, description, affected_population, regulatory_notification_required, repair_cost_eur)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_WATER_QUALITY
```
Create models/staging/stg_water_quality.sql to flatten RAW_WATER_QUALITY.
Fields: water quality samples (sample_id, plant_id, parameter, value, legal_min, legal_max, compliant, sample_point, reported_to_authority)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_PLANTS; -- expected: 350
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_SENSOR_EVENTS; -- expected: 10,000
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_INCIDENTS; -- expected: 750
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_WATER_QUALITY; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: PLANT_360
```
Create models/marts/plant_360.sql:
360-degree plant view: base data + total_sensor_alarms + alarm_rate_pct + total_incidents + open_incidents + avg_repair_cost_eur + quality_non_compliant_count + regulatory_notifications + days_since_last_inspection. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: OPERATIONAL_HEALTH
```
Create models/marts/operational_health.sql:
Operational health per plant and day: total_sensor_events, alarm_count, alarm_rate_pct, top_alarm_sensor, subsystems_affected, avg_flow_m3h, energy_consumption_kWh.
Materialize as table in GOLD.
```

**[CoCo]** Gold: COMPLIANCE_RISK
```
Create models/marts/compliance_risk.sql:
Compliance risk per plant: non_compliant_samples_last_90d + regulatory_notifications + open_critical_incidents + alarm_rate + days_overdue_inspection + compliance_risk_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.GOLD.PLANT_360;
SELECT COUNT(*) FROM UTILITIES_COPILOT.GOLD.OPERATIONAL_HEALTH;
SELECT COUNT(*) FROM UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Operational incident classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table UTILITIES_COPILOT.SILVER.STG_INCIDENTS.
Analyze the "description" field and return a JSON with: incident_type (pump_failure|water_quality|network_leak|chlorination_failure|tank_alarm|pipe_break|scada_failure|contamination), urgency (critical|high|medium|low), responsible_team (operations|quality|maintenance|laboratory|regulatory|emergency), regulatory_notification (yes|no), brief_summary (max 15 words)
System prompt: "You are an incident classification system for a water management company (utilities). Classify the described incident."
Create view UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Plant executive summary (Talk to your Plant)
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table UTILITIES_COPILOT.GOLD.PLANT_360.
Generate a 3-4 sentence executive summary in English using fields: plant_name, plant_type, operator, region, status, criticality, total_sensor_alarms, total_incidents, open_incidents, quality_non_compliant_count, regulatory_notifications
System prompt: "You are a water treatment plant operations director. Generate a 3-4 sentence executive summary in English about the operational status of this facility."
Create view UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Operational action recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK.
Return a JSON with: regulatory_risk_level (high|medium|low), main_action (urgent_inspection|treatment_adjustment|preventive_maintenance|authority_notification|intensive_monitoring), critical_parameters (array 2-3), maximum_deadline_hours (number), population_impact (number_of_people), technical_recommendation
System prompt: "You are a utilities management and water regulatory compliance expert. Analyze this plant and return ONLY valid JSON."
Filter WHERE compliance_risk_score >= 30.
Create view UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in UTILITIES_COPILOT.GOLD called SV_UTILITIES_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers water treatment plants (ETAP/WWTP/pumping), SCADA sensor events, operational incidents and water quality samples. Water management/utilities company in Spain (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA).
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Utilities & Water Management.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent UTILITIES_COPILOT_AGENT connected to Semantic View UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: water management and utilities company in Spain (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA)
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT
  COMMENT = 'Copilot for Utilities & Water Management'
  PROFILE = '{"display_name": "Water & Utilities Plant Copilot — Talk to your Plant"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for water management and utilities company in Spain (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA). Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "utilities_analyst",
        "description": "Data covers water treatment plants (ETAP/WWTP/pumping), SCADA sensor events, operational incidents and water quality samples. Water management/utilities company in Spain (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA)."
      }
    }],
    "tool_resources": {
      "utilities_analyst": {
        "semantic_view": "UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which plants have the highest regulatory non-compliance risk?"
- "How many critical SCADA alarms are currently active?"
- "Which water quality parameters are outside legal limits?"
- "Give me the operational status of all high-criticality plants."

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database UTILITIES_COPILOT, schema AI, named UTILITIES_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from PLANT_360
2. Distribution — bar chart (st.bar_chart) by category/region from OPERATIONAL_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from COMPLIANCE_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
