# 🛍️ Retail Customer Intelligence Copilot
**DB:** `RETAIL_COPILOT`

---

## FASE 0 — Configurazione

**[CoCo]**
```
Crea un database chiamato RETAIL_COPILOT con quattro schemi: BRONZE, SILVER, GOLD e AI.
Usa il warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE RETAIL_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Integrazione Git
```
Crea una API Integration di tipo git_https_api chiamata GIT_HACKATHON_INTEGRATION
che permetta di accedere a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
e un Git Repository in RETAIL_COPILOT.PUBLIC chiamato HACKATHON_REPO che punta a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
Non richiede autenticazione (repository pubblico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY RETAIL_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @RETAIL_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/retail/;
```

**[CoCo]** Tabelle Bronze
```
In RETAIL_COPILOT.BRONZE crea le seguenti tabelle con colonna SRC VARIANT
e caricale dal repo Git con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS da data/retail/customers.json
- RAW_ORDERS da data/retail/orders.json
- RAW_SUPPORT_CASES da data/retail/support_cases.json
- RAW_INVENTORY da data/retail/inventory_movements.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_CUSTOMERS; -- atteso: 350
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_ORDERS; -- atteso: 10,000
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_SUPPORT_CASES; -- atteso: 750
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_INVENTORY; -- atteso: 2,000
```

---

## FASE 2 — Silver (dbt)

**[CoCo]** Inizializza dbt
```
Crea un progetto dbt chiamato retail_copilot connesso a RETAIL_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con le tabelle Bronze: RAW_CUSTOMERS, RAW_ORDERS, RAW_SUPPORT_CASES, RAW_INVENTORY
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Crea models/staging/stg_customers.sql che esegua il flatten di RAW_CUSTOMERS.
Campi: clienti (customer_id, full_name, region, loyalty_tier, channel_preference, avg_order_value_eur, total_lifetime_value_eur, preferred_category, status)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_ORDERS
```
Crea models/staging/stg_orders.sql che esegua il flatten di RAW_ORDERS.
Campi: ordini (order_id, customer_id, order_type, total_amount_eur, items_count, store_id, payment_method, product_category)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_SUPPORT_CASES
```
Crea models/staging/stg_support_cases.sql che esegua il flatten di RAW_SUPPORT_CASES.
Campi: casi di supporto (case_id, customer_id, category, priority, description, sla_breached, satisfaction_score)
Materializza come table in SILVER.
```

**[CoCo]** Silver: STG_INVENTORY
```
Crea models/staging/stg_inventory.sql che esegua il flatten di RAW_INVENTORY.
Campi: movimenti di inventario (movement_id, product_id, store_id, movement_type, quantity, unit_cost_eur, product_category)
Materializza come table in SILVER.
```

**[CoCo]** Esegui Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_CUSTOMERS; -- atteso: 350
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_ORDERS; -- atteso: 10,000
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_SUPPORT_CASES; -- atteso: 750
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_INVENTORY; -- atteso: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_360
```
Crea models/marts/customer_360.sql:
Vista 360 del cliente retail: dati base + total_orders + total_spent_eur + avg_order_value + total_returns + open_cases + avg_satisfaction + loyalty_tier + days_since_last_purchase. JOIN di tutte le Silver.
Materializza come table in GOLD.
```

**[CoCo]** Gold: SALES_PERFORMANCE
```
Crea models/marts/sales_performance.sql:
Performance di vendita per store_id, product_category e giorno: total_orders, total_revenue_eur, avg_basket_size, return_rate, distinct_customers, top_payment_method.
Materializza come table in GOLD.
```

**[CoCo]** Gold: CHURN_RISK
```
Crea models/marts/churn_risk.sql:
Rischio di churn per cliente: total_lifetime_value_eur + days_since_last_purchase + support_cases_last_90d + sla_breaches + return_rate + churn_score (0-100).
Materializza come table in GOLD.
```

**[CoCo]** Esegui Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.GOLD.CUSTOMER_360;
SELECT COUNT(*) FROM RETAIL_COPILOT.GOLD.SALES_PERFORMANCE;
SELECT COUNT(*) FROM RETAIL_COPILOT.GOLD.CHURN_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Caso 1: Classificazione dei casi di supporto
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella RETAIL_COPILOT.SILVER.STG_SUPPORT_CASES.
Analizza il campo "description" e restituisce un JSON con: tipo_incidente (reso|spedizione|prodotto|fatturazione|fedelta|account|tecnologia), urgenza (critica|alta|media|bassa), dipartimento (logistica|assistenza_clienti|fatturazione|marketing|IT), impatto_cliente (alto|medio|basso), riassunto_breve (max 15 parole)
Prompt di sistema: "Sei un sistema di classificazione degli incidenti di un'azienda retail."
Crea la vista RETAIL_COPILOT.AI.VW_CASE_CLASSIFICATION. Limita a 50 record.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_CASE_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Riassunto esecutivo del cliente
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella RETAIL_COPILOT.GOLD.CUSTOMER_360.
Genera un riassunto esecutivo in italiano (3-4 frasi) con i campi: full_name, loyalty_tier, region, total_orders, total_spent_eur, avg_order_value, preferred_category, days_since_last_purchase, open_cases
Prompt di sistema: "Sei un analista CRM di un'azienda retail. Genera un riassunto esecutivo in italiano (3-4 frasi) del profilo d'acquisto di questo cliente."
Crea la vista RETAIL_COPILOT.AI.VW_CUSTOMER_EXECUTIVE_SUMMARY. Limita a 20 record.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_CUSTOMER_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Raccomandazione personalizzata di offerta
```
Scrivi una query SQL che usi SNOWFLAKE.CORTEX.COMPLETE con il modello 'llama3.1-70b'
sulla tabella RETAIL_COPILOT.GOLD.CHURN_RISK.
Restituisce un JSON con: segmento_comportamentale (fedele|a_rischio|dormiente|nuovo), offerta_principale (sconto o prodotto), canale_contatto (email|push|sms|posta), momento_ottimale (giorno/ora), messaggio_personalizzato
Prompt di sistema: "Sei un esperto di marketing personalizzato retail. Analizza questo cliente e restituisci SOLO un JSON valido."
Filtra WHERE churn_score >= 30.
Crea la vista RETAIL_COPILOT.AI.VW_PERSONALIZATION_RECOMMENDATIONS. Limita a 25 record.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_PERSONALIZATION_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View in RETAIL_COPILOT.GOLD chiamata SV_RETAIL_COPILOT
sulle tabelle Silver e Gold per query in linguaggio naturale.
Contesto del business: I dati coprono clienti retail, ordini/acquisti, casi di supporto/resi e inventario. Catena retail in Italia.
Includi dimensioni con commenti in italiano, metriche con sinonimi
e istruzioni AI_SQL_GENERATION con il contesto del business.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[SQL]**
```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.RETAIL_COPILOT_AGENT
  COMMENT = 'Copilot di Retail e Beni di Consumo'
  PROFILE = '{"display_name": "Retail Customer Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Sei un assistente di azienda retail/commerciale in Italia. Rispondi sempre in italiano.",
      "response": "Rispondi in modo chiaro e conciso in italiano. Usa il formato tabella quando appropriato."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "retail_analyst",
        "description": "I dati coprono clienti retail, ordini/acquisti, casi di supporto/resi e inventario. Catena retail in Italia."
      }
    }],
    "tool_resources": {
      "retail_analyst": {
        "semantic_view": "RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.RETAIL_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Apri **Snowsight > AI & ML > Snowflake Intelligence** e cerca l'agente.
