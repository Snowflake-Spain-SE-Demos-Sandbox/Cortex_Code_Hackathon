# 🛡️ Insurance Call Center Copilot
**DB:** `INSURANCE_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato INSURANCE_COPILOT con cinque schemi: BRONZE, SILVER, GOLD, AI e ML.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE INSURANCE_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration git_https_api chiamata GIT_HACKATHON_INTEGRATION
e un Git Repository in INSURANCE_COPILOT.PUBLIC chiamato HACKATHON_REPO puntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY INSURANCE_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @INSURANCE_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/insurance/;
```

**[CoCo]** Tabelle Bronze
```
In INSURANCE_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS da data/insurance/customers.json
- RAW_CALLS da data/insurance/call_transcripts.json
- RAW_CLAIMS da data/insurance/claims.json
- RAW_AGENTS da data/insurance/agents.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CUSTOMERS; -- atteso: 350
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CALLS; -- atteso: 500
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_CLAIMS; -- atteso: 1,000
SELECT COUNT(*) FROM INSURANCE_COPILOT.BRONZE.RAW_AGENTS; -- atteso: 50
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Assicurati di essere in **Projects > Workspaces** in Snowsight per eseguire dbt.

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato insurance_copilot connesso a INSURANCE_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_CUSTOMERS, RAW_CALLS, RAW_CLAIMS, RAW_AGENTS.
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Crea models/staging/stg_customers.sql che esegua il flatten di RAW_CUSTOMERS.
Campi: clienti assicurativi (customer_id, full_name, age, region, insurance_types, vehicle_brand, vehicle_model, vehicle_year, hogar_type, auto_premium_eur, hogar_premium_eur, total_premium_eur, years_as_customer, num_policies, num_claims_last_2y, payment_status, renewal_date, nps_score, status)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_CALLS
```
Crea models/staging/stg_calls.sql che esegua il flatten di RAW_CALLS.
Campi: chiamate al call center (call_id, customer_id, agent_id, call_datetime, duration_seconds, insurance_type, call_type, routing_department, transcript TESTO COMPLETO, channel, fcr, csat_score, churn_signal_detected, sentiment_label, topics ARRAY)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_CLAIMS
```
Crea models/staging/stg_claims.sql che esegua il flatten di RAW_CLAIMS.
Campi: sinistri (claim_id, customer_id, insurance_type, claim_type, claim_date, amount_requested_eur, amount_approved_eur, franchise_eur, status, resolution_days, perito_assigned, involves_third_party)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_AGENTS
```
Crea models/staging/stg_agents.sql che esegua il flatten di RAW_AGENTS.
Campi: agenti del call center (agent_id, full_name, team, experience_years, calls_per_day_avg, fcr_rate, avg_handle_time_seconds, qa_score_avg, active)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CUSTOMERS; -- atteso: 350
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CALLS; -- atteso: 500
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_CLAIMS; -- atteso: 1,000
SELECT COUNT(*) FROM INSURANCE_COPILOT.SILVER.STG_AGENTS; -- atteso: 50
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_RISK_360
```
Crea models/marts/customer_risk_360.sql:
Vista 360 del cliente: dati base + total_calls + total_complaints + num_churn_calls + total_claims_eur + avg_csat_score + days_since_last_call + days_to_renewal + fcr_rate. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: CALL_INTELLIGENCE_360
```
Crea models/marts/call_intelligence_360.sql:
Vista per chiamata con cliente + agente: call_id, customer_id, agent_id, call_type, insurance_type, duration_seconds, routing_department, sentiment_label, fcr, csat_score, churn_signal_detected, customer_tenure, customer_premium, agent_qa_score.
Materializza come table in GOLD.
```

**[CoCo]** Gold: AGENT_PERFORMANCE
```
Crea models/marts/agent_performance.sql:
Performance per agente: agent_id, full_name, team, total_calls, avg_duration_seconds, fcr_rate, avg_csat, cancellation_calls, complaint_calls, qa_score_avg, experience_years.
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM INSURANCE_COPILOT.GOLD.CUSTOMER_RISK_360;
SELECT COUNT(*) FROM INSURANCE_COPILOT.GOLD.CALL_INTELLIGENCE_360;
SELECT COUNT(*) FROM INSURANCE_COPILOT.GOLD.AGENT_PERFORMANCE;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Riassunto automatico delle chiamate
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella INSURANCE_COPILOT.SILVER.STG_CALLS.
Analizza il campo "transcript" e restituisce un JSON con: riassunto (3-4 frasi), punti_chiave (array 2-3), esito_chiamata (risolto|in_attesa|escalato|senza_risoluzione), prossima_azione
Prompt di sistema: "Sei un analista di qualita di un call center assicurativo. Riassumi la seguente trascrizione di chiamata."
Crea la vista INSURANCE_COPILOT.AI.VW_CALL_SUMMARY. Limita a 500 record.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CALL_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 2: Classificazione e routing intelligente
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella INSURANCE_COPILOT.SILVER.STG_CALLS.
Analizza il campo "transcript" e restituisce un JSON con: categoria_chiamata (sinistro|consulta|cancellazione|rinnovo|reclamo|assistenza|cambio_dati|fatturazione), sottocategoria, dipartimento_ottimale (sinistri_auto|sinistri_casa|ritenzione|commerciale|assistenza_clienti|assistenza_24h|backoffice|fatturazione|qualita), urgenza (critica|alta|media|bassa), richiede_escalation (si|no), motivo_routing
Prompt di sistema: "Sei un sistema di classificazione chiamate di un call center assicurativo. Classifica la chiamata e determina il routing ottimale."
Crea la vista INSURANCE_COPILOT.AI.VW_CALL_ROUTING. Limita a 500 record.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CALL_ROUTING LIMIT 3;
```

**[CoCo]** Caso 3: QA automatico degli agenti
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella INSURANCE_COPILOT.SILVER.STG_CALLS.
Analizza il campo "transcript" e restituisce un JSON con: score_empatia (0-10), score_risoluzione (0-10), score_protocollo (0-10), score_qa_totale (0-100), punti_forza (array 2), aree_miglioramento (array 2), rispetta_script (si|no), tono_generale (professionale|neutro|migliorabile|inadeguato)
Prompt di sistema: "Sei un valutatore di qualita (QA) esperto in call center assicurativi. Valuta le prestazioni dell'agente nella seguente chiamata in base a empatia, risoluzione del problema, rispetto del protocollo e tono professionale."
Crea la vista INSURANCE_COPILOT.AI.VW_AGENT_QA. Limita a 500 record.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_AGENT_QA LIMIT 3;
```

