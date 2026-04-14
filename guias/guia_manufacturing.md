# 🏭 Smart Factory Copilot
**DB:** `MANUFACTURING_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada MANUFACTURING_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE MANUFACTURING_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en MANUFACTURING_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/;
```

**[CoCo]** Tablas Bronze
```
En MANUFACTURING_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_EQUIPMENT desde data/manufacturing/equipment.json
- RAW_SENSOR_READINGS desde data/manufacturing/sensor_readings.json
- RAW_WORK_ORDERS desde data/manufacturing/work_orders.json
- RAW_PRODUCTION_BATCHES desde data/manufacturing/production_batches.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_EQUIPMENT; -- esperado: 350
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_SENSOR_READINGS; -- esperado: 10,000
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_WORK_ORDERS; -- esperado: 750
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_PRODUCTION_BATCHES; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Asegurate de estar en **Projects > Workspaces** en Snowsight para ejecutar dbt.

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado manufacturing_copilot conectado a MANUFACTURING_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_EQUIPMENT, RAW_SENSOR_READINGS, RAW_WORK_ORDERS, RAW_PRODUCTION_BATCHES
```

**[CoCo]** Silver: STG_EQUIPMENT
```
Crea models/staging/stg_equipment.sql que haga flatten de RAW_EQUIPMENT.
Campos: equipos/maquinas (equipment_id, name, type, plant, production_line, manufacturer, status, criticality, hours_operated)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_SENSOR_READINGS
```
Crea models/staging/stg_sensor_readings.sql que haga flatten de RAW_SENSOR_READINGS.
Campos: lecturas de sensores IoT (reading_id, equipment_id, sensor_type, value, threshold_min, threshold_max, is_anomaly)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_WORK_ORDERS
```
Crea models/staging/stg_work_orders.sql que haga flatten de RAW_WORK_ORDERS.
Campos: ordenes de trabajo (work_order_id, equipment_id, category, priority, description, estimated_hours, spare_parts_cost_eur)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_PRODUCTION_BATCHES
```
Crea models/staging/stg_production_batches.sql que haga flatten de RAW_PRODUCTION_BATCHES.
Campos: lotes de produccion (batch_id, product_name, production_line, planned_quantity, actual_quantity, defect_count, material_cost_eur, quality_grade)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_EQUIPMENT; -- esperado: 350
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_SENSOR_READINGS; -- esperado: 10,000
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS; -- esperado: 750
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_PRODUCTION_BATCHES; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: EQUIPMENT_360
```
Crea models/marts/equipment_360.sql:
Vista 360 de cada equipo: datos basicos + total_sensor_anomalies + total_work_orders + open_work_orders + total_maintenance_cost_eur + avg_production_quality + hours_operated + days_since_last_maintenance. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: PRODUCTION_HEALTH
```
Crea models/marts/production_health.sql:
Salud de produccion por production_line, plant y dia: total_batches, total_produced, total_defects, defect_rate_pct, avg_quality_grade, total_material_cost, total_energy_cost.
Materializa como table en GOLD.
```

**[CoCo]** Gold: MAINTENANCE_RISK
```
Crea models/marts/maintenance_risk.sql:
Riesgo de mantenimiento por equipo: hours_operated + anomaly_count_last_90d + pending_work_orders + avg_sensor_deviation + maintenance_risk_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
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

**[CoCo]** Caso 1: Clasificacion de ordenes de trabajo
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS.
Analiza el campo "description" y devuelve un JSON con: tipo_mantenimiento (correctivo|preventivo|predictivo|mejora|inspeccion|calibracion), urgencia (critica|alta|media|baja), especialidad_requerida (mecanica|electrica|instrumentacion|automatizacion|general), impacto_produccion (alto|medio|bajo|nulo), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de ordenes de trabajo de una planta industrial."
Crea la vista MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de equipo
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: name, type, plant, production_line, status, criticality, hours_operated, total_sensor_anomalies, total_work_orders, total_maintenance_cost_eur
Prompt del sistema: "Eres un ingeniero de fiabilidad industrial. Genera un resumen ejecutivo en espanol (3-4 frases) del estado de este equipo."
Crea la vista MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de mantenimiento predictivo
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK.
Devuelve un JSON con: nivel_riesgo_fallo (alto|medio|bajo), accion_recomendada (sustitucion|reparacion|inspeccion|monitorizacion), componente_sospechoso, plazo_maximo_dias (numero), impacto_si_falla (linea_parada|produccion_reducida|calidad_degradada), coste_estimado_eur
Prompt del sistema: "Eres un experto en mantenimiento predictivo industrial. Analiza este equipo y devuelve SOLO un JSON valido."
Filtra WHERE maintenance_risk_score >= 30.
Crea la vista MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en MANUFACTURING_COPILOT.GOLD llamada SV_MANUFACTURING_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren equipos industriales, lecturas de sensores IoT, ordenes de trabajo/mantenimiento y lotes de produccion. Planta industrial en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent en Snowflake Intelligence para Manufacturing e Industria.
- Crea el objeto Snowflake Intelligence predeterminado si no existe
- Crea la base de datos SNOWFLAKE_INTELLIGENCE y el schema AGENTS
- Crea el agente MANUFACTURING_COPILOT_AGENT conectado a la Semantic View MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT
  con modelo de orquestacion claude-4-sonnet, respondiendo siempre en espanol
  y contexto de negocio: empresa manufacturera/industrial en Espana
- Concede acceso al rol PUBLIC al agente y a la Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT
  COMMENT = 'Copilot de Manufacturing e Industria'
  PROFILE = '{"display_name": "Smart Factory Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de empresa manufacturera/industrial en Espana. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "manufacturing_analyst",
        "description": "Los datos cubren equipos industriales, lecturas de sensores IoT, ordenes de trabajo/mantenimiento y lotes de produccion. Planta industrial en Espana."
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

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.

---

## Prueba el Agente

Preguntas de ejemplo para validar el agente:

- "¿Que equipos tienen mayor riesgo de fallo en los proximos 7 dias?"
- "¿Cuantas ordenes de trabajo criticas estan pendientes por planta?"
- "¿Cual es la tasa de defectos por linea de produccion esta semana?"
- "¿Que sensores IoT generan mas anomalias por equipo?"

---

## FASE FINAL — App Streamlit en Snowflake

**[CoCo]**
```
Crea una app Streamlit in Snowflake (SiS) en la base de datos MANUFACTURING_COPILOT, schema AI, llamada MANUFACTURING_APP.
La app debe conectarse a Snowflake con la sesion activa (get_active_session).
Incluye 4 secciones usando st.tabs:
1. KPI — tarjetas con st.metric con los totales y ratios principales de EQUIPMENT_360
2. Distribucion — grafico de barras (st.bar_chart) por categoria/region desde PRODUCTION_HEALTH
3. Top 20 — tabla interactiva (st.dataframe) ordenada por score de riesgo desde MAINTENANCE_RISK
4. Tendencia — grafico de lineas (st.line_chart) con evolucion temporal de los eventos principales
Usa titulos y etiquetas descriptivos en espanol. Aplica un tema oscuro con st.set_page_config.
```
