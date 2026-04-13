# 💧 Water & Utilities Plant Copilot — Talk to your Plant
**DB:** `UTILITIES_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato UTILITIES_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE UTILITIES_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in UTILITIES_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @UTILITIES_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/utilities/;
```

**[CoCo]** Tabelle Bronze
```
In UTILITIES_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_PLANTS da data/utilities/plants.json
- RAW_SENSOR_EVENTS da data/utilities/sensor_events.json
- RAW_INCIDENTS da data/utilities/incidents.json
- RAW_WATER_QUALITY da data/utilities/water_quality.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_PLANTS; -- atteso: 350
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_SENSOR_EVENTS; -- atteso: 10,000
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_INCIDENTS; -- atteso: 750
SELECT COUNT(*) FROM UTILITIES_COPILOT.BRONZE.RAW_WATER_QUALITY; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato utilities_copilot connesso a UTILITIES_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_PLANTS, RAW_SENSOR_EVENTS, RAW_INCIDENTS, RAW_WATER_QUALITY
```

**[CoCo]** Silver: STG_PLANTS
```
Crea models/staging/stg_plants.sql che esegua il flatten di RAW_PLANTS.
Campi: impianti/installazioni (plant_id, plant_name, plant_type, operator, region, zone, capacity_m3_day, current_flow_m3_day, status, criticality, annual_budget_eur)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_SENSOR_EVENTS
```
Crea models/staging/stg_sensor_events.sql che esegua il flatten di RAW_SENSOR_EVENTS.
Campi: eventi di sensori SCADA (event_id, plant_id, sensor_type, value, alarm_min, alarm_max, is_alarm, subsystem)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_INCIDENTS
```
Crea models/staging/stg_incidents.sql che esegua il flatten di RAW_INCIDENTS.
Campi: incidenti operativi (incident_id, plant_id, category, priority, description, affected_population, regulatory_notification_required, repair_cost_eur)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_WATER_QUALITY
```
Crea models/staging/stg_water_quality.sql che esegua il flatten di RAW_WATER_QUALITY.
Campi: campioni di qualita dell'acqua (sample_id, plant_id, parameter, value, legal_min, legal_max, compliant, sample_point, reported_to_authority)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_PLANTS; -- atteso: 350
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_SENSOR_EVENTS; -- atteso: 10,000
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_INCIDENTS; -- atteso: 750
SELECT COUNT(*) FROM UTILITIES_COPILOT.SILVER.STG_WATER_QUALITY; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: PLANT_360
```
Crea models/marts/plant_360.sql:
Vista 360 di ogni impianto: dati base + total_sensor_alarms + alarm_rate_pct + total_incidents + open_incidents + avg_repair_cost_eur + quality_non_compliant_count + regulatory_notifications + days_since_last_inspection. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: OPERATIONAL_HEALTH
```
Crea models/marts/operational_health.sql:
Salute operativa per impianto e giorno: total_sensor_events, alarm_count, alarm_rate_pct, top_alarm_sensor, subsystems_affected, avg_flow_m3h, energy_consumption_kWh.
Materializza come table in GOLD.
```

**[CoCo]** Gold: COMPLIANCE_RISK
```
Crea models/marts/compliance_risk.sql:
Rischio di compliance per impianto: non_compliant_samples_last_90d + regulatory_notifications + open_critical_incidents + alarm_rate + days_overdue_inspection + compliance_risk_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM UTILITIES_COPILOT.GOLD.PLANT_360;
SELECT COUNT(*) FROM UTILITIES_COPILOT.GOLD.OPERATIONAL_HEALTH;
SELECT COUNT(*) FROM UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Classificazione degli incidenti operativi
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella UTILITIES_COPILOT.SILVER.STG_INCIDENTS.
Analizza il campo "description" e restituisce un JSON con: tipo_incidente (guasto_pompa|qualita_acqua|perdita_rete|guasto_clorazione|allarme_serbatoio|rottura_tubatura|guasto_scada|contaminazione), urgenza (critica|alta|media|bassa), team_responsabile (operazioni|qualita|manutenzione|laboratorio|regolatorio|emergenze), notifica_regolatoria (si|no), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione degli incidenti di un'azienda di gestione dell'acqua (utilities). Classifica l'incidente descritto."
Crea la vista UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_INCIDENT_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo dell'impianto (Talk to your Plant)
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella UTILITIES_COPILOT.GOLD.PLANT_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: plant_name, plant_type, operator, region, status, criticality, total_sensor_alarms, total_incidents, open_incidents, quality_non_compliant_count, regulatory_notifications
Prompt di sistema: "Sei un direttore operativo di un impianto di trattamento delle acque. Genera un riassunto esecutivo in italiano (3-4 frasi) dello stato operativo di questa installazione."
Crea la vista UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_PLANT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di azione operativa
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella UTILITIES_COPILOT.GOLD.COMPLIANCE_RISK.
Restituisce un JSON con: livello_rischio_regolatorio (alto|medio|basso), azione_principale (ispezione_urgente|regolazione_trattamento|manutenzione_preventiva|notifica_autorita|monitoraggio_intensivo), parametri_critici (array 2-3), termine_massimo_ore (numero), impatto_popolazione (numero_persone), raccomandazione_tecnica
Prompt di sistema: "Sei un esperto di gestione delle utilities e compliance regolatoria dell'acqua. Analizza questo impianto e restituisci SOLO un JSON valido."
Filtra WHERE compliance_risk_score >= 30.
Crea la vista UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM UTILITIES_COPILOT.AI.VW_OPERATIONAL_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in UTILITIES_COPILOT.GOLD chiamata SV_UTILITIES_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono impianti di trattamento delle acque (ETAP/EDAR/pompaggio), eventi di sensori SCADA, incidenti operativi e campioni di qualita dell'acqua. Azienda di utilities/gestione dell'acqua in Italia (Veolia, Acque Italiane, ACEA, Gruppo CAP).
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT
  COMMENT = 'Copilot di Utilities e Gestione dell'Acqua'
  PROFILE = '{"display_name": "Water & Utilities Plant Copilot — Talk to your Plant"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di azienda di gestione dell'acqua e utilities in Italia (Veolia, Acque Italiane, ACEA, Gruppo CAP). Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "utilities_analyst",
        "description": "I dati coprono impianti di trattamento delle acque (ETAP/EDAR/pompaggio), eventi di sensori SCADA, incidenti operativi e campioni di qualita dell'acqua. Azienda di utilities/gestione dell'acqua in Italia (Veolia, Acque Italiane, ACEA, Gruppo CAP)."
      }
    }],
    "tool_resources": {
      "utilities_analyst": {
        "semantic_view": "UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.UTILITIES_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW UTILITIES_COPILOT.GOLD.SV_UTILITIES_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
