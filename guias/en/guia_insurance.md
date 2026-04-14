# 🛡️ Insurance Call Center Copilot
**DB:** `INSURANCE_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called INSURANCE_COPILOT with five schemas: BRONZE, SILVER, GOLD, AI and ML.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE INSURANCE_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in INSURANCE_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY INSURANCE_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @INSURANCE_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/insurance/;
```

**[CoCo]** Bronze Tables
```
In INSURANCE_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS da data/insurance/customers.json
- RAW_CALLS da data/insurance/call_transcripts.json
- RAW_CLAIMS da data/insurance/claims.json
- RAW_AGENTS da data/insurance/agents.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CUSTOMERS; -- expected: 350
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CALLS; -- expected: 500
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CLAIMS; -- expected: 1,000
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_AGENTS; -- expected: 50
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called insurance_copilot connected to INSURANCE_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_CUSTOMERS, RAW_CALLS, RAW_CLAIMS, RAW_AGENTS.
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Create models/staging/stg_customers.sql to flatten RAW_CUSTOMERS.
Fields: clientes de seguro (customer_id, full_name, age, region, insurance_types, vehicle_brand, vehicle_model, vehicle_year, hogar_type, auto_premium_eur, hogar_premium_eur, total_premium_eur, years_as_customer, num_policies, num_claims_last_2y, payment_status, renewal_date, nps_score, status)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_CALLS
```
Create models/staging/stg_calls.sql to flatten RAW_CALLS.
Fields: llamadas al call center (call_id, customer_id, agent_id, call_datetime, duration_seconds, insurance_type, call_type, routing_department, transcript TEXTO COMPLETO, channel, fcr, csat_score, churn_signal_detected, sentiment_label, topics ARRAY)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_CLAIMS
```
Create models/staging/stg_claims.sql to flatten RAW_CLAIMS.
Fields: siniestros (claim_id, customer_id, insurance_type, claim_type, claim_date, amount_requested_eur, amount_approved_eur, franchise_eur, status, resolution_days, perito_assigned, involves_third_party)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_AGENTS
```
Create models/staging/stg_agents.sql to flatten RAW_AGENTS.
Fields: agentes del call center (agent_id, full_name, team, experience_years, calls_per_day_avg, fcr_rate, avg_handle_time_seconds, qa_score_avg, active)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CUSTOMERS; -- expected: 350
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CALLS; -- expected: 500
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CLAIMS; -- expected: 1,000
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_AGENTS; -- expected: 50
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_RISK_360
```
Create models/marts/customer_risk_360.sql:
Vista 360 del cliente: datos base + total_calls + total_complaints + num_churn_calls + total_claims_eur + avg_csat_score + days_since_last_call + days_to_renewal + fcr_rate. JOIN de todas las Silver.
Materialize as table in GOLD.
```

**[CoCo]** Gold: CALL_INTELLIGENCE_360
```
Create models/marts/call_intelligence_360.sql:
Vista por llamada con cliente + agente: call_id, customer_id, agent_id, call_type, insurance_type, duration_seconds, routing_department, sentiment_label, fcr, csat_score, churn_signal_detected, customer_tenure, customer_premium, agent_qa_score.
Materialize as table in GOLD.
```

**[CoCo]** Gold: AGENT_PERFORMANCE
```
Create models/marts/agent_performance.sql:
Performance por agente: agent_id, full_name, team, total_calls, avg_duration_seconds, fcr_rate, avg_csat, cancellation_calls, complaint_calls, qa_score_avg, experience_years.
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.GOLD.CUSTOMER_RISK_360;
SELECT COUNT(*) FROM INSURANCE_COPILOT.GOLD.CALL_INTELLIGENCE_360;
SELECT COUNT(*) FROM INSURANCE_COPILOT.GOLD.AGENT_PERFORMANCE;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Resumen automatico de llamadas
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table INSURANCE_COPILOT.SILVER.STG_CALLS.
Analyze the field "transcript" and return a JSON with: resumen (3-4 frases), puntos_clave (array 2-3), resultado_llamada (resuelto|pendiente|escalado|sin_resolucion), proxima_accion
System prompt: "Eres un analista de calidad de un call center de seguros. Resume la siguiente transcripcion de llamada."
Create view INSURANCE_COPILOT.AI.VW_CALL_SUMMARY. Limit to 500 records.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CALL_SUMMARY LIMIT 3;
```

