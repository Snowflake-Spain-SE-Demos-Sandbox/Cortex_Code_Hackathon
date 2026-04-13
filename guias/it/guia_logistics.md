# 🚚 Logistics & Fleet Copilot
**DB:** `LOGISTICS_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato LOGISTICS_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE LOGISTICS_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in LOGISTICS_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/;
```

**[CoCo]** Tabelle Bronze
```
In LOGISTICS_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_FLEET da data/logistics/fleet_vehicles.json
- RAW_ROUTE_EVENTS da data/logistics/route_events.json
- RAW_DELIVERY_TICKETS da data/logistics/delivery_tickets.json
- RAW_SHIPMENTS da data/logistics/shipments.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_FLEET; -- atteso: 350
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_ROUTE_EVENTS; -- atteso: 10,000
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_DELIVERY_TICKETS; -- atteso: 750
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_SHIPMENTS; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato logistics_copilot connesso a LOGISTICS_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_FLEET, RAW_ROUTE_EVENTS, RAW_DELIVERY_TICKETS, RAW_SHIPMENTS
```

**[CoCo]** Silver: STG_FLEET
```
Crea models/staging/stg_fleet.sql che esegua il flatten di RAW_FLEET.
Campi: veicoli della flotta (vehicle_id, plate_number, type, brand, base_location, max_payload_kg, fuel_type, km_total, status)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_ROUTE_EVENTS
```
Crea models/staging/stg_route_events.sql che esegua il flatten di RAW_ROUTE_EVENTS.
Campi: eventi di percorso (event_id, vehicle_id, shipment_id, event_type, location, gps_lat, gps_lon, speed_kmh)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_DELIVERY_TICKETS
```
Crea models/staging/stg_delivery_tickets.sql che esegua il flatten di RAW_DELIVERY_TICKETS.
Campi: ticket di incidenti (ticket_id, shipment_id, vehicle_id, category, priority, description, sla_breached, estimated_cost_eur)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_SHIPMENTS
```
Crea models/staging/stg_shipments.sql che esegua il flatten di RAW_SHIPMENTS.
Campi: spedizioni (shipment_id, client_name, origin, destination, service_type, amount_eur, weight_kg, status)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_FLEET; -- atteso: 350
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_ROUTE_EVENTS; -- atteso: 10,000
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_DELIVERY_TICKETS; -- atteso: 750
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_SHIPMENTS; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: VEHICLE_360
```
Crea models/marts/vehicle_360.sql:
Vista 360 del veicolo: dati base + total_shipments + total_route_events + incidents_count + total_km_last_90d + total_revenue_eur + avg_delivery_time_hours + days_until_maintenance. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: ROUTE_HEALTH
```
Crea models/marts/route_health.sql:
Salute dei percorsi per origin, destination e giorno: total_shipments, total_events, delay_events, avg_delivery_hours, on_time_rate, total_revenue_eur, distinct_vehicles.
Materializza come table in GOLD.
```

**[CoCo]** Gold: DELIVERY_RISK
```
Crea models/marts/delivery_risk.sql:
Rischio di incidenti per veicolo/percorso: pending_tickets + sla_breaches_last_90d + delay_count + damage_claims + km_since_maintenance + delivery_risk_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.GOLD.VEHICLE_360;
SELECT COUNT(*) FROM LOGISTICS_COPILOT.GOLD.ROUTE_HEALTH;
SELECT COUNT(*) FROM LOGISTICS_COPILOT.GOLD.DELIVERY_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Classificazione degli incidenti di consegna
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella LOGISTICS_COPILOT.SILVER.STG_DELIVERY_TICKETS.
Analizza il campo "description" e restituisce un JSON con: tipo_incidente (ritardo|danno|indirizzo|veicolo|documentazione|rifiuto|temperatura|peso), urgenza (critica|alta|media|bassa), dipartimento (operazioni|flotta|qualita|assistenza_clienti|legale), impatto_servizio (alto|medio|basso), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione degli incidenti di un'azienda di trasporti e logistica."
Crea la vista LOGISTICS_COPILOT.AI.VW_DELIVERY_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_DELIVERY_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo del veicolo/flotta
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella LOGISTICS_COPILOT.GOLD.VEHICLE_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: plate_number, type, base_location, km_total, total_shipments, incidents_count, total_revenue_eur, days_until_maintenance
Prompt di sistema: "Sei un responsabile della flotta di trasporto. Genera un riassunto esecutivo in italiano (3-4 frasi) dello stato di questo veicolo."
Crea la vista LOGISTICS_COPILOT.AI.VW_FLEET_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_FLEET_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione di ottimizzazione del percorso
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella LOGISTICS_COPILOT.GOLD.DELIVERY_RISK.
Restituisce un JSON con: livello_rischio (alto|medio|basso), azione_principale (cambio_percorso|manutenzione_veicolo|rinforzo_capacita|formazione_autista), miglioramento_stimato_pct, termine_implementazione_giorni, impatto_costo_eur, raccomandazione_dettagliata
Prompt di sistema: "Sei un esperto di ottimizzazione logistica. Analizza questo percorso/veicolo e restituisci SOLO un JSON valido."
Filtra WHERE delivery_risk_score >= 30.
Crea la vista LOGISTICS_COPILOT.AI.VW_ROUTE_OPTIMIZATION. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_ROUTE_OPTIMIZATION LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in LOGISTICS_COPILOT.GOLD chiamata SV_LOGISTICS_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono veicoli della flotta, eventi di percorso, ticket di incidenti di consegna e spedizioni. Azienda di trasporti/logistica in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.LOGISTICS_COPILOT_AGENT
  COMMENT = 'Copilot di Trasporti, Logistica e Travel'
  PROFILE = '{"display_name": "Logistics & Fleet Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di azienda di trasporti e logistica in Italia. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "logistics_analyst",
        "description": "I dati coprono veicoli della flotta, eventi di percorso, ticket di incidenti di consegna e spedizioni. Azienda di trasporti/logistica in Italia."
      }
    }],
    "tool_resources": {
      "logistics_analyst": {
        "semantic_view": "LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.LOGISTICS_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
