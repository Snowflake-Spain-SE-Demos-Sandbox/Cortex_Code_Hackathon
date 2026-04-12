# 🏦 FSI Risk & Compliance Copilot
**DB:** `FSI_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada FSI_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE FSI_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en FSI_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY FSI_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/;
```

**[CoCo]** Tablas Bronze
```
En FSI_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CLIENTS desde data/fsi/clients.json
- RAW_TRANSACTIONS desde data/fsi/transactions.json
- RAW_RISK_ALERTS desde data/fsi/risk_alerts.json
- RAW_ACCOUNTS desde data/fsi/accounts.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_CLIENTS; -- esperado: 350
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_TRANSACTIONS; -- esperado: 10,000
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_RISK_ALERTS; -- esperado: 750
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_ACCOUNTS; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado fsi_copilot conectado a FSI_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_CLIENTS, RAW_TRANSACTIONS, RAW_RISK_ALERTS, RAW_ACCOUNTS
```

**[CoCo]** Silver: STG_CLIENTS
```
Crea models/staging/stg_clients.sql que haga flatten de RAW_CLIENTS.
Campos: clientes financieros (client_id, company_name, segment, product_type, monthly_volume_eur, risk_profile, kyc_status, aml_score)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_TRANSACTIONS
```
Crea models/staging/stg_transactions.sql que haga flatten de RAW_TRANSACTIONS.
Campos: transacciones (transaction_id, client_id, transaction_type, amount_eur, channel, status, country)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_RISK_ALERTS
```
Crea models/staging/stg_risk_alerts.sql que haga flatten de RAW_RISK_ALERTS.
Campos: alertas de riesgo/fraude (alert_id, client_id, category, severity, description, rule_triggered, regulatory_report_required)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_ACCOUNTS
```
Crea models/staging/stg_accounts.sql que haga flatten de RAW_ACCOUNTS.
Campos: cuentas y productos (account_id, client_id, account_type, balance_eur, interest_rate_pct, status)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_CLIENTS; -- esperado: 350
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_TRANSACTIONS; -- esperado: 10,000
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_RISK_ALERTS; -- esperado: 750
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_ACCOUNTS; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CLIENT_360
```
Crea models/marts/client_360.sql:
Vista 360 del cliente financiero: datos basicos + total_accounts + total_balance_eur + total_transactions + avg_transaction_amount + total_alerts + open_alerts + kyc_status + aml_score. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: TRANSACTION_HEALTH
```
Crea models/marts/transaction_health.sql:
Salud transaccional por tipo y dia: total_transactions, total_volume_eur, rejected_count, avg_amount, distinct_clients, top_channel.
Materializa como table en GOLD.
```

**[CoCo]** Gold: RISK_EXPOSURE
```
Crea models/marts/risk_exposure.sql:
Exposicion al riesgo por cliente: monthly_volume_eur + total_alerts_last_90d + critical_alerts + accounts_in_mora + kyc_expired + compliance_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.GOLD.CLIENT_360;
SELECT COUNT(*) FROM FSI_COPILOT.GOLD.TRANSACTION_HEALTH;
SELECT COUNT(*) FROM FSI_COPILOT.GOLD.RISK_EXPOSURE;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Clasificacion automatica de alertas de riesgo
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla FSI_COPILOT.SILVER.STG_RISK_ALERTS.
Analiza el campo "description" y devuelve un JSON con: tipo_riesgo (fraude|blanqueo|compliance|credito|operacional|ciber), urgencia (critica|alta|media|baja), departamento (fraude|compliance|riesgos|ciberseguridad|legal), requiere_reporte_regulatorio (si|no), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un analista de riesgos de una entidad financiera."
Crea la vista FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de cliente
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla FSI_COPILOT.GOLD.CLIENT_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: company_name, segment, product_type, monthly_volume_eur, total_accounts, total_balance_eur, total_alerts, aml_score, kyc_status
Prompt del sistema: "Eres un analista de banca privada. Genera un resumen ejecutivo en espanol (3-4 frases) del perfil financiero de este cliente."
Crea la vista FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de accion de compliance
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla FSI_COPILOT.GOLD.RISK_EXPOSURE.
Devuelve un JSON con: nivel_riesgo (alto|medio|bajo), accion_principal, acciones_regulatorias (array 2-3), requiere_SAR (si|no), plazo_maximo_dias (numero), recomendacion_kyc
Prompt del sistema: "Eres un experto en compliance financiero (KYC/AML). Analiza este cliente y devuelve SOLO un JSON valido."
Filtra WHERE compliance_score >= 40.
Crea la vista FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en FSI_COPILOT.GOLD llamada SV_FSI_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren clientes financieros, transacciones, alertas de riesgo/fraude/AML y cuentas/productos. Entidad financiera en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW FSI_COPILOT.GOLD.SV_FSI_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.FSI_COPILOT_AGENT
  COMMENT = 'Copilot de Servicios Financieros'
  PROFILE = '{"display_name": "FSI Risk & Compliance Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de entidad financiera en Espana (banca, inversiones, seguros). Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "fsi_analyst",
        "description": "Los datos cubren clientes financieros, transacciones, alertas de riesgo/fraude/AML y cuentas/productos. Entidad financiera en Espana."
      }
    }],
    "tool_resources": {
      "fsi_analyst": {
        "semantic_view": "FSI_COPILOT.GOLD.SV_FSI_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.FSI_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW FSI_COPILOT.GOLD.SV_FSI_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.
