# 📡 Telco Operations Copilot
**DB:** `TELCO_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada TELCO_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE TELCO_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en TELCO_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/;
```

**[CoCo]** Tablas Bronze
```
En TELCO_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS desde data/telco/customers.json
- RAW_NETWORK_EVENTS desde data/telco/network_events.json
- RAW_TICKETS desde data/telco/tickets.json
- RAW_BILLING_USAGE desde data/telco/billing_usage.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_CUSTOMERS; -- esperado: 350
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_NETWORK_EVENTS; -- esperado: 10,000
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_TICKETS; -- esperado: 750
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_BILLING_USAGE; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado telco_copilot conectado a TELCO_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_CUSTOMERS, RAW_NETWORK_EVENTS, RAW_TICKETS, RAW_BILLING_USAGE
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Crea models/staging/stg_customers.sql que haga flatten de RAW_CUSTOMERS.
Campos: clientes telco (customer_id, company_name, plan, region, segment, monthly_revenue_eur, nps_score, status)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_NETWORK_EVENTS
```
Crea models/staging/stg_network_events.sql que haga flatten de RAW_NETWORK_EVENTS.
Campos: eventos de red (event_id, customer_id, event_type, severity, cell_tower_id, duration_ms, metadata)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_TICKETS
```
Crea models/staging/stg_tickets.sql que haga flatten de RAW_TICKETS.
Campos: tickets de soporte (ticket_id, customer_id, category, priority, description, sla_breached, satisfaction_score)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_BILLING_USAGE
```
Crea models/staging/stg_billing_usage.sql que haga flatten de RAW_BILLING_USAGE.
Campos: registros de facturacion (billing_id, customer_id, service_type, amount_eur, payment_status)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_CUSTOMERS; -- esperado: 350
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_NETWORK_EVENTS; -- esperado: 10,000
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_TICKETS; -- esperado: 750
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_BILLING_USAGE; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_360
```
Crea models/marts/customer_360.sql:
Vista 360 de cada cliente: datos basicos + total_tickets + open_tickets + avg_satisfaction + sla_breach_rate + total_network_events + critical_events + total_billed_eur. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: NETWORK_HEALTH
```
Crea models/marts/network_health.sql:
Salud de red por cell_tower_id y dia: total_events, critical_events, high_events, avg_duration_ms, distinct_customers_affected, top_event_type.
Materializa como table en GOLD.
```

**[CoCo]** Gold: REVENUE_AT_RISK
```
Crea models/marts/revenue_at_risk.sql:
Revenue en riesgo por cliente: monthly_revenue_eur + total_tickets_last_90d + sla_breaches_last_90d + unpaid_amount + risk_score (0-100 basado en status, SLA, tickets, impagos, NPS).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.GOLD.CUSTOMER_360;
SELECT COUNT(*) FROM TELCO_COPILOT.GOLD.NETWORK_HEALTH;
SELECT COUNT(*) FROM TELCO_COPILOT.GOLD.REVENUE_AT_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Clasificacion automatica de tickets
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla TELCO_COPILOT.SILVER.STG_TICKETS.
Analiza el campo "description" y devuelve un JSON con: clasificacion_tecnica (conectividad|hardware|software|cobertura|facturacion|comercial), severidad_estimada (critica|alta|media|baja), departamento_responsable (ingenieria_red|soporte_nivel2|facturacion|comercial|campo), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de tickets de soporte de una empresa de telecomunicaciones."
Crea la vista TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de cuenta
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla TELCO_COPILOT.GOLD.CUSTOMER_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: company_name, plan, region, segment, monthly_revenue_eur, total_tickets, open_tickets, avg_satisfaction, sla_breach_rate, critical_events, total_billed_eur
Prompt del sistema: "Eres un analista de cuentas de una empresa de telecomunicaciones. Genera un resumen ejecutivo en espanol (3-4 frases) de este cliente."
Crea la vista TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de acciones de retencion
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla TELCO_COPILOT.GOLD.REVENUE_AT_RISK.
Devuelve un JSON con: nivel_urgencia (alta|media|baja), accion_principal, acciones_complementarias (array 2-3), oferta_sugerida, canal_contacto (telefono|email|visita_presencial), mensaje_personalizado
Prompt del sistema: "Eres un experto en retencion de clientes de telecomunicaciones. Analiza este cliente en riesgo y devuelve SOLO un JSON valido."
Filtra WHERE risk_score >= 30.
Crea la vista TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en TELCO_COPILOT.GOLD llamada SV_TELCO_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren clientes, tickets de soporte, eventos de red (incidencias tecnicas) y facturacion. Las regiones son ciudades espanolas.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT
  COMMENT = 'Copilot de Telecomunicaciones'
  PROFILE = '{"display_name": "Telco Operations Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de empresa de telecomunicaciones en Espana. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "telco_analyst",
        "description": "Los datos cubren clientes, tickets de soporte, eventos de red (incidencias tecnicas) y facturacion. Las regiones son ciudades espanolas."
      }
    }],
    "tool_resources": {
      "telco_analyst": {
        "semantic_view": "TELCO_COPILOT.GOLD.SV_TELCO_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.