**[CoCo]** Caso 4: Insight di CX: sentiment e temi
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella INSURANCE_COPILOT.SILVER.STG_CALLS.
Analizza il campo "transcript" e restituisce un JSON con: sentiment_cliente (molto_positivo|positivo|neutro|negativo|molto_negativo), score_soddisfazione (0-10), temi_principali (array 3), menzione_concorrente (si|no), nome_concorrente, parole_chiave (array 5), livello_frustrazione (0-10)
Prompt di sistema: "Sei un analista di customer experience (CX) specializzato in assicurazioni. Analizza sentiment, temi e livello di soddisfazione del cliente nella seguente chiamata."
Crea la vista INSURANCE_COPILOT.AI.VW_CX_INSIGHTS. Limita a 500 record.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CX_INSIGHTS LIMIT 3;
```

**[CoCo]** Caso 5: Rilevamento segnali di churn
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella INSURANCE_COPILOT.SILVER.STG_CALLS.
Analizza il campo "transcript" e restituisce un JSON con: rischio_churn (critico|alto|medio|basso), intenzione_cancellazione (si|no), segnali_rilevati (array con segnali specifici trovati), menzione_prezzo (si|no), menzione_concorrente (si|no), livello_insoddisfazione (0-10), azione_raccomandata (chiamata_ritenzione|offerta_sconto|escalation_esecutiva|follow_up_standard|nessuna_azione)
Prompt di sistema: "Sei uno specialista nella ritenzione dei clienti di una compagnia assicurativa. Rileva i segnali di churn nella seguente trascrizione di chiamata."
Crea la vista INSURANCE_COPILOT.AI.VW_CHURN_SIGNALS. Limita a 500 record.
```

