# 📡 Telco Operations Copilot
**DB:** `TELCO_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato TELCO_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE TELCO_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in TELCO_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/telco/;
```

**[CoCo]** Tabelle Bronze
```
In TELCO_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS da data/telco/customers.json
- RAW_NETWORK_EVENTS da data/telco/network_events.json
- RAW_TICKETS da data/telco/tickets.json
- RAW_BILLING_USAGE da data/telco/billing_usage.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_CUSTOMERS; -- atteso: 350
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_NETWORK_EVENTS; -- atteso: 10,000
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_TICKETS; -- atteso: 750
SELECT COUNT(*) FROM TELCO_COPILOT.BRONZE.RAW_BILLING_USAGE; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato telco_copilot connesso a TELCO_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_CUSTOMERS, RAW_NETWORK_EVENTS, RAW_TICKETS, RAW_BILLING_USAGE
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Crea models/staging/stg_customers.sql che esegua il flatten di RAW_CUSTOMERS.
Campi: clienti telco (customer_id, company_name, plan, region, segment, monthly_revenue_eur, nps_score, status)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_NETWORK_EVENTS
```
Crea models/staging/stg_network_events.sql che esegua il flatten di RAW_NETWORK_EVENTS.
Campi: eventi di rete (event_id, customer_id, event_type, severity, cell_tower_id, duration_ms, metadata)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_TICKETS
```
Crea models/staging/stg_tickets.sql che esegua il flatten di RAW_TICKETS.
Campi: ticket di supporto (ticket_id, customer_id, category, priority, description, sla_breached, satisfaction_score)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_BILLING_USAGE
```
Crea models/staging/stg_billing_usage.sql che esegua il flatten di RAW_BILLING_USAGE.
Campi: record di fatturazione (billing_id, customer_id, service_type, amount_eur, payment_status)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_CUSTOMERS; -- atteso: 350
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_NETWORK_EVENTS; -- atteso: 10,000
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_TICKETS; -- atteso: 750
SELECT COUNT(*) FROM TELCO_COPILOT.SILVER.STG_BILLING_USAGE; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_360
```
Crea models/marts/customer_360.sql:
Vista 360 di ogni cliente: dati base + total_tickets + open_tickets + avg_satisfaction + sla_breach_rate + total_network_events + critical_events + total_billed_eur. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: NETWORK_HEALTH
```
Crea models/marts/network_health.sql:
Salute della rete per cell_tower_id e giorno: total_events, critical_events, high_events, avg_duration_ms, distinct_customers_affected, top_event_type.
Materializza come table in GOLD.
```

**[CoCo]** Gold: REVENUE_AT_RISK
```
Crea models/marts/revenue_at_risk.sql:
Ricavo a rischio per cliente: monthly_revenue_eur + total_tickets_last_90d + sla_breaches_last_90d + unpaid_amount + risk_score (0-100 basato su status, SLA, ticket, insoluti, NPS).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
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

**[CoCo]** Caso 1: Classificazione automatica dei ticket
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella TELCO_COPILOT.SILVER.STG_TICKETS.
Analizza il campo "description" e restituisce un JSON con: classificazione_tecnica (connettivita|hardware|software|copertura|fatturazione|commerciale), severita_stimata (critica|alta|media|bassa), dipartimento_responsabile (ingegneria_rete|supporto_livello2|fatturazione|commerciale|campo), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione dei ticket di supporto di un'azienda di telecomunicazioni."
Crea la vista TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo dell'account
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella TELCO_COPILOT.GOLD.CUSTOMER_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: company_name, plan, region, segment, monthly_revenue_eur, total_tickets, open_tickets, avg_satisfaction, sla_breach_rate, critical_events, total_billed_eur
Prompt di sistema: "Sei un analista di account di un'azienda di telecomunicazioni. Genera un riassunto esecutivo in italiano (3-4 frasi) di questo cliente."
Crea la vista TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di azioni di fidelizzazione
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella TELCO_COPILOT.GOLD.REVENUE_AT_RISK.
Restituisce un JSON con: livello_urgenza (alta|media|bassa), azione_principale, azioni_complementari (array 2-3), offerta_suggerita, canale_contatto (telefono|email|visita_personale), messaggio_personalizzato
Prompt di sistema: "Sei un esperto di fidelizzazione clienti nelle telecomunicazioni. Analizza questo cliente a rischio e restituisci SOLO un JSON valido."
Filtra WHERE risk_score >= 30.
Crea la vista TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in TELCO_COPILOT.GOLD chiamata SV_TELCO_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono clienti, ticket di supporto, eventi di rete (incidenti tecnici) e fatturazione. Le regioni sono citta italiane.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
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
  COMMENT = 'Copilot di Telecomunicazioni'
  PROFILE = '{"display_name": "Telco Operations Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di azienda di telecomunicazioni in Italia. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "telco_analyst",
        "description": "I dati coprono clienti, ticket di supporto, eventi di rete (incidenti tecnici) e fatturazione. Le regioni sono citta italiane."
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

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
