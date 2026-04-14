# 🏥 Clinical Intelligence Copilot
**DB:** `HEALTHCARE_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called HEALTHCARE_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE HEALTHCARE_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in HEALTHCARE_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY HEALTHCARE_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @HEALTHCARE_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/healthcare/;
```

**[CoCo]** Bronze Tables
```
In HEALTHCARE_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_PATIENTS from data/healthcare/patients.json
- RAW_ENCOUNTERS from data/healthcare/encounters.json
- RAW_CLINICAL_NOTES from data/healthcare/clinical_notes.json
- RAW_CLAIMS from data/healthcare/claims.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_PATIENTS; -- expected: 350
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_ENCOUNTERS; -- expected: 10,000
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_CLINICAL_NOTES; -- expected: 750
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_CLAIMS; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called healthcare_copilot connected to HEALTHCARE_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_PATIENTS, RAW_ENCOUNTERS, RAW_CLINICAL_NOTES, RAW_CLAIMS
```

**[CoCo]** Silver: STG_PATIENTS
```
Create models/staging/stg_patients.sql to flatten RAW_PATIENTS.
Fields: patients (patient_id, full_name, age, gender, region, insurance_type, chronic_conditions, department, risk_level)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_ENCOUNTERS
```
Create models/staging/stg_encounters.sql to flatten RAW_ENCOUNTERS.
Fields: clinical encounters (encounter_id, patient_id, encounter_type, department, diagnosis_code, duration_minutes)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_CLINICAL_NOTES
```
Create models/staging/stg_clinical_notes.sql to flatten RAW_CLINICAL_NOTES.
Fields: clinical notes (note_id, patient_id, category, priority, description, requires_followup)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_CLAIMS
```
Create models/staging/stg_claims.sql to flatten RAW_CLAIMS.
Fields: insurance claims/billing (claim_id, patient_id, claim_type, amount_eur, insurance_response, status)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_PATIENTS; -- expected: 350
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_ENCOUNTERS; -- expected: 10,000
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_CLINICAL_NOTES; -- expected: 750
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_CLAIMS; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: PATIENT_360
```
Create models/marts/patient_360.sql:
360-degree patient view: base data + total_encounters + emergency_visits + total_claims_eur + pending_claims + active_conditions + followup_notes + last_encounter_date. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: CLINICAL_HEALTH
```
Create models/marts/clinical_health.sql:
Clinical health by department and day: total_encounters, avg_duration_minutes, emergency_rate, readmission_count, distinct_patients, top_diagnosis_code.
Materialize as table in GOLD.
```

**[CoCo]** Gold: READMISSION_RISK
```
Create models/marts/readmission_risk.sql:
Readmission risk per patient: age + chronic_conditions + emergency_visits_last_90d + pending_followups + claim_rejections + readmission_risk_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.GOLD.PATIENT_360;
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.GOLD.CLINICAL_HEALTH;
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.GOLD.READMISSION_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Clinical note classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table HEALTHCARE_COPILOT.SILVER.STG_CLINICAL_NOTES.
Analyze the "description" field and return a JSON with: main_specialty (cardiology|traumatology|oncology|internal_medicine|emergency|neurology|pneumology), urgency (critical|high|medium|low), care_type (diagnosis|follow_up|interconsultation|discharge|emergency), requires_follow_up (yes|no), clinical_summary (max 15 words)
System prompt: "You are a clinical classification system for a hospital."
Create view HEALTHCARE_COPILOT.AI.VW_CLINICAL_NOTE_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_CLINICAL_NOTE_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Patient executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table HEALTHCARE_COPILOT.GOLD.PATIENT_360.
Generate a 3-4 sentence executive summary in English using fields: full_name, age, gender, chronic_conditions, total_encounters, emergency_visits, active_conditions, total_claims_eur, risk_level
System prompt: "You are an internal medicine physician. Generate a 3-4 sentence clinical executive summary in English about this patient."
Create view HEALTHCARE_COPILOT.AI.VW_PATIENT_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_PATIENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Clinical follow-up recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table HEALTHCARE_COPILOT.GOLD.READMISSION_RISK.
Return a JSON with: readmission_risk_level (high|medium|low), main_action (urgent_appointment|teleconsult|home_hospitalization|monitoring), referral_specialty, deadline_days (number), follow_up_plan (array 2-3 actions), message_for_doctor
System prompt: "You are a clinical management specialist. Analyze this patient and return ONLY valid JSON."
Filter WHERE readmission_risk_score >= 30.
Create view HEALTHCARE_COPILOT.AI.VW_FOLLOWUP_RECOMMENDATIONS. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_FOLLOWUP_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in HEALTHCARE_COPILOT.GOLD called SV_HEALTHCARE_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers patients, clinical encounters, clinical notes and insurance claims. Hospital/clinic in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Healthcare & Life Sciences.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent HEALTHCARE_COPILOT_AGENT connected to Semantic View HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: healthcare organization in Spain (hospital/clinic)
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.HEALTHCARE_COPILOT_AGENT
  COMMENT = 'Copilot for Healthcare & Life Sciences'
  PROFILE = '{"display_name": "Clinical Intelligence Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for healthcare organization in Spain (hospital/clinic). Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "healthcare_analyst",
        "description": "Data covers patients, clinical encounters, clinical notes and insurance claims. Hospital/clinic in Spain."
      }
    }],
    "tool_resources": {
      "healthcare_analyst": {
        "semantic_view": "HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.HEALTHCARE_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which patients have the highest risk of urgent readmission?"
- "Which departments have the highest emergency rate?"
- "How many clinical notes require immediate follow-up?"
- "What is the total amount of pending insurance claims?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database HEALTHCARE_COPILOT, schema AI, named HEALTHCARE_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from PATIENT_360
2. Distribution — bar chart (st.bar_chart) by category/region from CLINICAL_HEALTH
3. Top 20 — interactive table (st.dataframe) sorted by risk score from READMISSION_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