**[✓ SQL]**
```sql
SELECT * FROM INSURANCE_COPILOT.AI.VW_CHURN_SIGNALS LIMIT 3;
```

---

## FASE 4 — Machine Learning: Rilevamento Churn

**[CoCo]**
```
Crea una vista INSURANCE_COPILOT.ML.CHURN_FEATURES che combini GOLD.CUSTOMER_RISK_360
con i segnali di churn dalle viste AI (VW_CX_INSIGHTS + VW_CHURN_SIGNALS).
Campi di input del modello: age, years_as_customer, num_policies, total_premium_eur,
  num_claims_last_2y, total_calls, num_complaint_calls, num_churn_signal_calls,
  avg_csat_score, payment_status_encoded, days_to_renewal, nps_score.
Campo obiettivo (label): will_churn BOOLEAN (derivato da status = 'cancelado'
  o churn_signal_detected nell'ultima chiamata).
Poi allena un modello di classificazione churn usando SNOWFLAKE.ML.CLASSIFICATION.
```

**[SQL]**
```sql
-- 1. Creare lo schema ML e la vista delle features
CREATE SCHEMA IF NOT EXISTS INSURANCE_COPILOT.ML;

CREATE OR REPLACE VIEW INSURANCE_COPILOT.ML.CHURN_FEATURES AS
SELECT
    c.customer_id,
    c.age,
    c.years_as_customer,
    c.num_policies,
    c.total_premium_eur,
    c.num_claims_last_2y,
    r.total_calls,
    r.num_complaint_calls,
    r.num_churn_calls         AS num_churn_signal_calls,
    r.avg_csat_score,
    CASE c.payment_status
        WHEN 'al_dia'      THEN 0
        WHEN 'demora_leve' THEN 1
        WHEN 'impago'      THEN 2 END  AS payment_status_encoded,
    DATEDIFF('day', CURRENT_DATE, c.renewal_date::DATE) AS days_to_renewal,
    COALESCE(c.nps_score, 5)           AS nps_score,
    (c.status = 'cancelado'
     OR r.num_churn_calls > 0)::BOOLEAN AS will_churn
FROM INSURANCE_COPILOT.SILVER.STG_CUSTOMERS c
LEFT JOIN INSURANCE_COPILOT.GOLD.CUSTOMER_RISK_360 r USING (customer_id);

-- 2. Dividir en train (80%) y test (20%)
CREATE OR REPLACE VIEW INSURANCE_COPILOT.ML.CHURN_FEATURES_TRAIN AS
  SELECT * FROM INSURANCE_COPILOT.ML.CHURN_FEATURES
  WHERE MOD(ABS(HASH(customer_id)), 10) < 8;

CREATE OR REPLACE VIEW INSURANCE_COPILOT.ML.CHURN_FEATURES_TEST AS
  SELECT * FROM INSURANCE_COPILOT.ML.CHURN_FEATURES
  WHERE MOD(ABS(HASH(customer_id)), 10) >= 8;

-- 3. Entrenar el modelo de clasificacion
CREATE OR REPLACE SNOWFLAKE.ML.CLASSIFICATION INSURANCE_COPILOT.ML.CHURN_MODEL(
  INPUT_DATA  => SYSTEM$REFERENCE('VIEW', 'INSURANCE_COPILOT.ML.CHURN_FEATURES_TRAIN'),
  TARGET_COLNAME => 'WILL_CHURN'
);

-- 4. Predizioni su tutti i clienti attivi
CREATE OR REPLACE TABLE INSURANCE_COPILOT.ML.CHURN_PREDICTIONS AS
SELECT
    customer_id,
    CHURN_MODEL!PREDICT(OBJECT_CONSTRUCT(*)) AS prediction
FROM INSURANCE_COPILOT.ML.CHURN_FEATURES;

-- 5. Valutazione del modello
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_EVALUATION_METRICS();
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_CONFUSION_MATRIX();
CALL INSURANCE_COPILOT.ML.CHURN_MODEL!SHOW_FEATURE_IMPORTANCE();

-- 6. Clienti con maggior rischio di churn
SELECT
    p.customer_id,
    c.full_name,
    c.total_premium_eur,
    p.prediction:class::BOOLEAN        AS predicted_churn,
    p.prediction:probability:True::FLOAT AS churn_probability
FROM INSURANCE_COPILOT.ML.CHURN_PREDICTIONS p
JOIN INSURANCE_COPILOT.SILVER.STG_CUSTOMERS c USING (customer_id)
ORDER BY churn_probability DESC
LIMIT 20;
```

