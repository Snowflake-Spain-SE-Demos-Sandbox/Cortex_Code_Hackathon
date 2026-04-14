# 🏭 Smart Factory Copilot
**DB:** `MANUFACTURING_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato MANUFACTURING_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE MANUFACTURING_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in MANUFACTURING_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MANUFACTURING_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/manufacturing/;
```

**[CoCo]** Tabelle Bronze
```
In MANUFACTURING_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_EQUIPMENT da data/manufacturing/equipment.json
- RAW_SENSOR_READINGS da data/manufacturing/sensor_readings.json
- RAW_WORK_ORDERS da data/manufacturing/work_orders.json
- RAW_PRODUCTION_BATCHES da data/manufacturing/production_batches.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_EQUIPMENT; -- atteso: 350
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_SENSOR_READINGS; -- atteso: 10,000
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_WORK_ORDERS; -- atteso: 750
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.BRONZE.RAW_PRODUCTION_BATCHES; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Assicurati di essere in **Projects > Workspaces** in Snowsight per eseguire dbt.

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato manufacturing_copilot connesso a MANUFACTURING_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_EQUIPMENT, RAW_SENSOR_READINGS, RAW_WORK_ORDERS, RAW_PRODUCTION_BATCHES
```

**[CoCo]** Silver: STG_EQUIPMENT
```
Crea models/staging/stg_equipment.sql che esegua il flatten di RAW_EQUIPMENT.
Campi: attrezzature/macchine (equipment_id, name, type, plant, production_line, manufacturer, status, criticality, hours_operated)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_SENSOR_READINGS
```
Crea models/staging/stg_sensor_readings.sql che esegua il flatten di RAW_SENSOR_READINGS.
Campi: letture di sensori IoT (reading_id, equipment_id, sensor_type, value, threshold_min, threshold_max, is_anomaly)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_WORK_ORDERS
```
Crea models/staging/stg_work_orders.sql che esegua il flatten di RAW_WORK_ORDERS.
Campi: ordini di lavoro (work_order_id, equipment_id, category, priority, description, estimated_hours, spare_parts_cost_eur)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_PRODUCTION_BATCHES
```
Crea models/staging/stg_production_batches.sql che esegua il flatten di RAW_PRODUCTION_BATCHES.
Campi: lotti di produzione (batch_id, product_name, production_line, planned_quantity, actual_quantity, defect_count, material_cost_eur, quality_grade)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_EQUIPMENT; -- atteso: 350
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_SENSOR_READINGS; -- atteso: 10,000
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS; -- atteso: 750
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.SILVER.STG_PRODUCTION_BATCHES; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: EQUIPMENT_360
```
Crea models/marts/equipment_360.sql:
Vista 360 di ogni attrezzatura: dati base + total_sensor_anomalies + total_work_orders + open_work_orders + total_maintenance_cost_eur + avg_production_quality + hours_operated + days_since_last_maintenance. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: PRODUCTION_HEALTH
```
Crea models/marts/production_health.sql:
Salute della produzione per production_line, plant e giorno: total_batches, total_produced, total_defects, defect_rate_pct, avg_quality_grade, total_material_cost, total_energy_cost.
Materializza come table in GOLD.
```

**[CoCo]** Gold: MAINTENANCE_RISK
```
Crea models/marts/maintenance_risk.sql:
Rischio di manutenzione per attrezzatura: hours_operated + anomaly_count_last_90d + pending_work_orders + avg_sensor_deviation + maintenance_risk_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360;
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.GOLD.PRODUCTION_HEALTH;
SELECT COUNT(*) FROM MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Classificazione degli ordini di lavoro
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella MANUFACTURING_COPILOT.SILVER.STG_WORK_ORDERS.
Analizza il campo "description" e restituisce un JSON con: tipo_manutenzione (correttiva|preventiva|predittiva|miglioramento|ispezione|calibrazione), urgenza (critica|alta|media|bassa), specializzazione_richiesta (meccanica|elettrica|strumentazione|automazione|generale), impatto_produzione (alto|medio|basso|nullo), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione degli ordini di lavoro di un impianto industriale."
Crea la vista MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_WORK_ORDER_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo dell'attrezzatura
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella MANUFACTURING_COPILOT.GOLD.EQUIPMENT_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: name, type, plant, production_line, status, criticality, hours_operated, total_sensor_anomalies, total_work_orders, total_maintenance_cost_eur
Prompt di sistema: "Sei un ingegnere di affidabilita industriale. Genera un riassunto esecutivo in italiano (3-4 frasi) dello stato di questa attrezzatura."
Crea la vista MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_EQUIPMENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di manutenzione predittiva
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella MANUFACTURING_COPILOT.GOLD.MAINTENANCE_RISK.
Restituisce un JSON con: livello_rischio_guasto (alto|medio|basso), azione_raccomandata (sostituzione|riparazione|ispezione|monitoraggio), componente_sospetto, termine_massimo_giorni (numero), impatto_se_guasto (linea_ferma|produzione_ridotta|qualita_degradata), costo_stimato_eur
Prompt di sistema: "Sei un esperto di manutenzione predittiva industriale. Analizza questa attrezzatura e restituisci SOLO un JSON valido."
Filtra WHERE maintenance_risk_score >= 30.
Crea la vista MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM MANUFACTURING_COPILOT.AI.VW_PREDICTIVE_MAINTENANCE LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in MANUFACTURING_COPILOT.GOLD chiamata SV_MANUFACTURING_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono attrezzature industriali, letture di sensori IoT, ordini di lavoro/manutenzione e lotti di produzione. Impianto industriale in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent in Snowflake Intelligence per Manifatturiero e Industria.
- Crea l'oggetto Snowflake Intelligence predefinito se non esiste
- Crea il database SNOWFLAKE_INTELLIGENCE e lo schema AGENTS
- Crea l'agente MANUFACTURING_COPILOT_AGENT connesso alla Semantic View MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT
  con modello di orchestrazione claude-4-sonnet, rispondendo sempre in italiano
  e contesto di business: azienda manifatturiera/industriale in Italia