**[CoCo]** Case 2: Clasificacion y routing inteligente
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table INSURANCE_COPILOT.SILVER.STG_CALLS.
Analyze the field "transcript" and return a JSON with: categoria_llamada (siniestro|consulta|cancelacion|renovacion|queja|asistencia|cambio_datos|facturacion), subcategoria, departamento_optimo (siniestros_auto|siniestros_hogar|retencion|comercial|atencion_cliente|asistencia_24h|backoffice|facturacion|calidad), urgencia (critica|alta|media|baja), requiere_escalado (si|no), motivo_routing
System prompt: "Eres un sistema de clasificacion de llamadas de un call center de seguros. Clasifica la siguiente llamada y determina el routing optimo."
Create view INSURANCE_COPILOT.AI.VW_CALL_ROUTING. Limit to 500 records.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CALL_ROUTING LIMIT 3;
```

**[CoCo]** Case 3: QA automatico de agentes
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table INSURANCE_COPILOT.SILVER.STG_CALLS.
Analyze the field "transcript" and return a JSON with: score_empatia (0-10), score_resolucion (0-10), score_protocolo (0-10), score_qa_total (0-100), fortalezas (array 2), areas_mejora (array 2), cumple_script (si|no), tono_general (profesional|neutro|mejorable|inadecuado)
System prompt: "Eres un evaluador de calidad (QA) experto en call centers de seguros. Evalua el desempeno del agente en la siguiente llamada segun empatia, resolucion del problema, cumplimiento del protocolo y tono profesional."
Create view INSURANCE_COPILOT.AI.VW_AGENT_QA. Limit to 500 records.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_AGENT_QA LIMIT 3;
```

**[CoCo]** Case 4: Insights de CX: sentimiento y temas
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table INSURANCE_COPILOT.SILVER.STG_CALLS.
Analyze the field "transcript" and return a JSON with: sentimiento_cliente (muy_positivo|positivo|neutro|negativo|muy_negativo), score_satisfaccion (0-10), temas_principales (array 3), mencion_competidor (si|no), nombre_competidor, palabras_clave (array 5), nivel_frustracion (0-10)
System prompt: "Eres un analista de experiencia del cliente (CX) especializado en seguros. Analiza el sentimiento, temas y nivel de satisfaccion del cliente en la siguiente llamada."
Create view INSURANCE_COPILOT.AI.VW_CX_INSIGHTS. Limit to 500 records.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CX_INSIGHTS LIMIT 3;
```

**[CoCo]** Case 5: Deteccion de senales de churn
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table INSURANCE_COPILOT.SILVER.STG_CALLS.
Analyze the field "transcript" and return a JSON with: riesgo_churn (critico|alto|medio|bajo), intencion_cancelacion (si|no), senales_detectadas (array con senales especificas encontradas), mencion_precio (si|no), mencion_competidor (si|no), nivel_insatisfaccion (0-10), accion_recomendada (llamada_retencion|oferta_descuento|escalado_ejecutivo|seguimiento_estandar|sin_accion)
System prompt: "Eres un especialista en retencion de clientes de una compania de seguros. Detecta senales de churn en la siguiente transcripcion de llamada."
Create view INSURANCE_COPILOT.AI.VW_CHURN_SIGNALS. Limit to 500 records.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CHURN_SIGNALS LIMIT 3;
```

---

## FASE 4 — Machine Learning: Churn Detection

**[CoCo]**
```
Create a view INSURANCE_COPILOT.ML.CHURN_FEATURES combining GOLD.CUSTOMER_RISK_360
with churn signals from the AI views (VW_CX_INSIGHTS + VW_CHURN_SIGNALS).
Model input fields: age, years_as_customer, num_policies, total_premium_eur,
  num_claims_last_2y, total_calls, num_complaint_calls, num_churn_signal_calls,
  avg_csat_score, payment_status_encoded, days_to_renewal, nps_score.
Target field (label): will_churn BOOLEAN (derived from status = 'cancelado'
  or churn_signal_detected in the last call).
Then train a churn classification model using SNOWFLAKE.ML.CLASSIFICATION.
```

