# 💧 Guia del Hackathon: Water & Utilities Plant Copilot — Talk to your Plant
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Utilities y Gestion del Agua
**Base de datos:** UTILITIES_COPILOT

---

## FASE 0: CREAR CUENTA TRIAL DE SNOWFLAKE (10 minutos)

> **Objetivo:** Cada participante crea su propia cuenta trial de Snowflake para el hackathon.

### Paso 0.1 — Registro en Snowflake

1. Abre el navegador y ve a: **https://signup.snowflake.com/**
2. Rellena el formulario con tu email corporativo
3. En la pantalla de configuracion, selecciona:
   - **Cloud Provider:** AWS
   - **Edition:** Enterprise
   - **Region:** US West (Oregon)
4. Acepta los terminos y haz clic en **"Get Started"**
5. Revisa tu email y haz clic en el enlace de activacion
6. Establece tu usuario y contrasena

> **Importante:** La cuenta trial incluye **$400 en creditos** y dura **30 dias**.

### Paso 0.2 — Primer acceso y configuracion

1. Accede a tu cuenta Snowflake desde la URL que recibiste por email
2. Verifica que tienes el rol **ACCOUNTADMIN**
3. Comprueba que el warehouse **COMPUTE_WH** esta disponible

```
Ejecuta: SELECT CURRENT_ACCOUNT(), CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_REGION();
```

### Paso 0.3 — Activar Cortex Code

1. En Snowsight, ve a **Projects > Cortex Code**
2. Acepta los terminos de uso de Cortex AI
3. Abre una nueva sesion de Cortex Code
4. Verifica con un prompt simple:

```
Dime que version de Snowflake estoy usando con SELECT CURRENT_VERSION();
```

> **Checkpoint:** Cuenta trial activa con Cortex Code funcionando.

---

### Datos disponibles

Los 4 ficheros JSON estan en un repositorio Git de GitHub:

```
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon/
  └── data/utilities/
        ├── plants.json                    (350 plantas/instalaciones)
        ├── sensor_events.json             (10,000 eventos de sensores SCADA)
        ├── incidents.json                 (750 incidencias operacionales)
        └── water_quality.json             (2,000 muestras de calidad del agua)
```

### Arquitectura objetivo

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
         │
   ┌─────▼─────────────────────────────────────────────┐
   │  BRONZE (raw JSON en tablas VARIANT)               │
   │  ↓                                                 │
   │  SILVER (normalizado, flatten, cast, dedup)        │
   │  ↓                                                 │
   │  GOLD (KPIs, metricas, vistas analiticas)          │
   │  ↓                                                 │
   │  AI (3 casos de uso con Cortex)                    │
   │  ↓                                                 │
   │  SEMANTIC VIEW (modelo semantico para NL2SQL)      │
   │  ↓                                                 │
   │  AGENT (Copilot en Snowflake Intelligence)         │
   └────────────────────────────────────────────────────┘
