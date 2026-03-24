# 📡 Guia del Hackathon: Telco Operations Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Telecomunicaciones
**Base de datos:** TELCO_COPILOT

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
  └── data/telco/
        ├── customers.json                 (350 clientes telco)
        ├── network_events.json            (10,000 eventos de red)
        ├── tickets.json                   (750 tickets de soporte)
        └── billing_usage.json             (2,000 registros de facturacion)
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
Crea una base de datos llamada TELCO_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en TELCO_COPILOT.PUBLIC llamado HACKATHON_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En TELCO_COPILOT.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE TELCO_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_CUSTOMERS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_CUSTOMERS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/customers.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_NETWORK_EVENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_NETWORK_EVENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/network_events.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_TICKETS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_TICKETS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/tickets.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_BILLING_USAGE (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_BILLING_USAGE (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/billing_usage.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en TELCO_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_CUSTOMERS: 350 filas / RAW_NETWORK_EVENTS: 10,000 filas / RAW_TICKETS: 750 filas / RAW_BILLING_USAGE: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado telco_copilot que conecte a la base de datos TELCO_COPILOT.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos TELCO_COPILOT,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: TELCO_COPILOT
- schema: BRONZE
- tables: RAW_CUSTOMERS, RAW_NETWORK_EVENTS, RAW_TICKETS, RAW_BILLING_USAGE
```


### Modelo Silver: STG_CUSTOMERS

```
Crea un modelo dbt models/staging/stg_customers.sql que haga flatten del JSON
de la tabla RAW_CUSTOMERS (source Bronze). Extrae todos los campos relevantes del JSON:
clientes telco (customer_id, company_name, plan, region, segment, monthly_revenue_eur, nps_score, status)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_NETWORK_EVENTS

```
Crea un modelo dbt models/staging/stg_network_events.sql que haga flatten del JSON
de la tabla RAW_NETWORK_EVENTS (source Bronze). Extrae todos los campos relevantes del JSON:
eventos de red (event_id, customer_id, event_type, severity, cell_tower_id, duration_ms, metadata)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_TICKETS

```
Crea un modelo dbt models/staging/stg_tickets.sql que haga flatten del JSON
de la tabla RAW_TICKETS (source Bronze). Extrae todos los campos relevantes del JSON:
tickets de soporte (ticket_id, customer_id, category, priority, description, sla_breached, satisfaction_score)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_BILLING_USAGE

```
Crea un modelo dbt models/staging/stg_billing_usage.sql que haga flatten del JSON
de la tabla RAW_BILLING_USAGE (source Bronze). Extrae todos los campos relevantes del JSON:
registros de facturacion (billing_id, customer_id, service_type, amount_eur, payment_status)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en TELCO_COPILOT.SILVER con datos tipados.


### Modelo Gold: CUSTOMER_360

```
Crea un modelo dbt models/marts/customer_360.sql:
Vista 360 de cada cliente: datos basicos + total_tickets + open_tickets + avg_satisfaction + sla_breach_rate + total_network_events + critical_events + total_billed_eur. JOIN de todas las Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: NETWORK_HEALTH

```
Crea un modelo dbt models/marts/network_health.sql:
Salud de red por cell_tower_id y dia: total_events, critical_events, high_events, avg_duration_ms, distinct_customers_affected, top_event_type.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: REVENUE_AT_RISK

```
Crea un modelo dbt models/marts/revenue_at_risk.sql:
Revenue en riesgo por cliente: monthly_revenue_eur + total_tickets_last_90d + sla_breaches_last_90d + unpaid_amount + risk_score (0-100 basado en status, SLA, tickets, impagos, NPS).
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en TELCO_COPILOT.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion automatica de tickets

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion automatica de tickets de la tabla TELCO_COPILOT.SILVER.STG_TICKETS.

El LLM debe analizar el campo "description" y devolver un JSON con: clasificacion_tecnica (conectividad|hardware|software|cobertura|facturacion|comercial), severidad_estimada (critica|alta|media|baja), departamento_responsable (ingenieria_red|soporte_nivel2|facturacion|comercial|campo), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de tickets de soporte de una empresa de telecomunicaciones."

Crea una vista en el esquema TELCO_COPILOT.AI llamada VW_TICKET_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de cuenta

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de cuenta de la tabla TELCO_COPILOT.GOLD.CUSTOMER_360.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: company_name, plan, region, segment, monthly_revenue_eur, total_tickets, open_tickets, avg_satisfaction, sla_breach_rate, critical_events, total_billed_eur

El prompt del LLM debe decir:
"Eres un analista de cuentas de una empresa de telecomunicaciones. Genera un resumen ejecutivo en espanol (3-4 frases) de este cliente."

Crea una vista en el esquema TELCO_COPILOT.AI llamada VW_ACCOUNT_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de acciones de retencion

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de acciones de retencion de la tabla TELCO_COPILOT.GOLD.REVENUE_AT_RISK.

El LLM debe analizar los datos del registro y devolver un JSON con: nivel_urgencia (alta|media|baja), accion_principal, acciones_complementarias (array 2-3), oferta_sugerida, canal_contacto (telefono|email|visita_presencial), mensaje_personalizado

El prompt del LLM debe decir:
"Eres un experto en retencion de clientes de telecomunicaciones. Analiza este cliente en riesgo y devuelve SOLO un JSON valido."

Crea una vista en el esquema TELCO_COPILOT.AI llamada VW_RETENTION_RECOMMENDATIONS.
Limita a 25 registros con filtro risk_score >= 30.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION LIMIT 3;
- SELECT * FROM TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en TELCO_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en TELCO_COPILOT.GOLD llamada SV_TELCO_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_customers: TELCO_COPILOT.SILVER.STG_CUSTOMERS (PK: customer_id)
- stg_network_events: TELCO_COPILOT.SILVER.STG_NETWORK_EVENTS (PK: event_id)
- stg_tickets: TELCO_COPILOT.SILVER.STG_TICKETS (PK: ticket_id)
- stg_billing_usage: TELCO_COPILOT.SILVER.STG_BILLING_USAGE (PK: billing_id)
- customer_360: TELCO_COPILOT.GOLD.CUSTOMER_360
- network_health: TELCO_COPILOT.GOLD.NETWORK_HEALTH
- revenue_at_risk: TELCO_COPILOT.GOLD.REVENUE_AT_RISK

Relaciones: todas las tablas se relacionan por customer_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una empresa de telecomunicaciones en Espana.
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (TELCO_COPILOT.GOLD.SV_TELCO_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT
  COMMENT = 'Agente de telecomunicaciones para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Telco Operations Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una empresa de telecomunicaciones en Espana. Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "telco_analyst",
          "description": "Consulta datos operativos: Los datos cubren clientes, tickets de soporte, eventos de red (incidencias tecnicas) y facturacion. Las regiones son ciudades espanolas."
        }
      }
    ],
    "tool_resources": {
      "telco_analyst": {
        "semantic_view": "TELCO_COPILOT.GOLD.SV_TELCO_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"Telco Operations Copilot"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a TELCO_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de CUSTOMER_360 con filtros
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
