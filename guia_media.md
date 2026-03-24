# 📺 Guia del Hackathon: Media & Audience Intelligence Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Media, Entertainment y Advertising
**Base de datos:** MEDIA_COPILOT

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
  └── data/media/
        ├── audiences.json                 (350 perfiles de audiencia)
        ├── engagement_events.json         (10,000 eventos de engagement)
        ├── content_feedback.json          (750 feedback de contenido)
        └── campaigns.json                 (2,000 campanas publicitarias)
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
Crea una base de datos llamada MEDIA_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en MEDIA_COPILOT.PUBLIC llamado HACKATHON_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY MEDIA_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY MEDIA_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En MEDIA_COPILOT.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE MEDIA_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_AUDIENCES (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_AUDIENCES (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/audiences.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_ENGAGEMENT (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_ENGAGEMENT (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/engagement_events.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_FEEDBACK (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_FEEDBACK (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/content_feedback.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_CAMPAIGNS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_CAMPAIGNS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/campaigns.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en MEDIA_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_AUDIENCES: 350 filas / RAW_ENGAGEMENT: 10,000 filas / RAW_FEEDBACK: 750 filas / RAW_CAMPAIGNS: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado media_copilot que conecte a la base de datos MEDIA_COPILOT.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos MEDIA_COPILOT,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: MEDIA_COPILOT
- schema: BRONZE
- tables: RAW_AUDIENCES, RAW_ENGAGEMENT, RAW_FEEDBACK, RAW_CAMPAIGNS
```


### Modelo Silver: STG_AUDIENCES

```
Crea un modelo dbt models/staging/stg_audiences.sql que haga flatten del JSON
de la tabla RAW_AUDIENCES (source Bronze). Extrae todos los campos relevantes del JSON:
perfiles de audiencia (audience_id, username, region, subscription_type, preferred_genre, age_group, watch_hours_monthly, status)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_ENGAGEMENT

```
Crea un modelo dbt models/staging/stg_engagement.sql que haga flatten del JSON
de la tabla RAW_ENGAGEMENT (source Bronze). Extrae todos los campos relevantes del JSON:
eventos de engagement (event_id, audience_id, event_type, content_id, content_type, genre, duration_seconds, platform)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_FEEDBACK

```
Crea un modelo dbt models/staging/stg_feedback.sql que haga flatten del JSON
de la tabla RAW_FEEDBACK (source Bronze). Extrae todos los campos relevantes del JSON:
feedback de contenido (feedback_id, audience_id, category, priority, description, sla_breached, satisfaction_score)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_CAMPAIGNS

```
Crea un modelo dbt models/staging/stg_campaigns.sql que haga flatten del JSON
de la tabla RAW_CAMPAIGNS (source Bronze). Extrae todos los campos relevantes del JSON:
campanas publicitarias (campaign_id, advertiser_name, channel, budget_eur, spend_eur, impressions, clicks, conversions, cpm_eur)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en MEDIA_COPILOT.SILVER con datos tipados.


### Modelo Gold: AUDIENCE_360

```
Crea un modelo dbt models/marts/audience_360.sql:
Vista 360 de audiencia: datos basicos + total_events + watch_hours + likes + shares + total_feedback + open_feedback + avg_satisfaction + days_since_last_activity. JOIN de todas las Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: CONTENT_PERFORMANCE

```
Crea un modelo dbt models/marts/content_performance.sql:
Performance de contenido por content_type, genre y dia: total_plays, total_duration_hours, completion_rate, like_rate, share_rate, distinct_audiences, top_platform.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: AUDIENCE_CHURN_RISK

```
Crea un modelo dbt models/marts/audience_churn_risk.sql:
Riesgo de churn por audiencia: subscription_type + days_since_last_activity + feedback_count + avg_watch_hours + engagement_trend + churn_score (0-100).
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en MEDIA_COPILOT.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion de feedback de contenido

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion de feedback de contenido de la tabla MEDIA_COPILOT.SILVER.STG_FEEDBACK.

El LLM debe analizar el campo "description" y devolver un JSON con: tipo_problema (calidad_video|audio|subtitulos|contenido|error_tecnico|cuenta|sugerencia), urgencia (critica|alta|media|baja), equipo_responsable (ingenieria|contenido|soporte|producto|legal), impacto_experiencia (alto|medio|bajo), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de incidencias de una plataforma de streaming."

Crea una vista en el esquema MEDIA_COPILOT.AI llamada VW_FEEDBACK_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de campana

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de campana de la tabla MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: content_type, genre, total_plays, total_duration_hours, completion_rate, like_rate, distinct_audiences

El prompt del LLM debe decir:
"Eres un analista de medios digitales. Genera un resumen ejecutivo en espanol (3-4 frases) del rendimiento de este contenido/campana."

Crea una vista en el esquema MEDIA_COPILOT.AI llamada VW_CAMPAIGN_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de contenido personalizado

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de contenido personalizado de la tabla MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK.

El LLM debe analizar los datos del registro y devolver un JSON con: segmento_audiencia (superfan|regular|casual|dormido), contenido_recomendado (genero + tipo), canal_reengagement (push|email|in_app|home_banner), momento_optimo, mensaje_personalizado

El prompt del LLM debe decir:
"Eres un experto en personalizacion de contenido digital. Analiza este usuario y devuelve SOLO un JSON valido."

Crea una vista en el esquema MEDIA_COPILOT.AI llamada VW_CONTENT_RECOMMENDATIONS.
Limita a 25 registros con filtro churn_score >= 30.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION LIMIT 3;
- SELECT * FROM MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en MEDIA_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en MEDIA_COPILOT.GOLD llamada SV_MEDIA_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_audiences: MEDIA_COPILOT.SILVER.STG_AUDIENCES (PK: audience_id)
- stg_engagement: MEDIA_COPILOT.SILVER.STG_ENGAGEMENT (PK: event_id)
- stg_feedback: MEDIA_COPILOT.SILVER.STG_FEEDBACK (PK: feedback_id)
- stg_campaigns: MEDIA_COPILOT.SILVER.STG_CAMPAIGNS (PK: campaign_id)
- audience_360: MEDIA_COPILOT.GOLD.AUDIENCE_360
- content_performance: MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE
- audience_churn_risk: MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK

Relaciones: todas las tablas se relacionan por audience_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una plataforma de contenido digital y publicidad en Espana.
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT
  COMMENT = 'Agente de media, entertainment y advertising para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Media & Audience Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una plataforma de contenido digital y publicidad en Espana. Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "media_analyst",
          "description": "Consulta datos operativos: Los datos cubren audiencias/suscriptores, eventos de engagement, feedback de contenido y campanas publicitarias. Plataforma de streaming/media en Espana."
        }
      }
    ],
    "tool_resources": {
      "media_analyst": {
        "semantic_view": "MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"Media & Audience Intelligence Copilot"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a MEDIA_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de AUDIENCE_360 con filtros
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
