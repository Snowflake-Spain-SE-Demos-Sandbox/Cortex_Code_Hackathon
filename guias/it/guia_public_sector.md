# 🌐 Smart City Copilot
**DB:** `PUBLIC_SECTOR_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato PUBLIC_SECTOR_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE PUBLIC_SECTOR_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in PUBLIC_SECTOR_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @PUBLIC_SECTOR_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/public_sector/;
```

**[CoCo]** Tabelle Bronze
```
In PUBLIC_SECTOR_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CITIZENS da data/public_sector/citizens.json
- RAW_SERVICE_REQUESTS da data/public_sector/service_requests.json
- RAW_INSPECTIONS da data/public_sector/inspection_reports.json
- RAW_BUDGET da data/public_sector/budget_items.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_CITIZENS; -- atteso: 350
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_SERVICE_REQUESTS; -- atteso: 10,000
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_INSPECTIONS; -- atteso: 750
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.BRONZE.RAW_BUDGET; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato public_sector_copilot connesso a PUBLIC_SECTOR_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_CITIZENS, RAW_SERVICE_REQUESTS, RAW_INSPECTIONS, RAW_BUDGET
```

**[CoCo]** Silver: STG_CITIZENS
```
Crea models/staging/stg_citizens.sql che esegua il flatten di RAW_CITIZENS.
Campi: cittadini (citizen_id, full_name, age, district, municipality, profile_type, preferred_channel, digital_literacy)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_SERVICE_REQUESTS
```
Crea models/staging/stg_service_requests.sql che esegua il flatten di RAW_SERVICE_REQUESTS.
Campi: richieste di servizio (request_id, citizen_id, request_type, department, channel, status, priority, response_days)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_INSPECTIONS
```
Crea models/staging/stg_inspections.sql che esegua il flatten di RAW_INSPECTIONS.
Campi: relazioni di ispezione (report_id, inspector_id, category, severity, description, fine_amount_eur, follow_up_required)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_BUDGET
```
Crea models/staging/stg_budget.sql che esegua il flatten di RAW_BUDGET.
Campi: voci di bilancio (item_id, department, fiscal_year, budget_line, approved_amount_eur, executed_amount_eur, execution_pct)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_CITIZENS; -- atteso: 350
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_SERVICE_REQUESTS; -- atteso: 10,000
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS; -- atteso: 750
SELECT COUNT(*) FROM PUBLIC_SECTOR_COPILOT.SILVER.STG_BUDGET; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CITIZEN_360
```
Crea models/marts/citizen_360.sql:
Vista 360 del cittadino: dati base + total_requests + resolved_requests + avg_response_days + total_complaints + digital_channel_usage_pct + days_since_last_interaction. JOIN di Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: SERVICE_HEALTH
```
Crea models/marts/service_health.sql:
Salute dei servizi per department e periodo: total_requests, resolved_count, avg_response_days, overdue_count, citizen_satisfaction_proxy, top_request_type.
Materializza come table in GOLD.
```

**[CoCo]** Gold: BUDGET_EXECUTION
```
Crea models/marts/budget_execution.sql:
Esecuzione del bilancio per department e trimestre: approved_total_eur, executed_total_eur, execution_pct, pending_eur, overbudget_items, underexecuted_items.
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
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

**[CoCo]** Caso 1: Classificazione dei rapporti di ispezione
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella PUBLIC_SECTOR_COPILOT.SILVER.STG_INSPECTIONS.
Analizza il campo "description" e restituisce un JSON con: ambito (urbanistica|sanita|ambiente|sicurezza_lavoro|accessibilita|attivita|opera_pubblica|rumore), gravita (lieve|grave|molto_grave), organo_competente (urbanistica|sanita|ambiente|polizia_locale|vigili_fuoco|tribunale), richiede_sanzione (si|no), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione delle ispezioni di una pubblica amministrazione."
Crea la vista PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_INSPECTION_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo del servizio pubblico
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: department, total_requests, resolved_count, avg_response_days, overdue_count, top_request_type
Prompt di sistema: "Sei un analista di gestione pubblica. Genera un riassunto esecutivo in italiano (3-4 frasi) dello stato di questo servizio/dipartimento."
Crea la vista PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di miglioramento del servizio
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella PUBLIC_SECTOR_COPILOT.GOLD.SERVICE_HEALTH.
Restituisce un JSON con: livello_efficienza (buono|migliorabile|deficiente), azione_principale (digitalizzazione|formazione|riallocazione_risorse|semplificazione_processo), aree_miglioramento (array 2-3), risparmio_stimato_pct, termine_implementazione_mesi, impatto_cittadino (alto|medio|basso)
Prompt di sistema: "Sei un consulente di trasformazione digitale del settore pubblico. Analizza questo dipartimento e restituisci SOLO un JSON valido."
Filtra WHERE avg_response_days > 15.
Crea la vista PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM PUBLIC_SECTOR_COPILOT.AI.VW_SERVICE_IMPROVEMENT LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in PUBLIC_SECTOR_COPILOT.GOLD chiamata SV_PUBLIC_SECTOR_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono cittadini, richieste di servizio, relazioni di ispezione e bilanci. Pubblica amministrazione / comune in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
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
  COMMENT = 'Copilot di Settore Pubblico'
  PROFILE = '{"display_name": "Smart City Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di pubblica amministrazione / comune in Italia. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "public_sector_analyst",
        "description": "I dati coprono cittadini, richieste di servizio, relazioni di ispezione e bilanci. Pubblica amministrazione / comune in Italia."
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

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