**[SQL]**
```sql
-- 1. Create ML schema and features view
CREATE SCHEMA IF NOT EXISTS INSURANCE_COPILOT.ML;

CREATE OR REPLACE VIEW INSURANCE_COPILOT.ML.CHURN_FEATURES AS
SELECT
    c.customer_id,
    c.age,
    c.years_as_customer,
    c.num_policies,
    c.total_premium_eur,
    c.num_claims_last_2y,
    r.total_calls,
    r.num_complaint_calls,
    r.num_churn_calls         AS num_churn_signal_calls,
    r.avg_csat_score,
    CASE c.payment_status
        WHEN 'al_dia'      THEN 0
        WHEN 'demora_leve' THEN 1
        WHEN 'impago'      THEN 2 END  AS payment_status_encoded,
    DATEDIFF('day', CURRENT_DATE, c.renewal_date::DATE) AS days_to_renewal,
    COALESCE(c.nps_score, 5)           AS nps_score,
    (c.status = 'cancelado'
     OR r.num_churn_calls > 0)::BOOLEAN AS will_churn
FROM INSURANCE_COPILOT.SILVER.STG_CUSTOMERS c
LEFT JOIN INSURANCE_COPILOT.GOLD.CUSTOMER_RISK_360 r USING (customer_id);

-- 2. Dividir en train (80%) y test (20%)
CREATE OR REPLACE VIEW INSURANCE_COPILOT.ML.CHURN_FEATURES_TRAIN AS
  SELECT * FROM INSURANCE_COPILOT.ML.CHURN_FEATURES
  WHERE MOD(ABS(HASH(customer_id)), 10) < 8;

CREATE OR REPLACE VIEW INSURANCE_COPILOT.ML.CHURN_FEATURES_TEST AS
  SELECT * FROM INSURANCE_COPILOT.ML.CHURN_FEATURES
  WHERE MOD(ABS(HASH(customer_id)), 10) >= 8;

-- 3. Entrenar el modelo de clasificacion
CREATE OR REPLACE SNOWFLAKE.ML.CLASSIFICATION INSURANCE_COPILOT.ML.CHURN_MODEL(
  INPUT_DATA  => SYSTEM$REFERENCE('VIEW', 'INSURANCE_COPILOT.ML.CHURN_FEATURES_TRAIN'),
  TARGET_COLNAME => 'WILL_CHURN'
);

-- 4. Predictions for all active customers
CREATE OR REPLACE TABLE INSURANCE_COPILOT.ML.CHURN_PREDICTIONS AS
SELECT
    customer_id,
    CHURN_MODEL!PREDICT(OBJECT_CONSTRUCT(*)) AS prediction
FROM INSURANCE_COPILOT.ML.CHURN_FEATURES;

-- 5. Model evaluation
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_EVALUATION_METRICS();
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_CONFUSION_MATRIX();
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_FEATURE_IMPORTANCE();

-- 6. Customers with highest churn risk
SELECT
    p.customer_id,
    c.full_name,
    c.total_premium_eur,
    p.prediction:class::BOOLEAN        AS predicted_churn,
    p.prediction:probability:True::FLOAT AS churn_probability
FROM INSURANCE_COPILOT.ML.CHURN_PREDICTIONS p
JOIN INSURANCE_COPILOT.SILVER.STG_CUSTOMERS c USING (customer_id)
ORDER BY churn_probability DESC
LIMIT 20;
```

**[✓ SQL]**
```sql
-- Verify predictions
SELECT COUNT(*), AVG(prediction:probability:True::FLOAT) AS avg_churn_prob
FROM INSURANCE_COPILOT.ML.CHURN_PREDICTIONS
WHERE prediction:class::BOOLEAN = TRUE;
```

---

## FASE 5 — Semantic View

**[CoCo]**
```
Create a Semantic View in INSURANCE_COPILOT.GOLD called SV_INSURANCE_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: auto and home insurance company in Spain (Seguros Meridian).
Include dimensions with English comments (insurance_type, call_type, region,
  claim_type, sentiment, routing_department), metrics with synonyms
(total_premium, call_duration, churn_score, fcr_rate, claim_amount)
and AI_SQL_GENERATION instructions with call center context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT;
```

---

## FASE 6 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Seguros Meridian.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent INSURANCE_COPILOT_AGENT connected to Semantic View INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and context: call center analytics assistant for auto and home insurance.
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.INSURANCE_COPILOT_AGENT
  COMMENT = 'Insurance Call Center Copilot'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are a call center analytics assistant for insurance. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "insurance_analyst",
        "description": "Los datos cubren clientes de seguros (auto y hogar), transcripciones de llamadas al call center con sentimiento y topics, siniestros y performance de agentes. Compania aseguradora Seguros Meridian en Espana."
      }
    }],
    "tool_resources": {
      "insurance_analyst": {
        "semantic_view": "INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.INSURANCE_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "What are the 10 customers with the highest churn risk according to the ML model?"
- "Which call types have the worst customer sentiment this week?"
- "Which agents have the worst QA scores in the last month?"
- "How much annual revenue is at risk from customers with a cancellation signal?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in INSURANCE_COPILOT, schema AI, named INSURANCE_APP.
The app connects using get_active_session() and has 4 tabs:
1. KPIs — st.metric: total calls, avg duration, FCR rate, avg NPS, revenue at risk
2. Calls — st.bar_chart by call type and sentiment. Filterable table with transcripts.
3. Churn Risk — st.dataframe Top 20 customers by churn_probability (from ML model). Red if > 0.7.
4. Agent Performance — table with QA score, FCR and CSAT per agent. Comparative bar chart.
Use descriptive English titles and apply st.set_page_config with dark theme.
```
