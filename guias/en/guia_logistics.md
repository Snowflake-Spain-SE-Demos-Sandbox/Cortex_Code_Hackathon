# 🚚 Logistics & Fleet Copilot
**DB:** `LOGISTICS_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called LOGISTICS_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE LOGISTICS_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in LOGISTICS_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/;
```

**[CoCo]** Bronze Tables
```
In LOGISTICS_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_FLEET from data/logistics/fleet_vehicles.json
- RAW_ROUTE_EVENTS from data/logistics/route_events.json
- RAW_DELIVERY_TICKETS from data/logistics/delivery_tickets.json
- RAW_SHIPMENTS from data/logistics/shipments.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_FLEET; -- expected: 350
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_ROUTE_EVENTS; -- expected: 10,000
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_DELIVERY_TICKETS; -- expected: 750
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_SHIPMENTS; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called logistics_copilot connected to LOGISTICS_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_FLEET, RAW_ROUTE_EVENTS, RAW_DELIVERY_TICKETS, RAW_SHIPMENTS
```

**[CoCo]** Silver: STG_FLEET
```
Create models/staging/stg_fleet.sql to flatten RAW_FLEET.
Fields: fleet vehicles (vehicle_id, plate_number, type, brand, base_location, max_payload_kg, fuel_type, km_total, status)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_ROUTE_EVENTS
```
Create models/staging/stg_route_events.sql to flatten RAW_ROUTE_EVENTS.
Fields: route events (event_id, vehicle_id, shipment_id, event_type, location, gps_lat, gps_lon, speed_kmh)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_DELIVERY_TICKETS
```
Create models/staging/stg_delivery_tickets.sql to flatten RAW_DELIVERY_TICKETS.
Fields: incident tickets (ticket_id, shipment_id, vehicle_id, category, priority, description, sla_breached, estimated_cost_eur)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_SHIPMENTS
```
Create models/staging/stg_shipments.sql to flatten RAW_SHIPMENTS.
Fields: shipments (shipment_id, client_name, origin, destination, service_type, amount_eur, weight_kg, status)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_FLEET; -- expected: 350
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_ROUTE_EVENTS; -- expected: 10,000
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_DELIVERY_TICKETS; -- expected: 750
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_SHIPMENTS; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: VEHICLE_360
```
Create models/marts/vehicle_360.sql:
360-degree vehicle view: base data + total_shipments + total_route_events + incidents_count + total_km_last_90d + total_revenue_eur + avg_delivery_time_hours + days_until_maintenance. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: ROUTE_HEALTH
```
Create models/marts/route_health.sql:
Route health by origin, destination and day: total_shipments, total_events, delay_events, avg_delivery_hours, on_time_rate, total_revenue_eur, distinct_vehicles.
Materialize as table in GOLD.
```

**[CoCo]** Gold: DELIVERY_RISK
```
Create models/marts/delivery_risk.sql:
Delivery risk per vehicle/route: pending_tickets + sla_breaches_last_90d + delay_count + damage_claims + km_since_maintenance + delivery_risk_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.GOLD.VEHICLE_360;
SELECT COUNT(*) FROM LOGISTICS_COPILOT.GOLD.ROUTE_HEALTH;
SELECT COUNT(*) FROM LOGISTICS_COPILOT.GOLD.DELIVERY_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Delivery incident classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table LOGISTICS_COPILOT.SILVER.STG_DELIVERY_TICKETS.
Analyze the "description" field and return a JSON with: incident_type (delay|damage|address|vehicle|documentation|refusal|temperature|weight), urgency (critical|high|medium|low), department (operations|fleet|quality|customer_service|legal), service_impact (high|medium|low), brief_summary (max 15 words)
System prompt: "You are an incident classification system for a transport and logistics company."
Create view LOGISTICS_COPILOT.AI.VW_DELIVERY_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_DELIVERY_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Vehicle/fleet executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table LOGISTICS_COPILOT.GOLD.VEHICLE_360.
Generate a 3-4 sentence executive summary in English using fields: plate_number, type, base_location, km_total, total_shipments, incidents_count, total_revenue_eur, days_until_maintenance
System prompt: "You are a transport fleet manager. Generate a 3-4 sentence executive summary in English about the status of this vehicle."
Create view LOGISTICS_COPILOT.AI.VW_FLEET_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_FLEET_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Route optimization recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table LOGISTICS_COPILOT.GOLD.DELIVERY_RISK.
Return a JSON with: risk_level (high|medium|low), main_action (route_change|vehicle_maintenance|capacity_reinforcement|driver_training), estimated_improvement_pct, implementation_deadline_days, cost_impact_eur, detailed_recommendation
System prompt: "You are a logistics optimization expert. Analyze this route/vehicle and return ONLY valid JSON."
Filter WHERE delivery_risk_score >= 30.
Create view LOGISTICS_COPILOT.AI.VW_ROUTE_OPTIMIZATION. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_ROUTE_OPTIMIZATION LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in LOGISTICS_COPILOT.GOLD called SV_LOGISTICS_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers fleet vehicles, route events, delivery incident tickets and shipments. Transport/logistics company in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Transport, Logistics & Travel.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent LOGISTICS_COPILOT_AGENT connected to Semantic View LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: transport and logistics company in Spain
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.LOGISTICS_COPILOT_AGENT
  COMMENT = 'Copilot for Transport, Logistics & Travel'
  PROFILE = '{"display_name": "Logistics & Fleet Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for transport and logistics company in Spain. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "logistics_analyst",
        "description": "Data covers fleet vehicles, route events, delivery incident tickets and shipments. Transport/logistics company in Spain."
      }
    }],
    "tool_resources": {
      "logistics_analyst": {
        "semantic_view": "LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.LOGISTICS_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which fleet vehicles need urgent maintenance?"
- "Which routes have the highest incident and delay rate?"
- "How many shipments have breached SLA this week?"
- "What is the estimated cost of open incidents by zone?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database LOGISTICS_COPILOT, schema AI, named LOGISTICS_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from VEHICLE_360
2. Distribution — bar chart (st.bar_chart) by category/region from ROUTE_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from DELIVERY_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