```

### Roles sugeridos en el equipo (5 personas)

| Rol | Que hace | Cuando |
|-----|----------|--------|
| **Ingeniero de ingesta** | Conecta el repo Git y carga los 4 JSON a Bronze | Fase 1 |
| **Ingeniero dbt 1** | Modelos Silver (flatten + cast) | Fase 2 |
| **Ingeniero dbt 2** | Modelos Gold (KPIs + metricas) | Fase 2 |
| **Ingeniero AI** | 3 prompts Cortex + Semantic View + Agent | Fase 3-5 |
| **Demo lead** | Prepara la presentacion desde el minuto 60 | Todo |

---

## FASE 1: BRONZE — Ingesta de JSON via Git Integration (25 minutos)

> **Objetivo:** Conectar el repositorio Git de GitHub a Snowflake y cargar los 4 ficheros JSON como tablas raw en Bronze.

### Paso 1.1 — Crear la base de datos y esquemas

```
Crea una base de datos llamada UTILITIES_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en UTILITIES_COPILOT.PUBLIC llamado HACKATHON_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En UTILITIES_COPILOT.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE UTILITIES_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_PLANTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_PLANTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/plants.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_SENSOR_EVENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_SENSOR_EVENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/sensor_events.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_INCIDENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_INCIDENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/incidents.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_WATER_QUALITY (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_WATER_QUALITY (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/water_quality.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en UTILITIES_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_PLANTS: 350 filas / RAW_SENSOR_EVENTS: 10,000 filas / RAW_INCIDENTS: 750 filas / RAW_WATER_QUALITY: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado utilities_copilot que conecte a la base de datos UTILITIES_COPILOT.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos UTILITIES_COPILOT,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: UTILITIES_COPILOT
- schema: BRONZE
- tables: RAW_PLANTS, RAW_SENSOR_EVENTS, RAW_INCIDENTS, RAW_WATER_QUALITY
```


### Modelo Silver: STG_PLANTS

```
Crea un modelo dbt models/staging/stg_plants.sql que haga flatten del JSON
de la tabla RAW_PLANTS (source Bronze). Extrae todos los campos relevantes del JSON:
plantas/instalaciones (plant_id, plant_name, plant_type, operator, region, zone, capacity_m3_day, current_flow_m3_day, status, criticality, annual_budget_eur)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_SENSOR_EVENTS

```
Crea un modelo dbt models/staging/stg_sensor_events.sql que haga flatten del JSON
de la tabla RAW_SENSOR_EVENTS (source Bronze). Extrae todos los campos relevantes del JSON:
eventos de sensores SCADA (event_id, plant_id, sensor_type, value, alarm_min, alarm_max, is_alarm, subsystem)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_INCIDENTS

```
Crea un modelo dbt models/staging/stg_incidents.sql que haga flatten del JSON
de la tabla RAW_INCIDENTS (source Bronze). Extrae todos los campos relevantes del JSON:
incidencias operacionales (incident_id, plant_id, category, priority, description, affected_population, regulatory_notification_required, repair_cost_eur)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_WATER_QUALITY

```
Crea un modelo dbt models/staging/stg_water_quality.sql que haga flatten del JSON
de la tabla RAW_WATER_QUALITY (source Bronze). Extrae todos los campos relevantes del JSON:
muestras de calidad del agua (sample_id, plant_id, parameter, value, legal_min, legal_max, compliant, sample_point, reported_to_authority)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en UTILITIES_COPILOT.SILVER con datos tipados.


### Modelo Gold: PLANT_360

```
Crea un modelo dbt models/marts/plant_360.sql:
Vista 360 de cada planta: datos basicos + total_sensor_alarms + alarm_rate_pct + total_incidents + open_incidents + avg_repair_cost_eur + quality_non_compliant_count + regulatory_notifications + days_since_last_inspection. JOIN de todas las Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: OPERATIONAL_HEALTH

```
Crea un modelo dbt models/marts/operational_health.sql:
Salud operacional por planta y dia: total_sensor_events, alarm_count, alarm_rate_pct, top_alarm_sensor, subsystems_affected, avg_flow_m3h, energy_consumption_kWh.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: COMPLIANCE_RISK

```
Crea un modelo dbt models/marts/compliance_risk.sql:
Riesgo de compliance por planta: non_compliant_samples_last_90d + regulatory_notifications + open_critical_incidents + alarm_rate + days_overdue_inspection + compliance_risk_score (0-100).
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en UTILITIES_COPILOT.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion de incidencias operacionales

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion de incidencias operacionales de la tabla UTILITIES_COPILOT.SILVER.STG_INCIDENTS.

El LLM debe analizar el campo "description" y devolver un JSON con: tipo_incidencia (averia_bomba|calidad_agua|fuga_red|fallo_cloracion|alarma_deposito|rotura_tuberia|fallo_scada|contaminacion), urgencia (critica|alta|media|baja), equipo_responsable (operaciones|calidad|mantenimiento|laboratorio|regulatorio|emergencias), notificacion_regulatoria (si|no), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de incidencias de una empresa de gestion del agua (utilities). Clasifica la incidencia descrita."

Crea una vista en el esquema UTILITIES_COPILOT.AI llamada VW_INCIDENT_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de planta (Talk to your Plant)

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de planta (talk to your plant) de la tabla UTILITIES_COPILOT.GOLD.PLANT_360.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: plant_name, plant_type, operator, region, status, criticality, total_sensor_alarms, total_incidents, open_incidents, quality_non_compliant_count, regulatory_notifications

El prompt del LLM debe decir:
"Eres un director de operaciones de una planta de tratamiento de agua. Genera un resumen ejecutivo en espanol (3-4 frases) del estado operacional de esta instalacion."

Crea una vista en el esquema UTILITIES_COPILOT.AI llamada VW_PLANT_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de accion operacional

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de accion operacional de la tabla UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK.

El LLM debe analizar los datos del registro y devolver un JSON con: nivel_riesgo_regulatorio (alto|medio|bajo), accion_principal (inspeccion_urgente|ajuste_tratamiento|mantenimiento_preventivo|notificacion_autoridad|monitorizacion_intensiva), parametros_criticos (array 2-3), plazo_maximo_horas (numero), impacto_poblacion (numero_personas), recomendacion_tecnica

El prompt del LLM debe decir:
"Eres un experto en gestion de utilities y compliance regulatorio del agua. Analiza esta planta y devuelve SOLO un JSON valido."

Crea una vista en el esquema UTILITIES_COPILOT.AI llamada VW_OPERATIONAL_RECOMMENDATIONS.
Limita a 25 registros con filtro compliance_risk_score >= 30.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION LIMIT 3;
- SELECT * FROM UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en UTILITIES_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en UTILITIES_COPILOT.GOLD llamada SV_UTILITIES_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_plants: UTILITIES_COPILOT.SILVER.STG_PLANTS (PK: plant_id)
- stg_sensor_events: UTILITIES_COPILOT.SILVER.STG_SENSOR_EVENTS (PK: event_id)
- stg_incidents: UTILITIES_COPILOT.SILVER.STG_INCIDENTS (PK: incident_id)
- stg_water_quality: UTILITIES_COPILOT.SILVER.STG_WATER_QUALITY (PK: sample_id)
- plant_360: UTILITIES_COPILOT.GOLD.PLANT_360
- operational_health: UTILITIES_COPILOT.GOLD.OPERATIONAL_HEALTH
- compliance_risk: UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK

Relaciones: todas las tablas se relacionan por plant_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una empresa de gestion del agua y utilities en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA).
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT
  COMMENT = 'Agente de utilities y gestion del agua para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Water & Utilities Plant Copilot — Talk to your Plant"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una empresa de gestion del agua y utilities en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA). Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "utilities_analyst",
          "description": "Consulta datos operativos: Los datos cubren plantas de tratamiento de agua (ETAP/EDAR/bombeo), eventos de sensores SCADA, incidencias operacionales y muestras de calidad del agua. Empresa de utilities/gestion del agua en Espana (Veolia, Aguas de Barcelona, Canal de Isabel II, EMASESA)."
        }
      }
    ],
    "tool_resources": {
      "utilities_analyst": {
        "semantic_view": "UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"Water & Utilities Plant Copilot — Talk to your Plant"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a UTILITIES_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de PLANT_360 con filtros
3. Para cada registro seleccionado, mostrar el resumen AI
4. Grafico con los top 10 por risk/score
```

---

## Tips para la Demo (5 minutos)

| Minuto | Contenido |
|--------|-----------|
| 0:00 - 0:30 | Presentar al equipo y enfoque |
| 0:30 - 1:00 | Pipeline Bronze > Silver > Gold |
| 1:00 - 2:30 | Demo 3 casos AI en vivo |
| 2:30 - 3:30 | Agente en Snowflake Intelligence |
| 3:30 - 4:30 | Streamlit o valor de negocio |
| 4:30 - 5:00 | Conclusiones |

---

## FASE FINAL: REVISION DE COSTES DEL HACKATHON

```sql
SELECT service_type, ROUND(SUM(credits_used), 2) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY service_type ORDER BY total_credits DESC;

SELECT function_name, model_name, SUM(tokens) AS total_tokens, ROUND(SUM(token_credits), 4) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY function_name, model_name ORDER BY total_credits DESC;

SELECT username, ROUND(SUM(credits), 4) AS total_credits, SUM(request_count) AS total_requests
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_ANALYST_USAGE_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY username ORDER BY total_credits DESC;

SELECT warehouse_name, ROUND(SUM(credits_used), 2) AS total_credits, ROUND(SUM(credits_used) * 3.00, 2) AS estimated_cost_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY warehouse_name ORDER BY total_credits DESC;
```

> **Nota:** Las vistas de ACCOUNT_USAGE tienen un retraso de hasta 2 horas.
