# 🛡️ Insurance Call Center Copilot
**DB:** `INSURANCE_COPILOT`

---

## FASE 0 — Configuracion

**[CoCo]**
```
Crea una base de datos llamada INSURANCE_COPILOT con cinco esquemas: BRONZE, SILVER, GOLD, AI y ML.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE INSURANCE_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integracion Git
```
Crea una API Integration git_https_api llamada GIT_HACKATHON_INTEGRATION
y un Git Repository en INSURANCE_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No requiere autenticacion (repositorio publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY INSURANCE_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @INSURANCE_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/insurance/;
```

**[CoCo]** Tablas Bronze
```
En INSURANCE_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS da data/insurance/customers.json
- RAW_CALLS da data/insurance/call_transcripts.json
- RAW_CLAIMS da data/insurance/claims.json
- RAW_AGENTS da data/insurance/agents.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CUSTOMERS; -- esperado: 350
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CALLS; -- esperado: 500
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CLAIMS; -- esperado: 1,000
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_AGENTS; -- esperado: 50
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Asegurate de estar en **Projects > Workspaces** en Snowsight para ejecutar dbt.

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado insurance_copilot conectado a INSURANCE_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_CUSTOMERS, RAW_CALLS, RAW_CLAIMS, RAW_AGENTS.
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Crea models/staging/stg_customers.sql que haga el flatten de RAW_CUSTOMERS.
Campos: clientes de seguro (customer_id, full_name, age, region, insurance_types, vehicle_brand, vehicle_model, vehicle_year, hogar_type, auto_premium_eur, hogar_premium_eur, total_premium_eur, years_as_customer, num_policies, num_claims_last_2y, payment_status, renewal_date, nps_score, status)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_CALLS
```
Crea models/staging/stg_calls.sql que haga el flatten de RAW_CALLS.
Campos: llamadas al call center (call_id, customer_id, agent_id, call_datetime, duration_seconds, insurance_type, call_type, routing_department, transcript TEXTO COMPLETO, channel, fcr, csat_score, churn_signal_detected, sentiment_label, topics ARRAY)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_CLAIMS
```
Crea models/staging/stg_claims.sql que haga el flatten de RAW_CLAIMS.
Campos: siniestros (claim_id, customer_id, insurance_type, claim_type, claim_date, amount_requested_eur, amount_approved_eur, franchise_eur, status, resolution_days, perito_assigned, involves_third_party)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_AGENTS
```
Crea models/staging/stg_agents.sql que haga el flatten de RAW_AGENTS.
Campos: agentes del call center (agent_id, full_name, team, experience_years, calls_per_day_avg, fcr_rate, avg_handle_time_seconds, qa_score_avg, active)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CUSTOMERS; -- esperado: 350
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CALLS; -- esperado: 500
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CLAIMS; -- esperado: 1,000
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_AGENTS; -- esperado: 50
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_RISK_360
```
Crea models/marts/customer_risk_360.sql:
Vista 360 del cliente: datos base + total_calls + total_complaints + num_churn_calls + total_claims_eur + avg_csat_score + days_since_last_call + days_to_renewal + fcr_rate. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: CALL_INTELLIGENCE_360
```
Crea models/marts/call_intelligence_360.sql:
Vista por llamada con cliente + agente: call_id, customer_id, agent_id, call_type, insurance_type, duration_seconds, routing_department, sentiment_label, fcr, csat_score, churn_signal_detected, customer_tenure, customer_premium, agent_qa_score.
Materializa como table en GOLD.
```

**[CoCo]** Gold: AGENT_PERFORMANCE
```
Crea models/marts/agent_performance.sql:
Performance por agente: agent_id, full_name, team, total_calls, avg_duration_seconds, fcr_rate, avg_csat, cancellation_calls, complaint_calls, qa_score_avg, experience_years.
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
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

**[CoCo]** Caso 1: Resumen automatico de llamadas
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla INSURANCE_COPILOT.SILVER.STG_CALLS.
Analiza el campo "transcript" y devuelve un JSON con: resumen (3-4 frases), puntos_clave (array 2-3), resultado_llamada (resuelto|pendiente|escalado|sin_resolucion), proxima_accion
Prompt de sistema: "Eres un analista de calidad de un call center de seguros. Resume la siguiente transcripcion de llamada."
Crea la vista INSURANCE_COPILOT.AI.VW_CALL_SUMMARY. Limita a 500 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CALL_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 2: Clasificacion y routing inteligente
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla INSURANCE_COPILOT.SILVER.STG_CALLS.
Analiza el campo "transcript" y devuelve un JSON con: categoria_llamada (siniestro|consulta|cancelacion|renovacion|queja|asistencia|cambio_datos|facturacion), subcategoria, departamento_optimo (siniestros_auto|siniestros_hogar|retencion|comercial|atencion_cliente|asistencia_24h|backoffice|facturacion|calidad), urgencia (critica|alta|media|baja), requiere_escalado (si|no), motivo_routing
Prompt de sistema: "Eres un sistema de clasificacion de llamadas de un call center de seguros. Clasifica la siguiente llamada y determina el routing optimo."
Crea la vista INSURANCE_COPILOT.AI.VW_CALL_ROUTING. Limita a 500 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CALL_ROUTING LIMIT 3;
```

**[CoCo]** Caso 3: QA automatico de agentes
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla INSURANCE_COPILOT.SILVER.STG_CALLS.
Analiza el campo "transcript" y devuelve un JSON con: score_empatia (0-10), score_resolucion (0-10), score_protocolo (0-10), score_qa_total (0-100), fortalezas (array 2), areas_mejora (array 2), cumple_script (si|no), tono_general (profesional|neutro|mejorable|inadecuado)
Prompt de sistema: "Eres un evaluador de calidad (QA) experto en call centers de seguros. Evalua el desempeno del agente en la siguiente llamada segun empatia, resolucion del problema, cumplimiento del protocolo y tono profesional."
Crea la vista INSURANCE_COPILOT.AI.VW_AGENT_QA. Limita a 500 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_AGENT_QA LIMIT 3;
```

**[CoCo]** Caso 4: Insights de CX: sentimiento y temas
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla INSURANCE_COPILOT.SILVER.STG_CALLS.
Analiza el campo "transcript" y devuelve un JSON con: sentimiento_cliente (muy_positivo|positivo|neutro|negativo|muy_negativo), score_satisfaccion (0-10), temas_principales (array 3), mencion_competidor (si|no), nombre_competidor, palabras_clave (array 5), nivel_frustracion (0-10)
Prompt de sistema: "Eres un analista de experiencia del cliente (CX) especializado en seguros. Analiza el sentimiento, temas y nivel de satisfaccion del cliente en la siguiente llamada."
Crea la vista INSURANCE_COPILOT.AI.VW_CX_INSIGHTS. Limita a 500 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CX_INSIGHTS LIMIT 3;
```

**[CoCo]** Caso 5: Deteccion de senales de churn
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla INSURANCE_COPILOT.SILVER.STG_CALLS.
Analiza el campo "transcript" y devuelve un JSON con: riesgo_churn (critico|alto|medio|bajo), intencion_cancelacion (si|no), senales_detectadas (array con senales especificas encontradas), mencion_precio (si|no), mencion_competidor (si|no), nivel_insatisfaccion (0-10), accion_recomendada (llamada_retencion|oferta_descuento|escalado_ejecutivo|seguimiento_estandar|sin_accion)
Prompt de sistema: "Eres un especialista en retencion de clientes de una compania de seguros. Detecta senales de churn en la siguiente transcripcion de llamada."
Crea la vista INSURANCE_COPILOT.AI.VW_CHURN_SIGNALS. Limita a 500 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CHURN_SIGNALS LIMIT 3;
```

---

## FASE 4 — Machine Learning: Deteccion de Churn

**[CoCo]**
```
Crea una vista INSURANCE_COPILOT.ML.CHURN_FEATURES que combine GOLD.CUSTOMER_RISK_360
con las senales de churn de las vistas AI (VW_CX_INSIGHTS + VW_CHURN_SIGNALS).
Campos de entrada del modelo: age, years_as_customer, num_policies, total_premium_eur,
  num_claims_last_2y, total_calls, num_complaint_calls, num_churn_signal_calls,
  avg_csat_score, payment_status_encoded, days_to_renewal, nps_score.
Campo objetivo (label): will_churn BOOLEAN (derivado de status = 'cancelado'
  o churn_signal_detected en la ultima llamada).
Luego entrena un modelo de clasificacion de churn usando SNOWFLAKE.ML.CLASSIFICATION.
```

**[SQL]**
```sql
-- 1. Crear el schema ML y la vista de features
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

-- 4. Predicciones sobre todos los clientes activos
CREATE OR REPLACE TABLE INSURANCE_COPILOT.ML.CHURN_PREDICTIONS AS
SELECT
    customer_id,
    CHURN_MODEL!PREDICT(OBJECT_CONSTRUCT(*)) AS prediction
FROM INSURANCE_COPILOT.ML.CHURN_FEATURES;

-- 5. Evaluacion del modelo
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_EVALUATION_METRICS();
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_CONFUSION_MATRIX();
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_FEATURE_IMPORTANCE();

-- 6. Clientes con mayor riesgo de churn
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
-- Verificar predicciones
SELECT COUNT(*), AVG(prediction:probability:True::FLOAT) AS avg_churn_prob
FROM INSURANCE_COPILOT.ML.CHURN_PREDICTIONS
WHERE prediction:class::BOOLEAN = TRUE;
```

---

## FASE 5 — Semantic View

**[CoCo]**
```
Crea una Semantic View en INSURANCE_COPILOT.GOLD llamada SV_INSURANCE_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto de negocio: compania de seguros de auto y hogar en Espana (Seguros Meridian).
Incluye dimensiones con comentarios en espanol (tipo_seguro, tipo_llamada, region,
  tipo_siniestro, sentimiento, departamento_routing), metricas con sinonimos
(prima_total, duracion_llamada, score_churn, tasa_resolucion_primer_contacto,
  importe_siniestro) e instrucciones AI_SQL_GENERATION con contexto del call center.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT;
```

---

## FASE 6 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent en Snowflake Intelligence para Seguros Meridian.
- Crea el objeto Snowflake Intelligence predeterminado si no existe
- Crea la base de datos SNOWFLAKE_INTELLIGENCE y el schema AGENTS
- Crea el agente INSURANCE_COPILOT_AGENT conectado a la Semantic View INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT
  con modelo de orquestacion claude-4-sonnet, respondiendo siempre en espanol
  y contexto: asistente de analisis de call center para seguros de auto y hogar.
- Concede acceso al rol PUBLIC al agente y a la Semantic View
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
      "orchestration": "Eres un asistente de analisis de call center de seguros. Responde siempre en español.",
      "response": "Responde de forma clara y concisa. Usa tablas cuando sea apropiado."
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

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.

---

## Prueba el Agente

Preguntas de ejemplo para validar el agente:

- "¿Cuales son los 10 clientes con mayor riesgo de churn segun el modelo ML?"
- "¿Que tipo de llamadas tienen peor sentimiento del cliente esta semana?"
- "¿Cuales son los agentes con peor puntuacion de QA en el ultimo mes?"
- "¿Cuanto revenue anual esta en riesgo por clientes con signal de cancelacion?"

---

## FASE FINAL — App Streamlit en Snowflake

**[CoCo]**
```
Crea una app Streamlit in Snowflake (SiS) en INSURANCE_COPILOT, schema AI, llamada INSURANCE_APP.
La app se conecta con get_active_session() y tiene 4 tabs:
1. KPIs — st.metric: total llamadas, duracion media, tasa FCR, NPS promedio, revenue en riesgo
2. Llamadas — st.bar_chart por tipo de llamada y sentimiento. Tabla filtrable con transcripciones.
3. Churn Risk — st.dataframe Top 20 clientes por churn_probability (del modelo ML). Color rojo si > 0.7.
4. Performance Agentes — tabla con QA score, FCR y CSAT por agente. Bar chart comparativo.
Usa titulos descriptivos en espanol y aplica st.set_page_config con tema oscuro.
```
