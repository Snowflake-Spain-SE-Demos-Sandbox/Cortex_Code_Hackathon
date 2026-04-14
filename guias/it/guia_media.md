# 📺 Media & Audience Intelligence Copilot
**DB:** `MEDIA_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato MEDIA_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE MEDIA_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in MEDIA_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY MEDIA_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/;
```

**[CoCo]** Tabelle Bronze
```
In MEDIA_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_AUDIENCES da data/media/audiences.json
- RAW_ENGAGEMENT da data/media/engagement_events.json
- RAW_FEEDBACK da data/media/content_feedback.json
- RAW_CAMPAIGNS da data/media/campaigns.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_AUDIENCES; -- atteso: 350
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_ENGAGEMENT; -- atteso: 10,000
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_FEEDBACK; -- atteso: 750
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_CAMPAIGNS; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Assicurati di essere in **Projects > Workspaces** in Snowsight per eseguire dbt.

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato media_copilot connesso a MEDIA_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_AUDIENCES, RAW_ENGAGEMENT, RAW_FEEDBACK, RAW_CAMPAIGNS
```

**[CoCo]** Silver: STG_AUDIENCES
```
Crea models/staging/stg_audiences.sql che esegua il flatten di RAW_AUDIENCES.
Campi: profili di audience (audience_id, username, region, subscription_type, preferred_genre, age_group, watch_hours_monthly, status)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_ENGAGEMENT
```
Crea models/staging/stg_engagement.sql che esegua il flatten di RAW_ENGAGEMENT.
Campi: eventi di engagement (event_id, audience_id, event_type, content_id, content_type, genre, duration_seconds, platform)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_FEEDBACK
```
Crea models/staging/stg_feedback.sql che esegua il flatten di RAW_FEEDBACK.
Campi: feedback sui contenuti (feedback_id, audience_id, category, priority, description, sla_breached, satisfaction_score)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_CAMPAIGNS
```
Crea models/staging/stg_campaigns.sql che esegua il flatten di RAW_CAMPAIGNS.
Campi: campagne pubblicitarie (campaign_id, advertiser_name, channel, budget_eur, spend_eur, impressions, clicks, conversions, cpm_eur)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_AUDIENCES; -- atteso: 350
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_ENGAGEMENT; -- atteso: 10,000
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_FEEDBACK; -- atteso: 750
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_CAMPAIGNS; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: AUDIENCE_360
```
Crea models/marts/audience_360.sql:
Vista 360 dell'audience: dati base + total_events + watch_hours + likes + shares + total_feedback + open_feedback + avg_satisfaction + days_since_last_activity. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: CONTENT_PERFORMANCE
```
Crea models/marts/content_performance.sql:
Performance dei contenuti per content_type, genre e giorno: total_plays, total_duration_hours, completion_rate, like_rate, share_rate, distinct_audiences, top_platform.
Materializza come table in GOLD.
```

**[CoCo]** Gold: AUDIENCE_CHURN_RISK
```
Crea models/marts/audience_churn_risk.sql:
Rischio di churn per audience: subscription_type + days_since_last_activity + feedback_count + avg_watch_hours + engagement_trend + churn_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
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

**[CoCo]** Caso 1: Classificazione del feedback sui contenuti
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella MEDIA_COPILOT.SILVER.STG_FEEDBACK.
Analizza il campo "description" e restituisce un JSON con: tipo_problema (qualita_video|audio|sottotitoli|contenuto|errore_tecnico|account|suggerimento), urgenza (critica|alta|media|bassa), team_responsabile (ingegneria|contenuti|supporto|prodotto|legale), impatto_esperienza (alto|medio|basso), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione degli incidenti di una piattaforma di streaming."
Crea la vista MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo della campagna
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: content_type, genre, total_plays, total_duration_hours, completion_rate, like_rate, distinct_audiences
Prompt di sistema: "Sei un analista di media digitali. Genera un riassunto esecutivo in italiano (3-4 frasi) delle performance di questo contenuto/campagna."
Crea la vista MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di contenuto personalizzato
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK.
Restituisce un JSON con: segmento_audience (superfan|regolare|casuale|dormiente), contenuto_raccomandato (genere + tipo), canale_reengagement (push|email|in_app|home_banner), momento_ottimale, messaggio_personalizzato
Prompt di sistema: "Sei un esperto di personalizzazione di contenuti digitali. Analizza questo utente e restituisci SOLO un JSON valido."
Filtra WHERE churn_score >= 30.
Crea la vista MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in MEDIA_COPILOT.GOLD chiamata SV_MEDIA_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono audience/abbonati, eventi di engagement, feedback sui contenuti e campagne pubblicitarie. Piattaforma streaming/media in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent in Snowflake Intelligence per Media, Entertainment e Advertising.
- Crea l'oggetto Snowflake Intelligence predefinito se non esiste
- Crea il database SNOWFLAKE_INTELLIGENCE e lo schema AGENTS
- Crea l'agente MEDIA_COPILOT_AGENT connesso alla Semantic View MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT
  con modello di orchestrazione claude-4-sonnet, rispondendo sempre in italiano
  e contesto di business: piattaforma di contenuti digitali e pubblicita in Italia
- Concedi accesso al ruolo PUBLIC all'agente e alla Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT
  COMMENT = 'Copilot di Media, Entertainment e Advertising'
  PROFILE = '{"display_name": "Media & Audience Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di piattaforma di contenuti digitali e pubblicita in Italia. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "media_analyst",
        "description": "I dati coprono audience/abbonati, eventi di engagement, feedback sui contenuti e campagne pubblicitarie. Piattaforma streaming/media in Italia."
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

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.

---

## Prova l'Agente

Domande di esempio per validare l'agente:

- "Quali utenti sono inattivi da piu di 30 giorni?"
- "Quali generi di contenuto hanno il tasso di completion piu alto?"
- "Quali campagne pubblicitarie hanno il CTR piu alto questo mese?"
- "Quanti abbonati sono a rischio di cancellazione?"

---

## FASE FINALE — App Streamlit in Snowflake

**[CoCo]**
```
Crea un'app Streamlit in Snowflake (SiS) nel database MEDIA_COPILOT, schema AI, chiamata MEDIA_APP.
L'app deve connettersi a Snowflake con la sessione attiva (get_active_session).
Includi 4 sezioni usando st.tabs:
1. KPI — card con st.metric con i totali e i ratio principali di AUDIENCE_360
2. Distribuzione — grafico a barre (st.bar_chart) per categoria/regione da CONTENT_PERFORMANCE
3. Top 20 — tabella interattiva (st.dataframe) ordinata per score di rischio da AUDIENCE_CHURN_RISK
4. Tendenza — grafico a linee (st.line_chart) con l'evoluzione temporale degli eventi principali
Usa titoli ed etichette descrittivi in italiano. Applica un tema scuro con st.set_page_config.
```