**[✓ SQL]**
```sql
-- Verificare le predizioni
SELECT COUNT(*), AVG(prediction:probability:True::FLOAT) AS avg_churn_prob
FROM INSURANCE_COPILOT.ML.CHURN_PREDICTIONS
WHERE prediction:class::BOOLEAN = TRUE;
```

---

## FASE 5 — Semantic View

**[CoCo]**
```
Crea una Semantic View in INSURANCE_COPILOT.GOLD chiamata SV_INSURANCE_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto di business: compagnia assicurativa auto e casa in Italia (Seguros Meridian).
Includi dimensioni con commenti in italiano (tipo_polizza, tipo_chiamata, regione,
  tipo_sinistro, sentiment, dipartimento_routing), metriche con sinonimi
(premio_totale, durata_chiamata, score_churn, tasso_fcr, importo_sinistro)
e istruzioni AI_SQL_GENERATION con contesto del call center.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT;
```

---

## FASE 6 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent in Snowflake Intelligence per Seguros Meridian.
- Crea l'oggetto Snowflake Intelligence predefinito se non esiste
- Crea il database SNOWFLAKE_INTELLIGENCE e lo schema AGENTS
- Crea l'agente INSURANCE_COPILOT_AGENT connesso alla Semantic View INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT
  con modello di orchestrazione claude-4-sonnet, rispondendo sempre in italiano
  e contesto: assistente di analisi call center per assicurazioni auto e casa.
- Concedi accesso al ruolo PUBLIC all'agente e alla Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.INSURANCE_COPILOT_AGENT
  COMMENT = 'Insurance Call Center Copilot'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di analisi call center assicurativo. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso. Usa tabelle quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "insurance_analyst",
        "description": "I dati coprono clienti assicurativi (auto e casa), trascrizioni delle chiamate al call center con sentiment e topics, sinistri e performance degli agenti. Compagnia assicurativa Seguros Meridian in Italia."
      }
    }],
    "tool_resources": {
      "insurance_analyst": {
        "semantic_view": "INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.INSURANCE_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW INSURANCE_COPILOT.GOLD.SV_INSURANCE_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.

---

## Prova l'Agente

Domande di esempio per validare l'agente:

- "Quali sono i 10 clienti con maggior rischio di churn secondo il modello ML?"
- "Quali tipi di chiamata hanno il peggior sentiment del cliente questa settimana?"
- "Quali agenti hanno il punteggio QA peggiore nell'ultimo mese?"
- "Quanto ricavo annuale e a rischio per clienti con segnale di cancellazione?"

---

## FASE FINALE — App Streamlit in Snowflake

**[CoCo]**
```
Crea un'app Streamlit in Snowflake (SiS) in INSURANCE_COPILOT, schema AI, chiamata INSURANCE_APP.
L'app si connette con get_active_session() e ha 4 tab:
1. KPI — st.metric: totale chiamate, durata media, tasso FCR, NPS medio, ricavo a rischio
2. Chiamate — st.bar_chart per tipo di chiamata e sentiment. Tabella filtrabile con trascrizioni.
3. Churn Risk — st.dataframe Top 20 clienti per churn_probability (dal modello ML). Rosso se > 0.7.
4. Performance Agenti — tabella con QA score, FCR e CSAT per agente. Bar chart comparativo.
Usa titoli descrittivi in italiano e applica st.set_page_config con tema scuro.
```
