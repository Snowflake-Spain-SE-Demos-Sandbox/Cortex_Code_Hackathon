# 📺 Media & Audience Intelligence Copilot
**DB:** `MEDIA_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada MEDIA_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE MEDIA_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en MEDIA_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY MEDIA_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/;
```

**[CoCo]** Tablas Bronze
```
En MEDIA_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_AUDIENCES desde data/media/audiences.json
- RAW_ENGAGEMENT desde data/media/engagement_events.json
- RAW_FEEDBACK desde data/media/content_feedback.json
- RAW_CAMPAIGNS desde data/media/campaigns.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_AUDIENCES; -- esperado: 350
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_ENGAGEMENT; -- esperado: 10,000
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_FEEDBACK; -- esperado: 750
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_CAMPAIGNS; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado media_copilot conectado a MEDIA_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_AUDIENCES, RAW_ENGAGEMENT, RAW_FEEDBACK, RAW_CAMPAIGNS
```

**[CoCo]** Silver: STG_AUDIENCES
```
Crea models/staging/stg_audiences.sql que haga flatten de RAW_AUDIENCES.
Campos: perfiles de audiencia (audience_id, username, region, subscription_type, preferred_genre, age_group, watch_hours_monthly, status)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_ENGAGEMENT
```
Crea models/staging/stg_engagement.sql que haga flatten de RAW_ENGAGEMENT.
Campos: eventos de engagement (event_id, audience_id, event_type, content_id, content_type, genre, duration_seconds, platform)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_FEEDBACK
```
Crea models/staging/stg_feedback.sql que haga flatten de RAW_FEEDBACK.
Campos: feedback de contenido (feedback_id, audience_id, category, priority, description, sla_breached, satisfaction_score)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_CAMPAIGNS
```
Crea models/staging/stg_campaigns.sql que haga flatten de RAW_CAMPAIGNS.
Campos: campanas publicitarias (campaign_id, advertiser_name, channel, budget_eur, spend_eur, impressions, clicks, conversions, cpm_eur)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_AUDIENCES; -- esperado: 350
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_ENGAGEMENT; -- esperado: 10,000
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_FEEDBACK; -- esperado: 750
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_CAMPAIGNS; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: AUDIENCE_360
```
Crea models/marts/audience_360.sql:
Vista 360 de audiencia: datos basicos + total_events + watch_hours + likes + shares + total_feedback + open_feedback + avg_satisfaction + days_since_last_activity. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: CONTENT_PERFORMANCE
```
Crea models/marts/content_performance.sql:
Performance de contenido por content_type, genre y dia: total_plays, total_duration_hours, completion_rate, like_rate, share_rate, distinct_audiences, top_platform.
Materializa como table en GOLD.
```

**[CoCo]** Gold: AUDIENCE_CHURN_RISK
```
Crea models/marts/audience_churn_risk.sql:
Riesgo de churn por audiencia: subscription_type + days_since_last_activity + feedback_count + avg_watch_hours + engagement_trend + churn_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.GOLD.AUDIENCE_360;
SELECT COUNT(*) FROM MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE;
SELECT COUNT(*) FROM MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Clasificacion de feedback de contenido
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla MEDIA_COPILOT.SILVER.STG_FEEDBACK.
Analiza el campo "description" y devuelve un JSON con: tipo_problema (calidad_video|audio|subtitulos|contenido|error_tecnico|cuenta|sugerencia), urgencia (critica|alta|media|baja), equipo_responsable (ingenieria|contenido|soporte|producto|legal), impacto_experiencia (alto|medio|bajo), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de incidencias de una plataforma de streaming."
Crea la vista MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de campana
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: content_type, genre, total_plays, total_duration_hours, completion_rate, like_rate, distinct_audiences
Prompt del sistema: "Eres un analista de medios digitales. Genera un resumen ejecutivo en espanol (3-4 frases) del rendimiento de este contenido/campana."
Crea la vista MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de contenido personalizado
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK.
Devuelve un JSON con: segmento_audiencia (superfan|regular|casual|dormido), contenido_recomendado (genero + tipo), canal_reengagement (push|email|in_app|home_banner), momento_optimo, mensaje_personalizado
Prompt del sistema: "Eres un experto en personalizacion de contenido digital. Analiza este usuario y devuelve SOLO un JSON valido."
Filtra WHERE churn_score >= 30.
Crea la vista MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en MEDIA_COPILOT.GOLD llamada SV_MEDIA_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren audiencias/suscriptores, eventos de engagement, feedback de contenido y campanas publicitarias. Plataforma de streaming/media en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT
  COMMENT = 'Copilot de Media, Entertainment y Advertising'
  PROFILE = '{"display_name": "Media & Audience Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de plataforma de contenido digital y publicidad en Espana. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "media_analyst",
        "description": "Los datos cubren audiencias/suscriptores, eventos de engagement, feedback de contenido y campanas publicitarias. Plataforma de streaming/media en Espana."
      }
    }],
    "tool_resources": {
      "media_analyst": {
        "semantic_view": "MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.
