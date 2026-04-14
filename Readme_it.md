# Build an AI-Powered Industry Copilot
## Hackathon Cortex Code Experience

**Data:** 27 maggio 2026
**Orario:** 09:30 - 13:30
**Luogo:** Uffici Snowflake, Madrid
**Partecipanti:** 30 persone
**Catering:** Pranzo servito dalle 13:30

**Registrazione:** [https://www.snowflake.com/event/hackathon-cortex-code-experience-madrid-20260527/](https://www.snowflake.com/event/hackathon-cortex-code-experience-madrid-20260527/)

---

## Scegli il tuo Settore

Ogni team seleziona un settore e lavora con i suoi dati specifici:

| # | Settore | Database | Guida | Dataset |
|---|---------|----------|-------|---------|
| 1 | 📡 **Telecomunicazioni** | `TELCO_COPILOT` | [guia_telco.md](guias/it/guia_telco.md) | `data/telco/` |
| 2 | 🏦 **Servizi Finanziari** | `FSI_COPILOT` | [guia_fsi.md](guias/it/guia_fsi.md) | `data/fsi/` |
| 3 | 🛍️ **Retail e Beni di Consumo** | `RETAIL_COPILOT` | [guia_retail.md](guias/it/guia_retail.md) | `data/retail/` |
| 4 | 🏭 **Manifatturiero e Industria** | `MANUFACTURING_COPILOT` | [guia_manufacturing.md](guias/it/guia_manufacturing.md) | `data/manufacturing/` |
| 5 | 🏥 **Sanità e Life Sciences** | `HEALTHCARE_COPILOT` | [guia_healthcare.md](guias/it/guia_healthcare.md) | `data/healthcare/` |
| 6 | 📺 **Media, Entertainment e Advertising** | `MEDIA_COPILOT` | [guia_media.md](guias/it/guia_media.md) | `data/media/` |
| 7 | 🚚 **Trasporti, Logistica e Travel** | `LOGISTICS_COPILOT` | [guia_logistics.md](guias/it/guia_logistics.md) | `data/logistics/` |
| 8 | 🌐 **Settore Pubblico** | `PUBLIC_SECTOR_COPILOT` | [guia_public_sector.md](guias/it/guia_public_sector.md) | `data/public_sector/` |
| 9 | 💧 **Utilities e Gestione dell'Acqua** | `UTILITIES_COPILOT` | [guia_utilities.md](guias/it/guia_utilities.md) | `data/utilities/` |
| 10 | 🛡️ **Assicurazioni — Call Center AI** | `INSURANCE_COPILOT` | [guia_insurance.md](guias/it/guia_insurance.md) | `data/insurance/` |

> **Settore speciale:** Il caso d'uso Assicurazioni include una **FASE 4 di Machine Learning** (rilevamento churn con `SNOWFLAKE.ML.CLASSIFICATION`) e **5 casi d'uso AI** su trascrizioni reali di chiamate di 5-10 minuti.

---

## Prerequisito: Account Trial Snowflake

Ogni partecipante deve creare il proprio account trial **prima dell'evento**:

1. Andare su **https://signup.snowflake.com/**
2. Compilare il modulo con la propria email aziendale
3. Selezionare: **AWS** | **Enterprise** | **US West (Oregon)**
4. Attivare l'account dall'email di conferma
5. L'account include **$400 di crediti** e 30 giorni di validità

(ALTER ACCOUNT DAB85420 SET COPILOT_ORCHESTRATOR_PARAM_153 = 'false' parameter_comment = 'Access for Enablement Session in Trial Account';)

---

## Obiettivo

Ogni team costruirà una mini piattaforma end-to-end per il proprio settore usando Snowflake Cortex Code come unico IDE:

- Connettere un repository Git di GitHub a Snowflake e ingerire dati JSON
- Trasformare con il pattern medallion Bronze / Silver / Gold usando dbt
- Applicare casi d'uso AI con Cortex COMPLETE sul livello Gold
- *(Solo Assicurazioni)* Addestrare un modello predittivo di churn con `SNOWFLAKE.ML.CLASSIFICATION`
- Creare una Semantic View per query in linguaggio naturale (NL2SQL)
- Distribuire un Cortex Agent come Copilot in Snowflake Intelligence
- Visualizzare i risultati in un'app Streamlit in Snowflake

---

## Agenda Dettagliata

| Ora | Durata | Blocco | Descrizione |
|-----|--------|--------|-------------|
| 09:30 - 10:00 | 30 min | **Accoglienza e registrazione** | Accreditamento, distribuzione materiali, caffè di benvenuto |
| 10:00 - 10:15 | 15 min | **Benvenuto** | Apertura istituzionale Snowflake. Presentazione dell'evento |
| 10:15 - 10:35 | 20 min | **Keynote: Cortex Code** | Demo live di Cortex Code: flusso Git > Bronze > Silver > Gold > AI > SV > Agent |
| 10:35 - 10:45 | 10 min | **Presentazione della sfida** | Spiegazione dei settori disponibili e delle fasi dell'hackathon |
| 10:45 - 10:50 | 5 min | **Formazione dei team** | Scelta del settore, assegnazione dei tavoli, distribuzione delle guide |
| 10:50 - 12:20 | 90 min | **Hackathon** | Lavoro in team seguendo la guida del proprio settore |
| | | *Fase 1 - Bronze (20 min)* | Connettere il repo Git e ingerire JSON come livello Bronze |
| | | *Fase 2 - Silver/Gold (30 min)* | Trasformazione con dbt: tabelle normalizzate e KPI |
| | | *Fase 3 - AI (20 min)* | Casi d'uso AI con Cortex COMPLETE |
| | | *Fase 4 - ML (10 min)* | *(Solo Assicurazioni)* Modello churn con Snowflake ML |
| | | *Fase 5/6 - Semantic View + Agent (10 min)* | Modello semantico NL2SQL + Copilot in Snowflake Intelligence |
| | | *Fase Finale - Streamlit (10 min)* | App Streamlit con le 4 sezioni |
| 12:20 - 12:30 | 10 min | **Coffee break** | Pausa, preparazione delle demo |
| 12:30 - 13:00 | 30 min | **Demo dei team** | 5 min per team |
| 13:00 - 13:15 | 15 min | **Premi** | Votazione e consegna dei premi |
| 13:15 - 13:30+ | 15 min+ | **Chiusura e networking** | Pranzo catering e networking |

---

## Architettura della Sfida

### Settori standard (9 settori)

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
        |
   [ BRONZE ]  JSON grezzo (4 file per settore)
        |
   [ SILVER ]  Tabelle normalizzate con dbt (flatten, cast, dedup)
        |
   [  GOLD  ]  KPI e viste analitiche (3 modelli per settore)
        |
   [   AI   ]  3 Casi d'Uso con Cortex COMPLETE (llama3.1-70b)
        |
   [SEM.VIEW]  Modello semantico NL2SQL (Cortex Analyst)
        |
   [ AGENT  ]  Copilot in Snowflake Intelligence (claude-4-sonnet)
        |
   [  APP   ]  Dashboard Streamlit in Snowflake
```

### Settore speciale: Assicurazioni — Call Center AI

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
        |
   [ BRONZE ]  4 file: customers, call_transcripts, claims, agents
        |
   [ SILVER ]  Tabelle normalizzate con dbt
        |
   [  GOLD  ]  CUSTOMER_RISK_360 | CALL_INTELLIGENCE_360 | AGENT_PERFORMANCE
        |
   [   AI   ]  5 Casi AI: riassunto chiamate | classificazione+routing |
               QA agenti | CX insights | rilevamento segnali churn
        |
   [   ML   ]  SNOWFLAKE.ML.CLASSIFICATION — modello predittivo di churn
        |
   [SEM.VIEW]  Semantic View NL2SQL su Gold + AI
        |
   [ AGENT  ]  Copilot in Snowflake Intelligence (claude-4-sonnet)
        |
   [  APP   ]  Streamlit: KPI | Chiamate | Churn Risk | Performance Agenti
```

---

## Dataset per Settore

### Settori standard (1-9)

Ogni settore include 4 file JSON con la stessa struttura di volumetria:

| File | Record | Ruolo |
|------|--------|-------|
| Entità principali | 350 | Clienti, pazienti, attrezzature, veicoli, ecc. |
| Eventi operativi | 10.000 | Transazioni, letture, ordini, incontri, ecc. |
| Ticket / Incidenti | 750 | Alert, casi, ordini di lavoro, note cliniche, ecc. |
| Dati finanziari | 2.000 | Fatturazione, conti, spedizioni, bilanci, ecc. |

### Settore 10: Assicurazioni — Call Center AI

Dataset specializzato per l'analisi delle chiamate al call center (assicurazioni auto e casa):

| File | Record | Contenuto |
|------|--------|-----------|
| `customers.json` | 350 | Clienti con polizze auto, casa o entrambe. Premio annuale, storico sinistri, NPS, stato |
| `call_transcripts.json` | 500 | Trascrizioni complete di chiamate di 5-10 min (8 tipi: sinistro, cancellazione, reclamo, rinnovo, ecc.) |
| `claims.json` | 1.000 | Sinistri con tipo, importi, franchigia, stato e giorni di risoluzione |
| `agents.json` | 50 | Agenti con team, esperienza, FCR, AHT e QA score |

**Repository:** https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon
**Percorso:** `data/{settore}/`

---

## Criteri di Valutazione

| Criterio | Peso |
|----------|------|
| Funzionalità end-to-end (da Bronze ad Agent) | 30% |
| Uso efficace di Cortex Code | 20% |
| Qualità dei casi AI (e ML per Assicurazioni) | 20% |
| Semantic View + Cortex Agent | 15% |
| Presentazione e storytelling | 15% |

---

## Revisione dei Costi Post-Hackathon

```sql
SELECT service_type, ROUND(SUM(credits_used), 2) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY service_type ORDER BY total_credits DESC;

SELECT function_name, model_name, SUM(tokens) AS total_tokens, ROUND(SUM(token_credits), 4) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY function_name, model_name ORDER BY total_credits DESC;
```

> Le viste di ACCOUNT_USAGE hanno un ritardo fino a 2 ore. Consultarle il giorno successivo.
