# 🏦 FSI Risk & Compliance Copilot
**DB:** `FSI_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato FSI_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE FSI_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in FSI_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY FSI_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @FSI_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/fsi/;
```

**[CoCo]** Tabelle Bronze
```
In FSI_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CLIENTS da data/fsi/clients.json
- RAW_TRANSACTIONS da data/fsi/transactions.json
- RAW_RISK_ALERTS da data/fsi/risk_alerts.json
- RAW_ACCOUNTS da data/fsi/accounts.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_CLIENTS; -- atteso: 350
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_TRANSACTIONS; -- atteso: 10,000
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_RISK_ALERTS; -- atteso: 750
SELECT COUNT(*) FROM FSI_COPILOT.BRONZE.RAW_ACCOUNTS; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato fsi_copilot connesso a FSI_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_CLIENTS, RAW_TRANSACTIONS, RAW_RISK_ALERTS, RAW_ACCOUNTS
```

**[CoCo]** Silver: STG_CLIENTS
```
Crea models/staging/stg_clients.sql che esegua il flatten di RAW_CLIENTS.
Campi: clienti finanziari (client_id, company_name, segment, product_type, monthly_volume_eur, risk_profile, kyc_status, aml_score)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_TRANSACTIONS
```
Crea models/staging/stg_transactions.sql che esegua il flatten di RAW_TRANSACTIONS.
Campi: transazioni (transaction_id, client_id, transaction_type, amount_eur, channel, status, country)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_RISK_ALERTS
```
Crea models/staging/stg_risk_alerts.sql che esegua il flatten di RAW_RISK_ALERTS.
Campi: alert di rischio/frode (alert_id, client_id, category, severity, description, rule_triggered, regulatory_report_required)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_ACCOUNTS
```
Crea models/staging/stg_accounts.sql che esegua il flatten di RAW_ACCOUNTS.
Campi: conti e prodotti (account_id, client_id, account_type, balance_eur, interest_rate_pct, status)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_CLIENTS; -- atteso: 350
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_TRANSACTIONS; -- atteso: 10,000
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_RISK_ALERTS; -- atteso: 750
SELECT COUNT(*) FROM FSI_COPILOT.SILVER.STG_ACCOUNTS; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CLIENT_360
```
Crea models/marts/client_360.sql:
Vista 360 del cliente finanziario: dati base + total_accounts + total_balance_eur + total_transactions + avg_transaction_amount + total_alerts + open_alerts + kyc_status + aml_score. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: TRANSACTION_HEALTH
```
Crea models/marts/transaction_health.sql:
Salute transazionale per tipo e giorno: total_transactions, total_volume_eur, rejected_count, avg_amount, distinct_clients, top_channel.
Materializza come table in GOLD.
```

**[CoCo]** Gold: RISK_EXPOSURE
```
Crea models/marts/risk_exposure.sql:
Esposizione al rischio per cliente: monthly_volume_eur + total_alerts_last_90d + critical_alerts + accounts_in_mora + kyc_expired + compliance_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
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

**[CoCo]** Caso 1: Classificazione automatica degli alert di rischio
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella FSI_COPILOT.SILVER.STG_RISK_ALERTS.
Analizza il campo "description" e restituisce un JSON con: tipo_rischio (frode|riciclaggio|compliance|credito|operativo|cyber), urgenza (critica|alta|media|bassa), dipartimento (frode|compliance|rischi|cybersecurity|legale), richiede_segnalazione_regolamentare (si|no), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un analista di rischio di un istituto finanziario."
Crea la vista FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_RISK_ALERT_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo del cliente
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella FSI_COPILOT.GOLD.CLIENT_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: company_name, segment, product_type, monthly_volume_eur, total_accounts, total_balance_eur, total_alerts, aml_score, kyc_status
Prompt di sistema: "Sei un analista di private banking. Genera un riassunto esecutivo in italiano (3-4 frasi) del profilo finanziario di questo cliente."
Crea la vista FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_CLIENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di azione di compliance
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella FSI_COPILOT.GOLD.RISK_EXPOSURE.
Restituisce un JSON con: livello_rischio (alto|medio|basso), azione_principale, azioni_regolamentari (array 2-3), richiede_SAR (si|no), termine_massimo_giorni (numero), raccomandazione_kyc
Prompt di sistema: "Sei un esperto di compliance finanziaria (KYC/AML). Analizza questo cliente e restituisci SOLO un JSON valido."
Filtra WHERE compliance_score >= 40.
Crea la vista FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM FSI_COPILOT.AI.VW_COMPLIANCE_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in FSI_COPILOT.GOLD chiamata SV_FSI_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono clienti finanziari, transazioni, alert di rischio/frode/AML e conti/prodotti. Istituto finanziario in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
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
  COMMENT = 'Copilot di Servizi Finanziari'
  PROFILE = '{"display_name": "FSI Risk & Compliance Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di istituto finanziario in Italia (banca, investimenti, assicurazioni). Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "fsi_analyst",
        "description": "I dati coprono clienti finanziari, transazioni, alert di rischio/frode/AML e conti/prodotti. Istituto finanziario in Italia."
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

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
