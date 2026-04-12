# 🏥 Clinical Intelligence Copilot
**DB:** `HEALTHCARE_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada HEALTHCARE_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE HEALTHCARE_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en HEALTHCARE_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY HEALTHCARE_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @HEALTHCARE_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/healthcare/;
```

**[CoCo]** Tablas Bronze
```
En HEALTHCARE_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_PATIENTS desde data/healthcare/patients.json
- RAW_ENCOUNTERS desde data/healthcare/encounters.json
- RAW_CLINICAL_NOTES desde data/healthcare/clinical_notes.json
- RAW_CLAIMS desde data/healthcare/claims.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_PATIENTS; -- esperado: 350
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_ENCOUNTERS; -- esperado: 10,000
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_CLINICAL_NOTES; -- esperado: 750
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.BRONZE.RAW_CLAIMS; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado healthcare_copilot conectado a HEALTHCARE_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_PATIENTS, RAW_ENCOUNTERS, RAW_CLINICAL_NOTES, RAW_CLAIMS
```

**[CoCo]** Silver: STG_PATIENTS
```
Crea models/staging/stg_patients.sql que haga flatten de RAW_PATIENTS.
Campos: pacientes (patient_id, full_name, age, gender, region, insurance_type, chronic_conditions, department, risk_level)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_ENCOUNTERS
```
Crea models/staging/stg_encounters.sql que haga flatten de RAW_ENCOUNTERS.
Campos: encuentros clinicos (encounter_id, patient_id, encounter_type, department, diagnosis_code, duration_minutes)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_CLINICAL_NOTES
```
Crea models/staging/stg_clinical_notes.sql que haga flatten de RAW_CLINICAL_NOTES.
Campos: notas clinicas (note_id, patient_id, category, priority, description, requires_followup)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_CLAIMS
```
Crea models/staging/stg_claims.sql que haga flatten de RAW_CLAIMS.
Campos: reclamaciones/facturacion (claim_id, patient_id, claim_type, amount_eur, insurance_response, status)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_PATIENTS; -- esperado: 350
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_ENCOUNTERS; -- esperado: 10,000
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_CLINICAL_NOTES; -- esperado: 750
SELECT COUNT(*) FROM HEALTHCARE_COPILOT.SILVER.STG_CLAIMS; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: PATIENT_360
```
Crea models/marts/patient_360.sql:
Vista 360 del paciente: datos basicos + total_encounters + emergency_visits + total_claims_eur + pending_claims + active_conditions + followup_notes + last_encounter_date. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: CLINICAL_HEALTH
```
Crea models/marts/clinical_health.sql:
Salud clinica por department y dia: total_encounters, avg_duration_minutes, emergency_rate, readmission_count, distinct_patients, top_diagnosis_code.
Materializa como table en GOLD.
```

**[CoCo]** Gold: READMISSION_RISK
```
Crea models/marts/readmission_risk.sql:
Riesgo de reingreso por paciente: age + chronic_conditions + emergency_visits_last_90d + pending_followups + claim_rejections + readmission_risk_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
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

**[CoCo]** Caso 1: Clasificacion de notas clinicas
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla HEALTHCARE_COPILOT.SILVER.STG_CLINICAL_NOTES.
Analiza el campo "description" y devuelve un JSON con: especialidad_principal (cardiologia|traumatologia|oncologia|medicina_interna|urgencias|neurologia|neumologia), urgencia (critica|alta|media|baja), tipo_atencion (diagnostico|seguimiento|interconsulta|alta|urgencia), requiere_seguimiento (si|no), resumen_clinico (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion clinica de un hospital."
Crea la vista HEALTHCARE_COPILOT.AI.VW_CLINICAL_NOTE_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_CLINICAL_NOTE_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de paciente
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla HEALTHCARE_COPILOT.GOLD.PATIENT_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: full_name, age, gender, chronic_conditions, total_encounters, emergency_visits, active_conditions, total_claims_eur, risk_level
Prompt del sistema: "Eres un medico internista. Genera un resumen clinico ejecutivo en espanol (3-4 frases) de este paciente."
Crea la vista HEALTHCARE_COPILOT.AI.VW_PATIENT_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_PATIENT_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de seguimiento clinico
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla HEALTHCARE_COPILOT.GOLD.READMISSION_RISK.
Devuelve un JSON con: nivel_riesgo_reingreso (alto|medio|bajo), accion_principal (cita_preferente|teleconsulta|hospitalizacion_domiciliaria|monitorizacion), especialidad_derivacion, plazo_dias (numero), plan_seguimiento (array 2-3 acciones), mensaje_para_medico
Prompt del sistema: "Eres un especialista en gestion clinica. Analiza este paciente y devuelve SOLO un JSON valido."
Filtra WHERE readmission_risk_score >= 30.
Crea la vista HEALTHCARE_COPILOT.AI.VW_FOLLOWUP_RECOMMENDATIONS. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM HEALTHCARE_COPILOT.AI.VW_FOLLOWUP_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en HEALTHCARE_COPILOT.GOLD llamada SV_HEALTHCARE_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren pacientes, encuentros clinicos, notas clinicas y reclamaciones de seguro. Hospital/clinica en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
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
  COMMENT = 'Copilot de Healthcare y Life Sciences'
  PROFILE = '{"display_name": "Clinical Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de organizacion sanitaria en Espana (hospital/clinica). Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "healthcare_analyst",
        "description": "Los datos cubren pacientes, encuentros clinicos, notas clinicas y reclamaciones de seguro. Hospital/clinica en Espana."
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

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.
