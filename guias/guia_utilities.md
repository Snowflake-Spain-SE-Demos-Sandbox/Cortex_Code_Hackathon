# 💧 Water & Utilities Plant Copilot — Talk to your Plant
**DB:** `UTILITIES_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada UTILITIES_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE UTILITIES_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en UTILITIES_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/;
```

**[CoCo]** Tablas Bronze
```
En UTILITIES_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_PLANTS desde data/utilities/plants.json
- RAW_SENSOR_EVENTS desde data/utilities/sensor_events.json
- RAW_INCIDENTS desde data/utilities/incidents.json
- RAW_WATER_QUALITY desde data/utilities/water_quality.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_PLANTS; -- esperado: 350
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_SENSOR_EVENTS; -- esperado: 10,000
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_INCIDENTS; -- esperado: 750
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_WATER_QUALITY; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Asegurate de estar en **Projects > Workspaces** en Snowsight para ejecutar dbt.

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado utilities_copilot conectado a UTILITIES_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_PLANTS, RAW_SENSOR_EVENTS, RAW_INCIDENTS, RAW_WATER_QUALITY
```

**[CoCo]** Silver: STG_PLANTS
```
Crea models/staging/stg_plants.sql que haga flatten de RAW_PLANTS.
Campos: plantas/instalaciones (plant_id, plant_name, plant_type, operator, region, zone, capacity_m3_day, current_flow_m3_day, status, criticality, annual_budget_eur)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_SENSOR_EVENTS
```
Crea models/staging/stg_sensor_events.sql que haga flatten de RAW_SENSOR_EVENTS.
Campos: eventos de sensores SCADA (event_id, plant_id, sensor_type, value, alarm_min, alarm_max, is_alarm, subsystem)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_INCIDENTS
```
Crea models/staging/stg_incidents.sql que haga flatten de RAW_INCIDENTS.
Campos: incidencias operacionales (incident_id, plant_id, category, priority, description, affected_population, regulatory_notification_required, repair_cost_eur)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_WATER_QUALITY
```
Crea models/staging/stg_water_quality.sql que haga flatten de RAW_WATER_QUALITY.
Campos: muestras de calidad del agua (sample_id, plant_id, parameter, value, legal_min, legal_max, compliant, sample_point, reported_to_authority)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_PLANTS; -- esperado: 350
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_SENSOR_EVENTS; -- esperado: 10,000
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_INCIDENTS; -- esperado: 750
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_WATER_QUALITY; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: PLANT_360
```
Crea models/marts/plant_360.sql:
Vista 360 de cada planta: datos basicos + total_sensor_alarms + alarm_rate_pct + total_incidents + open_incidents + avg_repair_cost_eur + quality_non_compliant_count + regulatory_notifications + days_since_last_inspection. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: OPERATIONAL_HEALTH
```
Crea models/marts/operational_health.sql:
Salud operacional por planta y dia: total_sensor_events, alarm_count, alarm_rate_pct, top_alarm_sensor, subsystems_affected, avg_flow_m3h, energy_consumption_kWh.
Materializa como table en GOLD.
```

**[CoCo]** Gold: COMPLIANCE_RISK
```
Crea models/marts/compliance_risk.sql:
Riesgo de compliance por planta: non_compliant_samples_last_90d + regulatory_notifications + open_critical_incidents + alarm_rate + days_overdue_inspection + compliance_risk_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
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

