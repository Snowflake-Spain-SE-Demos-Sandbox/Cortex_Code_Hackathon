# 🌐 Smart City Copilot
**DB:** `PUBLIC_SECTOR_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada PUBLIC_SECTOR_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE PUBLIC_SECTOR_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en PUBLIC_SECTOR_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/;
```

**[CoCo]** Tablas Bronze
```
En PUBLIC_SECTOR_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CITIZENS desde data/public_sector/citizens.json
- RAW_SERVICE_REQUESTS desde data/public_sector/service_requests.json
- RAW_INSPECTIONS desde data/public_sector/inspection_reports.json
- RAW_BUDGET desde data/public_sector/budget_items.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_CITIZENS; -- esperado: 350
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_SERVICE_REQUESTS; -- esperado: 10,000
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_INSPECTIONS; -- esperado: 750
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_BUDGET; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado public_sector_copilot conectado a PUBLIC_SECTOR_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_CITIZENS, RAW_SERVICE_REQUESTS, RAW_INSPECTIONS, RAW_BUDGET
```

**[CoCo]** Silver: STG_CITIZENS
```
Crea models/staging/stg_citizens.sql que haga flatten de RAW_CITIZENS.
Campos: ciudadanos (citizen_id, full_name, age, district, municipality, profile_type, preferred_channel, digital_literacy)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_SERVICE_REQUESTS
```
Crea models/staging/stg_service_requests.sql que haga flatten de RAW_SERVICE_REQUESTS.
Campos: solicitudes de servicio (request_id, citizen_id, request_type, department, channel, status, priority, response_days)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_INSPECTIONS
```
Crea models/staging/stg_inspections.sql que haga flatten de RAW_INSPECTIONS.
Campos: informes de inspeccion (report_id, inspector_id, category, severity, description, fine_amount_eur, follow_up_required)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_BUDGET
```
Crea models/staging/stg_budget.sql que haga flatten de RAW_BUDGET.
Campos: partidas presupuestarias (item_id, department, fiscal_year, budget_line, approved_amount_eur, executed_amount_eur, execution_pct)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_CITIZENS; -- esperado: 350
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_SERVICE_REQUESTS; -- esperado: 10,000
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS; -- esperado: 750
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_BUDGET; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CITIZEN_360
```
Crea models/marts/citizen_360.sql:
Vista 360 del ciudadano: datos basicos + total_requests + resolved_requests + avg_response_days + total_complaints + digital_channel_usage_pct + days_since_last_interaction. JOIN de Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: SERVICE_HEALTH
```
Crea models/marts/service_health.sql:
Salud de servicios por department y periodo: total_requests, resolved_count, avg_response_days, overdue_count, citizen_satisfaction_proxy, top_request_type.
Materializa como table en GOLD.
```

**[CoCo]** Gold: BUDGET_EXECUTION
```
Crea models/marts/budget_execution.sql:
Ejecucion presupuestaria por department y quarter: approved_total_eur, executed_total_eur, execution_pct, pending_eur, overbudget_items, underexecuted_items.
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.GOLD.CITIZEN_360;
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH;
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.GOLD.BUDGET_EXECUTION;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Clasificacion de informes de inspeccion
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS.
Analiza el campo "description" y devuelve un JSON con: ambito (urbanismo|sanidad|medio_ambiente|seguridad_laboral|accesibilidad|actividades|obra_publica|ruido), gravedad (leve|grave|muy_grave), organo_competente (urbanismo|sanidad|medio_ambiente|policia_local|bomberos|juzgado), requiere_sancion (si|no), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de inspecciones de una administracion publica."
Crea la vista PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de servicio publico
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: department, total_requests, resolved_count, avg_response_days, overdue_count, top_request_type
Prompt del sistema: "Eres un analista de gestion publica. Genera un resumen ejecutivo en espanol (3-4 frases) del estado de este servicio/departamento."
Crea la vista PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de mejora de servicio
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.
Devuelve un JSON con: nivel_eficiencia (bueno|mejorable|deficiente), accion_principal (digitalizacion|formacion|reasignacion_recursos|simplificacion_proceso), areas_mejora (array 2-3), ahorro_estimado_pct, plazo_implementacion_meses, impacto_ciudadano (alto|medio|bajo)
Prompt del sistema: "Eres un consultor de transformacion digital del sector publico. Analiza este departamento y devuelve SOLO un JSON valido."
Filtra WHERE avg_response_days > 15.
Crea la vista PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en PUBLIC_SECTOR_COPILOT.GOLD llamada SV_PUBLIC_SECTOR_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren ciudadanos, solicitudes de servicio, informes de inspeccion y presupuestos. Administracion publica / ayuntamiento en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.PUBLIC_SECTOR_COPILOT_AGENT
  COMMENT = 'Copilot de Sector Publico'
  PROFILE = '{"display_name": "Smart City Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de administracion publica / ayuntamiento en Espana. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "public_sector_analyst",
        "description": "Los datos cubren ciudadanos, solicitudes de servicio, informes de inspeccion y presupuestos. Administracion publica / ayuntamiento en Espana."
      }
    }],
    "tool_resources": {
      "public_sector_analyst": {
        "semantic_view": "PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.PUBLIC_SECTOR_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW PUBLIC_SECTOR_COPILOT.GOLD.SV_PUBLIC_SECTOR_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.
