# 🏦 Guia del Hackathon: FSI Risk & Compliance Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Servicios Financieros
**Base de datos:** FSI_COPILOT

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
  └── data/fsi/
        ├── clients.json                   (350 clientes financieros)
        ├── transactions.json              (10,000 transacciones)
        ├── risk_alerts.json               (750 alertas de riesgo/fraude)
        └── accounts.json                  (2,000 cuentas y productos)
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
Crea una base de datos llamada FSI_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en FSI_COPILOT.PUBLIC llamado HACKATHON_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY FSI_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY FSI_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En FSI_COPILOT.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE FSI_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_CLIENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_CLIENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/clients.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_TRANSACTIONS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_TRANSACTIONS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/transactions.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_RISK_ALERTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_RISK_ALERTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/risk_alerts.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_ACCOUNTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_ACCOUNTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/accounts.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en FSI_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_CLIENTS: 350 filas / RAW_TRANSACTIONS: 10,000 filas / RAW_RISK_ALERTS: 750 filas / RAW_ACCOUNTS: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado fsi_copilot que conecte a la base de datos FSI_COPILOT.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos FSI_COPILOT,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: FSI_COPILOT
- schema: BRONZE
- tables: RAW_CLIENTS, RAW_TRANSACTIONS, RAW_RISK_ALERTS, RAW_ACCOUNTS
```


### Modelo Silver: STG_CLIENTS

```
Crea un modelo dbt models/staging/stg_clients.sql que haga flatten del JSON
de la tabla RAW_CLIENTS (source Bronze). Extrae todos los campos relevantes del JSON:
clientes financieros (client_id, company_name, segment, product_type, monthly_volume_eur, risk_profile, kyc_status, aml_score)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_TRANSACTIONS

```
Crea un modelo dbt models/staging/stg_transactions.sql que haga flatten del JSON
de la tabla RAW_TRANSACTIONS (source Bronze). Extrae todos los campos relevantes del JSON:
transacciones (transaction_id, client_id, transaction_type, amount_eur, channel, status, country)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_RISK_ALERTS

```
Crea un modelo dbt models/staging/stg_risk_alerts.sql que haga flatten del JSON
de la tabla RAW_RISK_ALERTS (source Bronze). Extrae todos los campos relevantes del JSON:
alertas de riesgo/fraude (alert_id, client_id, category, severity, description, rule_triggered, regulatory_report_required)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_ACCOUNTS

```
Crea un modelo dbt models/staging/stg_accounts.sql que haga flatten del JSON
de la tabla RAW_ACCOUNTS (source Bronze). Extrae todos los campos relevantes del JSON:
cuentas y productos (account_id, client_id, account_type, balance_eur, interest_rate_pct, status)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en FSI_COPILOT.SILVER con datos tipados.


### Modelo Gold: CLIENT_360

```
Crea un modelo dbt models/marts/client_360.sql:
Vista 360 del cliente financiero: datos basicos + total_accounts + total_balance_eur + total_transactions + avg_transaction_amount + total_alerts + open_alerts + kyc_status + aml_score. JOIN de todas las Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: TRANSACTION_HEALTH

```
Crea un modelo dbt models/marts/transaction_health.sql:
Salud transaccional por tipo y dia: total_transactions, total_volume_eur, rejected_count, avg_amount, distinct_clients, top_channel.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: RISK_EXPOSURE

```
Crea un modelo dbt models/marts/risk_exposure.sql:
Exposicion al riesgo por cliente: monthly_volume_eur + total_alerts_last_90d + critical_alerts + accounts_in_mora + kyc_expired + compliance_score (0-100).
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en FSI_COPILOT.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion automatica de alertas de riesgo

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion automatica de alertas de riesgo de la tabla FSI_COPILOT.SILVER.STG_RISK_ALERTS.

El LLM debe analizar el campo "description" y devolver un JSON con: tipo_riesgo (fraude|blanqueo|compliance|credito|operacional|ciber), urgencia (critica|alta|media|baja), departamento (fraude|compliance|riesgos|ciberseguridad|legal), requiere_reporte_regulatorio (si|no), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un analista de riesgos de una entidad financiera."

Crea una vista en el esquema FSI_COPILOT.AI llamada VW_RISK_ALERT_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de cliente

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de cliente de la tabla FSI_COPILOT.GOLD.CLIENT_360.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: company_name, segment, product_type, monthly_volume_eur, total_accounts, total_balance_eur, total_alerts, aml_score, kyc_status

El prompt del LLM debe decir:
"Eres un analista de banca privada. Genera un resumen ejecutivo en espanol (3-4 frases) del perfil financiero de este cliente."

Crea una vista en el esquema FSI_COPILOT.AI llamada VW_CLIENT_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de accion de compliance

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de accion de compliance de la tabla FSI_COPILOT.GOLD.RISK_EXPOSURE.

El LLM debe analizar los datos del registro y devolver un JSON con: nivel_riesgo (alto|medio|bajo), accion_principal, acciones_regulatorias (array 2-3), requiere_SAR (si|no), plazo_maximo_dias (numero), recomendacion_kyc

El prompt del LLM debe decir:
"Eres un experto en compliance financiero (KYC/AML). Analiza este cliente y devuelve SOLO un JSON valido."

Crea una vista en el esquema FSI_COPILOT.AI llamada VW_COMPLIANCE_RECOMMENDATIONS.
Limita a 25 registros con filtro compliance_score >= 40.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION LIMIT 3;
- SELECT * FROM FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en FSI_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en FSI_COPILOT.GOLD llamada SV_FSI_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_clients: FSI_COPILOT.SILVER.STG_CLIENTS (PK: client_id)
- stg_transactions: FSI_COPILOT.SILVER.STG_TRANSACTIONS (PK: transaction_id)
- stg_risk_alerts: FSI_COPILOT.SILVER.STG_RISK_ALERTS (PK: alert_id)
- stg_accounts: FSI_COPILOT.SILVER.STG_ACCOUNTS (PK: account_id)
- client_360: FSI_COPILOT.GOLD.CLIENT_360
- transaction_health: FSI_COPILOT.GOLD.TRANSACTION_HEALTH
- risk_exposure: FSI_COPILOT.GOLD.RISK_EXPOSURE

Relaciones: todas las tablas se relacionan por client_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una entidad financiera en Espana (banca, inversiones, seguros).
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (FSI_COPILOT.GOLD.SV_FSI_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.FSI_COPILOT_AGENT
  COMMENT = 'Agente de servicios financieros para consultas en lenguaje natural'
  PROFILE = '{"display_name": "FSI Risk & Compliance Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una entidad financiera en Espana (banca, inversiones, seguros). Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "fsi_analyst",
          "description": "Consulta datos operativos: Los datos cubren clientes financieros, transacciones, alertas de riesgo/fraude/AML y cuentas/productos. Entidad financiera en Espana."
        }
      }
    ],
    "tool_resources": {
      "fsi_analyst": {
        "semantic_view": "FSI_COPILOT.GOLD.SV_FSI_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.FSI_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW FSI_COPILOT.GOLD.SV_FSI_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"FSI Risk & Compliance Copilot"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a FSI_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de CLIENT_360 con filtros
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