- Concedi accesso al ruolo PUBLIC all'agente e alla Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT
  COMMENT = 'Copilot di Manifatturiero e Industria'
  PROFILE = '{"display_name": "Smart Factory Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di azienda manifatturiera/industriale in Italia. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "manufacturing_analyst",
        "description": "I dati coprono attrezzature industriali, letture di sensori IoT, ordini di lavoro/manutenzione e lotti di produzione. Impianto industriale in Italia."
      }
    }],
    "tool_resources": {
      "manufacturing_analyst": {
        "semantic_view": "MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MANUFACTURING_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW MANUFACTURING_COPILOT.GOLD.SV_MANUFACTURING_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.

---

## Prova l'Agente

Domande di esempio per validare l'agente:

- "Quali attrezzature hanno maggior rischio di guasto nei prossimi 7 giorni?"
- "Quanti ordini di lavoro critici sono in attesa per impianto?"
- "Qual e il tasso di difetti per linea di produzione questa settimana?"
- "Quali sensori IoT generano piu anomalie per attrezzatura?"

---

## FASE FINALE — App Streamlit in Snowflake

**[CoCo]**
```
Crea un'app Streamlit in Snowflake (SiS) nel database MANUFACTURING_COPILOT, schema AI, chiamata MANUFACTURING_APP.
L'app deve connettersi a Snowflake con la sessione attiva (get_active_session).
Includi 4 sezioni usando st.tabs:
1. KPI — card con st.metric con i totali e i ratio principali di EQUIPMENT_360
2. Distribuzione — grafico a barre (st.bar_chart) per categoria/regione da PRODUCTION_HEALTH
3. Top 20 — tabella interattiva (st.dataframe) ordinata per score di rischio da MAINTENANCE_RISK
4. Tendenza — grafico a linee (st.line_chart) con l'evoluzione temporale degli eventi principali
Usa titoli ed etichette descrittivi in italiano. Applica un tema scuro con st.set_page_config.
```