**[CoCo]** Caso 1: Clasificacion de incidencias operacionales
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla UTILITIES_COPILOT.SILVER.STG_INCIDENTS.
Analiza el campo "description" y devuelve un JSON con: tipo_incidencia (averia_bomba|calidad_agua|fuga_red|fallo_cloracion|alarma_deposito|rotura_tuberia|fallo_scada|contaminacion), urgencia (critica|alta|media|baja), equipo_responsable (operaciones|calidad|mantenimiento|laboratorio|regulatorio|emergencias), notificacion_regulatoria (si|no), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de incidencias de una empresa de gestion del agua (utilities). Clasifica la incidencia descrita."
Crea la vista UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de planta (Talk to your Plant)
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla UTILITIES_COPILOT.GOLD.PLANT_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: plant_name, plant_type, operator, region, status, criticality, total_sensor_alarms, total_incidents, open_incidents, quality_non_compliant_count, regulatory_notifications
Prompt del sistema: "Eres un director de operaciones de una planta de tratamiento de agua. Genera un resumen ejecutivo en espanol (3-4 frases) del estado operacional de esta instalacion."
Crea la vista UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de accion operacional
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK.
Devuelve un JSON con: nivel_riesgo_regulatorio (alto|medio|bajo), accion_principal (inspeccion_urgente|ajuste_tratamiento|mantenimiento_preventivo|notificacion_autoridad|monitorizacion_intensiva), parametros_criticos (array 2-3), plazo_maximo_horas (numero), impacto_poblacion (numero_personas), recomendacion_tecnica
Prompt del sistema: "Eres un experto en gestion de utilities y compliance regulatorio del agua. Analiza esta planta y devuelve SOLO un JSON valido."
Filtra WHERE compliance_risk_score >= 30.
Crea la vista UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en UTILITIES_COPILOT.GOLD llamada SV_UTILITIES_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren plantas de tratamiento de agua (ETAP/EDAR/bombeo), eventos de sensores SCADA, incidencias operacionales y muestras de calidad del agua. Empresa de utilities/gestion del agua en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA).
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent en Snowflake Intelligence para Utilities y Gestion del Agua.
- Crea el objeto Snowflake Intelligence predeterminado si no existe
- Crea la base de datos SNOWFLAKE_INTELLIGENCE y el schema AGENTS
- Crea el agente UTILITIES_COPILOT_AGENT conectado a la Semantic View UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT
  con modelo de orquestacion claude-4-sonnet, respondiendo siempre en espanol
  y contexto de negocio: empresa de gestion del agua y utilities en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA)
- Concede acceso al rol PUBLIC al agente y a la Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT
  COMMENT = 'Copilot de Utilities y Gestion del Agua'
  PROFILE = '{"display_name": "Water & Utilities Plant Copilot — Talk to your Plant"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de empresa de gestion del agua y utilities en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA). Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "utilities_analyst",
        "description": "Los datos cubren plantas de tratamiento de agua (ETAP/EDAR/bombeo), eventos de sensores SCADA, incidencias operacionales y muestras de calidad del agua. Empresa de utilities/gestion del agua en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA)."
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

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.

---

## Prueba el Agente

Preguntas de ejemplo para validar el agente:

- "¿Que plantas tienen mayor riesgo de incumplimiento regulatorio?"
- "¿Cuantas alarmas criticas de SCADA estan activas en este momento?"
- "¿Que parametros de calidad del agua estan fuera de limites legales?"
- "Dame el estado operacional de todas las plantas con criticidad alta."

---

## FASE FINAL — App Streamlit en Snowflake

**[CoCo]**
```
Crea una app Streamlit in Snowflake (SiS) en la base de datos UTILITIES_COPILOT, schema AI, llamada UTILITIES_APP.
La app debe conectarse a Snowflake con la sesion activa (get_active_session).
Incluye 4 secciones usando st.tabs:
1. KPI — tarjetas con st.metric con los totales y ratios principales de PLANT_360
2. Distribucion — grafico de barras (st.bar_chart) por categoria/region desde OPERATIONAL_HEALTH
3. Top 20 — tabla interactiva (st.dataframe) ordenada por score de riesgo desde COMPLIANCE_RISK
4. Tendencia — grafico de lineas (st.line_chart) con evolucion temporal de los eventos principales
Usa titulos y etiquetas descriptivos en espanol. Aplica un tema oscuro con st.set_page_config.
```
