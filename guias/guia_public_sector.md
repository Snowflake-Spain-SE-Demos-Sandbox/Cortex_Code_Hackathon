# 🌐 Guia del Hackathon: Smart City Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Sector Publico
**Base de datos:** PUBLIC_SECTOR_COPILOT

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
  └── data/public_sector/
        ├── citizens.json                  (350 ciudadanos)
        ├── service_requests.json          (10,000 solicitudes de servicio)
        ├── inspection_reports.json        (750 informes de inspeccion)
        └── budget_items.json              (2,000 partidas presupuestarias)
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
Crea una base de datos llamada PUBLIC_SECTOR_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en PUBLIC_SECTOR_COPILOT.PUBLIC llamado HACKATHON_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En PUBLIC_SECTOR_COPILOT.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE PUBLIC_SECTOR_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_CITIZENS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_CITIZENS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/citizens.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_SERVICE_REQUESTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_SERVICE_REQUESTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/service_requests.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_INSPECTIONS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_INSPECTIONS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/inspection_reports.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_BUDGET (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_BUDGET (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/budget_items.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en PUBLIC_SECTOR_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_CITIZENS: 350 filas / RAW_SERVICE_REQUESTS: 10,000 filas / RAW_INSPECTIONS: 750 filas / RAW_BUDGET: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado public_sector_copilot que conecte a la base de datos PUBLIC_SECTOR_COPILOT.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos PUBLIC_SECTOR_COPILOT,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: PUBLIC_SECTOR_COPILOT
- schema: BRONZE
- tables: RAW_CITIZENS, RAW_SERVICE_REQUESTS, RAW_INSPECTIONS, RAW_BUDGET
```


### Modelo Silver: STG_CITIZENS

```
Crea un modelo dbt models/staging/stg_citizens.sql que haga flatten del JSON
de la tabla RAW_CITIZENS (source Bronze). Extrae todos los campos relevantes del JSON:
ciudadanos (citizen_id, full_name, age, district, municipality, profile_type, preferred_channel, digital_literacy)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_SERVICE_REQUESTS

```
Crea un modelo dbt models/staging/stg_service_requests.sql que haga flatten del JSON
de la tabla RAW_SERVICE_REQUESTS (source Bronze). Extrae todos los campos relevantes del JSON:
solicitudes de servicio (request_id, citizen_id, request_type, department, channel, status, priority, response_days)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_INSPECTIONS

```
Crea un modelo dbt models/staging/stg_inspections.sql que haga flatten del JSON
de la tabla RAW_INSPECTIONS (source Bronze). Extrae todos los campos relevantes del JSON:
informes de inspeccion (report_id, inspector_id, category, severity, description, fine_amount_eur, follow_up_required)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_BUDGET

```
Crea un modelo dbt models/staging/stg_budget.sql que haga flatten del JSON
de la tabla RAW_BUDGET (source Bronze). Extrae todos los campos relevantes del JSON:
partidas presupuestarias (item_id, department, fiscal_year, budget_line, approved_amount_eur, executed_amount_eur, execution_pct)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en PUBLIC_SECTOR_COPILOT.SILVER con datos tipados.


### Modelo Gold: CITIZEN_360

```
Crea un modelo dbt models/marts/citizen_360.sql:
Vista 360 del ciudadano: datos basicos + total_requests + resolved_requests + avg_response_days + total_complaints + digital_channel_usage_pct + days_since_last_interaction. JOIN de Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: SERVICE_HEALTH

```
Crea un modelo dbt models/marts/service_health.sql:
Salud de servicios por department y periodo: total_requests, resolved_count, avg_response_days, overdue_count, citizen_satisfaction_proxy, top_request_type.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: BUDGET_EXECUTION

```
Crea un modelo dbt models/marts/budget_execution.sql:
Ejecucion presupuestaria por department y quarter: approved_total_eur, executed_total_eur, execution_pct, pending_eur, overbudget_items, underexecuted_items.
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en PUBLIC_SECTOR_COPILOT.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion de informes de inspeccion

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion de informes de inspeccion de la tabla PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS.

El LLM debe analizar el campo "description" y devolver un JSON con: ambito (urbanismo|sanidad|medio_ambiente|seguridad_laboral|accesibilidad|actividades|obra_publica|ruido), gravedad (leve|grave|muy_grave), organo_competente (urbanismo|sanidad|medio_ambiente|policia_local|bomberos|juzgado), requiere_sancion (si|no), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de inspecciones de una administracion publica."

Crea una vista en el esquema PUBLIC_SECTOR_COPILOT.AI llamada VW_INSPECTION_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de servicio publico

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de servicio publico de la tabla PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: department, total_requests, resolved_count, avg_response_days, overdue_count, top_request_type

El prompt del LLM debe decir:
"Eres un analista de gestion publica. Genera un resumen ejecutivo en espanol (3-4 frases) del estado de este servicio/departamento."

Crea una vista en el esquema PUBLIC_SECTOR_COPILOT.AI llamada VW_SERVICE_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de mejora de servicio

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de mejora de servicio de la tabla PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.

El LLM debe analizar los datos del registro y devolver un JSON con: nivel_eficiencia (bueno|mejorable|deficiente), accion_principal (digitalizacion|formacion|reasignacion_recursos|simplificacion_proceso), areas_mejora (array 2-3), ahorro_estimado_pct, plazo_implementacion_meses, impacto_ciudadano (alto|medio|bajo)

El prompt del LLM debe decir:
"Eres un consultor de transformacion digital del sector publico. Analiza este departamento y devuelve SOLO un JSON valido."

Crea una vista en el esquema PUBLIC_SECTOR_COPILOT.AI llamada VW_SERVICE_IMPROVEMENT.
Limita a 25 registros con filtro avg_response_days > 15.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION LIMIT 3;
- SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en PUBLIC_SECTOR_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en PUBLIC_SECTOR_COPILOT.GOLD llamada SV_PUBLIC_SECTOR_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_citizens: PUBLIC_SECTOR_COPILOT.SILVER.STG_CITIZENS (PK: citizen_id)
- stg_service_requests: PUBLIC_SECTOR_COPILOT.SILVER.STG_SERVICE_REQUESTS (PK: request_id)
- stg_inspections: PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS (PK: report_id)
- stg_budget: PUBLIC_SECTOR_COPILOT.SILVER.STG_BUDGET (PK: item_id)
- citizen_360: PUBLIC_SECTOR_COPILOT.GOLD.CITIZEN_360
- service_health: PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH
- budget_execution: PUBLIC_SECTOR_COPILOT.GOLD.BUDGET_EXECUTION

Relaciones: todas las tablas se relacionan por citizen_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una administracion publica / ayuntamiento en Espana.
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.PUBLIC_SECTOR_COPILOT_AGENT
  COMMENT = 'Agente de sector publico para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Smart City Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una administracion publica / ayuntamiento en Espana. Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "public_sector_analyst",
          "description": "Consulta datos operativos: Los datos cubren ciudadanos, solicitudes de servicio, informes de inspeccion y presupuestos. Administracion publica / ayuntamiento en Espana."
        }
      }
    ],
    "tool_resources": {
      "public_sector_analyst": {
        "semantic_view": "PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.PUBLIC_SECTOR_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"Smart City Copilot"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a PUBLIC_SECTOR_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de CITIZEN_360 con filtros
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
