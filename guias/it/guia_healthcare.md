# 🏥 Clinical Intelligence Copilot
**DB:** `HEALTHCARE_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato HEALTHCARE_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE HEALTHCARE_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in HEALTHCARE_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY HEALTHCARE_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @HEALTHCARE_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/healthcare/;
```

**[CoCo]** Tabelle Bronze
```
In HEALTHCARE_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_PATIENTS da data/healthcare/patients.json
- RAW_ENCOUNTERS da data/healthcare/encounters.json
- RAW_CLINICAL_NOTES da data/healthcare/clinical_notes.json
- RAW_CLAIMS da data/healthcare/claims.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_PATIENTS; -- atteso: 350
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_ENCOUNTERS; -- atteso: 10,000
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_CLINICAL_NOTES; -- atteso: 750
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_CLAIMS; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato healthcare_copilot connesso a HEALTHCARE_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_PATIENTS, RAW_ENCOUNTERS, RAW_CLINICAL_NOTES, RAW_CLAIMS
```

**[CoCo]** Silver: STG_PATIENTS
```
Crea models/staging/stg_patients.sql che esegua il flatten di RAW_PATIENTS.
Campi: pazienti (patient_id, full_name, age, gender, region, insurance_type, chronic_conditions, department, risk_level)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_ENCOUNTERS
```
Crea models/staging/stg_encounters.sql che esegua il flatten di RAW_ENCOUNTERS.
Campi: incontri clinici (encounter_id, patient_id, encounter_type, department, diagnosis_code, duration_minutes)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_CLINICAL_NOTES
```
Crea models/staging/stg_clinical_notes.sql che esegua il flatten di RAW_CLINICAL_NOTES.
Campi: note cliniche (note_id, patient_id, category, priority, description, requires_followup)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_CLAIMS
```
Crea models/staging/stg_claims.sql che esegua il flatten di RAW_CLAIMS.
Campi: richieste di rimborso/fatturazione (claim_id, patient_id, claim_type, amount_eur, insurance_response, status)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_PATIENTS; -- atteso: 350
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_ENCOUNTERS; -- atteso: 10,000
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_CLINICAL_NOTES; -- atteso: 750
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_CLAIMS; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: PATIENT_360
```
Crea models/marts/patient_360.sql:
Vista 360 del paziente: dati base + total_encounters + emergency_visits + total_claims_eur + pending_claims + active_conditions + followup_notes + last_encounter_date. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: CLINICAL_HEALTH
```
Crea models/marts/clinical_health.sql:
Salute clinica per department e giorno: total_encounters, avg_duration_minutes, emergency_rate, readmission_count, distinct_patients, top_diagnosis_code.
Materializza come table in GOLD.
```

**[CoCo]** Gold: READMISSION_RISK
```
Crea models/marts/readmission_risk.sql:
Rischio di riammissione per paziente: age + chronic_conditions + emergency_visits_last_90d + pending_followups + claim_rejections + readmission_risk_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.GOLD.PATIENT_360;
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.GOLD.CLINICAL_HEALTH;
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.GOLD.READMISSION_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Classificazione delle note cliniche
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella HEALTHCARE_COPILOT.SILVER.STG_CLINICAL_NOTES.
Analizza il campo "description" e restituisce un JSON con: specializzazione_principale (cardiologia|traumatologia|oncologia|medicina_interna|urgenze|neurologia|pneumologia), urgenza (critica|alta|media|bassa), tipo_assistenza (diagnosi|follow_up|interconsulta|dimissione|urgenza), richiede_follow_up (si|no), riassunto_clinico (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione clinica di un ospedale."
Crea la vista HEALTHCARE_COPILOT.AI.VW_CLINICAL_NOTE_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_CLINICAL_NOTE_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo del paziente
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella HEALTHCARE_COPILOT.GOLD.PATIENT_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: full_name, age, gender, chronic_conditions, total_encounters, emergency_visits, active_conditions, total_claims_eur, risk_level
Prompt di sistema: "Sei un medico internista. Genera un riassunto clinico esecutivo in italiano (3-4 frasi) di questo paziente."
Crea la vista HEALTHCARE_COPILOT.AI.VW_PATIENT_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_PATIENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di follow-up clinico
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella HEALTHCARE_COPILOT.GOLD.READMISSION_RISK.
Restituisce un JSON con: livello_rischio_riammissione (alto|medio|basso), azione_principale (appuntamento_urgente|teleconsulta|ospedalizzazione_domiciliare|monitoraggio), specializzazione_derivazione, termine_giorni (numero), piano_follow_up (array 2-3 azioni), messaggio_per_medico
Prompt di sistema: "Sei uno specialista in gestione clinica. Analizza questo paziente e restituisci SOLO un JSON valido."
Filtra WHERE readmission_risk_score >= 30.
Crea la vista HEALTHCARE_COPILOT.AI.VW_FOLLOWUP_RECOMMENDATIONS. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_FOLLOWUP_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in HEALTHCARE_COPILOT.GOLD chiamata SV_HEALTHCARE_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono pazienti, incontri clinici, note cliniche e richieste di assicurazione. Ospedale/clinica in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.HEALTHCARE_COPILOT_AGENT
  COMMENT = 'Copilot di Sanita e Life Sciences'
  PROFILE = '{"display_name": "Clinical Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di organizzazione sanitaria in Italia (ospedale/clinica). Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "healthcare_analyst",
        "description": "I dati coprono pazienti, incontri clinici, note cliniche e richieste di assicurazione. Ospedale/clinica in Italia."
      }
    }],
    "tool_resources": {
      "healthcare_analyst": {
        "semantic_view": "HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.HEALTHCARE_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW HEALTHCARE_COPILOT.GOLD.SV_HEALTHCARE_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
