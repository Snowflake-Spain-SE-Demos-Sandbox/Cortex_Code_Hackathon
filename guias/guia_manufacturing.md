# 🏭 Guia del Hackathon: Smart Factory Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Manufacturing e Industria
**Base de datos:** MANUFACTURING_COPILOT

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
  └── data/manufacturing/
        ├── equipment.json                 (350 equipos/maquinas)
        ├── sensor_readings.json           (10,000 lecturas de sensores IoT)
        ├── work_orders.json               (750 ordenes de trabajo)
        └── production_batches.json        (2,000 lotes de produccion)
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
Crea una base de datos llamada MANUFACTURING_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en MANUFACTURING_COPILOT.PUBLIC llamado HACKATHON_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En MANUFACTURING_COPILOT.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE MANUFACTURING_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_EQUIPMENT (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_EQUIPMENT (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/equipment.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_SENSOR_READINGS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_SENSOR_READINGS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/sensor_readings.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_WORK_ORDERS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_WORK_ORDERS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/work_orders.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_PRODUCTION_BATCHES (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_PRODUCTION_BATCHES (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/production_batches.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en MANUFACTURING_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_EQUIPMENT: 350 filas / RAW_SENSOR_READINGS: 10,000 filas / RAW_WORK_ORDERS: 750 filas / RAW_PRODUCTION_BATCHES: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado manufacturing_copilot que conecte a la base de datos MANUFACTURING_COPILOT.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos MANUFACTURING_COPILOT,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: MANUFACTURING_COPILOT
- schema: BRONZE
- tables: RAW_EQUIPMENT, RAW_SENSOR_READINGS, RAW_WORK_ORDERS, RAW_PRODUCTION_BATCHES
```


### Modelo Silver: STG_EQUIPMENT

```
Crea un modelo dbt models/staging/stg_equipment.sql que haga flatten del JSON
de la tabla RAW_EQUIPMENT (source Bronze). Extrae todos los campos relevantes del JSON:
equipos/maquinas (equipment_id, name, type, plant, production_line, manufacturer, status, criticality, hours_operated)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_SENSOR_READINGS

```
Crea un modelo dbt models/staging/stg_sensor_readings.sql que haga flatten del JSON
de la tabla RAW_SENSOR_READINGS (source Bronze). Extrae todos los campos relevantes del JSON:
lecturas de sensores IoT (reading_id, equipment_id, sensor_type, value, threshold_min, threshold_max, is_anomaly)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_WORK_ORDERS

```
Crea un modelo dbt models/staging/stg_work_orders.sql que haga flatten del JSON
de la tabla RAW_WORK_ORDERS (source Bronze). Extrae todos los campos relevantes del JSON:
ordenes de trabajo (work_order_id, equipment_id, category, priority, description, estimated_hours, spare_parts_cost_eur)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_PRODUCTION_BATCHES

```
Crea un modelo dbt models/staging/stg_production_batches.sql que haga flatten del JSON
de la tabla RAW_PRODUCTION_BATCHES (source Bronze). Extrae todos los campos relevantes del JSON:
lotes de produccion (batch_id, product_name, production_line, planned_quantity, actual_quantity, defect_count, material_cost_eur, quality_grade)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en MANUFACTURING_COPILOT.SILVER con datos tipados.


### Modelo Gold: EQUIPMENT_360

```
Crea un modelo dbt models/marts/equipment_360.sql:
Vista 360 de cada equipo: datos basicos + total_sensor_anomalies + total_work_orders + open_work_orders + total_maintenance_cost_eur + avg_production_quality + hours_operated + days_since_last_maintenance. JOIN de todas las Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: PRODUCTION_HEALTH

```
Crea un modelo dbt models/marts/production_health.sql:
Salud de produccion por production_line, plant y dia: total_batches, total_produced, total_defects, defect_rate_pct, avg_quality_grade, total_material_cost, total_energy_cost.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: MAINTENANCE_RISK

```
Crea un modelo dbt models/marts/maintenance_risk.sql:
Riesgo de mantenimiento por equipo: hours_operated + anomaly_count_last_90d + pending_work_orders + avg_sensor_deviation + maintenance_risk_score (0-100).
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en MANUFACTURING_COPILOT.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion de ordenes de trabajo

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion de ordenes de trabajo de la tabla MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS.

El LLM debe analizar el campo "description" y devolver un JSON con: tipo_mantenimiento (correctivo|preventivo|predictivo|mejora|inspeccion|calibracion), urgencia (critica|alta|media|baja), especialidad_requerida (mecanica|electrica|instrumentacion|automatizacion|general), impacto_produccion (alto|medio|bajo|nulo), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de ordenes de trabajo de una planta industrial."

Crea una vista en el esquema MANUFACTURING_COPILOT.AI llamada VW_WORK_ORDER_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de equipo

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de equipo de la tabla MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: name, type, plant, production_line, status, criticality, hours_operated, total_sensor_anomalies, total_work_orders, total_maintenance_cost_eur

El prompt del LLM debe decir:
"Eres un ingeniero de fiabilidad industrial. Genera un resumen ejecutivo en espanol (3-4 frases) del estado de este equipo."

Crea una vista en el esquema MANUFACTURING_COPILOT.AI llamada VW_EQUIPMENT_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de mantenimiento predictivo

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de mantenimiento predictivo de la tabla MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK.

El LLM debe analizar los datos del registro y devolver un JSON con: nivel_riesgo_fallo (alto|medio|bajo), accion_recomendada (sustitucion|reparacion|inspeccion|monitorizacion), componente_sospechoso, plazo_maximo_dias (numero), impacto_si_falla (linea_parada|produccion_reducida|calidad_degradada), coste_estimado_eur

El prompt del LLM debe decir:
"Eres un experto en mantenimiento predictivo industrial. Analiza este equipo y devuelve SOLO un JSON valido."

Crea una vista en el esquema MANUFACTURING_COPILOT.AI llamada VW_PREDICTIVE_MAINTENANCE.
Limita a 25 registros con filtro maintenance_risk_score >= 30.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION LIMIT 3;
- SELECT * FROM MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en MANUFACTURING_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en MANUFACTURING_COPILOT.GOLD llamada SV_MANUFACTURING_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_equipment: MANUFACTURING_COPILOT.SILVER.STG_EQUIPMENT (PK: equipment_id)
- stg_sensor_readings: MANUFACTURING_COPILOT.SILVER.STG_SENSOR_READINGS (PK: reading_id)
- stg_work_orders: MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS (PK: work_order_id)
- stg_production_batches: MANUFACTURING_COPILOT.SILVER.STG_PRODUCTION_BATCHES (PK: batch_id)
- equipment_360: MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360
- production_health: MANUFACTURING_COPILOT.GOLD.PRODUCTION_HEALTH
- maintenance_risk: MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK

Relaciones: todas las tablas se relacionan por equipment_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una empresa manufacturera/industrial en Espana.
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT
  COMMENT = 'Agente de manufacturing e industria para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Smart Factory Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una empresa manufacturera/industrial en Espana. Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "manufacturing_analyst",
          "description": "Consulta datos operativos: Los datos cubren equipos industriales, lecturas de sensores IoT, ordenes de trabajo/mantenimiento y lotes de produccion. Planta industrial en Espana."
        }
      }
    ],
    "tool_resources": {
      "manufacturing_analyst": {
        "semantic_view": "MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"Smart Factory Copilot"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a MANUFACTURING_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de EQUIPMENT_360 con filtros
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
