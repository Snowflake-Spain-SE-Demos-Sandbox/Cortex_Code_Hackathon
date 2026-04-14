# 🛍️ Retail Customer Intelligence Copilot
**DB:** `RETAIL_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada RETAIL_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE RETAIL_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en RETAIL_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY RETAIL_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @RETAIL_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/retail/;
```

**[CoCo]** Tablas Bronze
```
En RETAIL_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_CUSTOMERS desde data/retail/customers.json
- RAW_ORDERS desde data/retail/orders.json
- RAW_SUPPORT_CASES desde data/retail/support_cases.json
- RAW_INVENTORY desde data/retail/inventory_movements.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_CUSTOMERS; -- esperado: 350
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_ORDERS; -- esperado: 10,000
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_SUPPORT_CASES; -- esperado: 750
SELECT COUNT(*) FROM RETAIL_COPILOT.BRONZE.RAW_INVENTORY; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Asegurate de estar en **Projects > Workspaces** en Snowsight para ejecutar dbt.

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado retail_copilot conectado a RETAIL_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_CUSTOMERS, RAW_ORDERS, RAW_SUPPORT_CASES, RAW_INVENTORY
```

**[CoCo]** Silver: STG_CUSTOMERS
```
Crea models/staging/stg_customers.sql que haga flatten de RAW_CUSTOMERS.
Campos: clientes (customer_id, full_name, region, loyalty_tier, channel_preference, avg_order_value_eur, total_lifetime_value_eur, preferred_category, status)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_ORDERS
```
Crea models/staging/stg_orders.sql que haga flatten de RAW_ORDERS.
Campos: pedidos (order_id, customer_id, order_type, total_amount_eur, items_count, store_id, payment_method, product_category)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_SUPPORT_CASES
```
Crea models/staging/stg_support_cases.sql que haga flatten de RAW_SUPPORT_CASES.
Campos: casos de soporte (case_id, customer_id, category, priority, description, sla_breached, satisfaction_score)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_INVENTORY
```
Crea models/staging/stg_inventory.sql que haga flatten de RAW_INVENTORY.
Campos: movimientos de inventario (movement_id, product_id, store_id, movement_type, quantity, unit_cost_eur, product_category)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_CUSTOMERS; -- esperado: 350
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_ORDERS; -- esperado: 10,000
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_SUPPORT_CASES; -- esperado: 750
SELECT COUNT(*) FROM RETAIL_COPILOT.SILVER.STG_INVENTORY; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: CUSTOMER_360
```
Crea models/marts/customer_360.sql:
Vista 360 del cliente retail: datos basicos + total_orders + total_spent_eur + avg_order_value + total_returns + open_cases + avg_satisfaction + loyalty_tier + days_since_last_purchase. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: SALES_PERFORMANCE
```
Crea models/marts/sales_performance.sql:
Performance de ventas por store_id, product_category y dia: total_orders, total_revenue_eur, avg_basket_size, return_rate, distinct_customers, top_payment_method.
Materializa como table en GOLD.
```

**[CoCo]** Gold: CHURN_RISK
```
Crea models/marts/churn_risk.sql:
Riesgo de churn por cliente: total_lifetime_value_eur + days_since_last_purchase + support_cases_last_90d + sla_breaches + return_rate + churn_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
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

**[CoCo]** Caso 1: Clasificacion de casos de soporte
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla RETAIL_COPILOT.SILVER.STG_SUPPORT_CASES.
Analiza el campo "description" y devuelve un JSON con: tipo_incidencia (devolucion|envio|producto|facturacion|fidelidad|cuenta|tecnologia), urgencia (critica|alta|media|baja), departamento (logistica|atencion_cliente|facturacion|marketing|IT), impacto_cliente (alto|medio|bajo), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de incidencias de una empresa de retail."
Crea la vista RETAIL_COPILOT.AI.VW_CASE_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_CASE_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de cliente
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla RETAIL_COPILOT.GOLD.CUSTOMER_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: full_name, loyalty_tier, region, total_orders, total_spent_eur, avg_order_value, preferred_category, days_since_last_purchase, open_cases
Prompt del sistema: "Eres un analista de CRM de una empresa retail. Genera un resumen ejecutivo en espanol (3-4 frases) del perfil de compra de este cliente."
Crea la vista RETAIL_COPILOT.AI.VW_CUSTOMER_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_CUSTOMER_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion personalizada de oferta
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla RETAIL_COPILOT.GOLD.CHURN_RISK.
Devuelve un JSON con: segmento_comportamental (fiel|en_riesgo|dormido|nuevo), oferta_principal (descuento o producto), canal_contacto (email|push|sms|correo_postal), momento_optimo (dia/hora), mensaje_personalizado
Prompt del sistema: "Eres un experto en marketing personalizado de retail. Analiza este cliente y devuelve SOLO un JSON valido."
Filtra WHERE churn_score >= 30.
Crea la vista RETAIL_COPILOT.AI.VW_PERSONALIZATION_RECOMMENDATIONS. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM RETAIL_COPILOT.AI.VW_PERSONALIZATION_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en RETAIL_COPILOT.GOLD llamada SV_RETAIL_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren clientes retail, pedidos/compras, casos de soporte/devoluciones e inventario. Cadena de retail en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent en Snowflake Intelligence para Retail y Consumer Goods.
- Crea el objeto Snowflake Intelligence predeterminado si no existe
- Crea la base de datos SNOWFLAKE_INTELLIGENCE y el schema AGENTS
- Crea el agente RETAIL_COPILOT_AGENT conectado a la Semantic View RETAIL_COPILOT.GOLD.SV_RETAIL_COPILOT
  con modelo de orquestacion claude-4-sonnet, respondiendo siempre en espanol
  y contexto de negocio: empresa de retail/comercio en Espana
- Concede acceso al rol PUBLIC al agente y a la Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.RETAIL_COPILOT_AGENT
  COMMENT = 'Copilot de Retail y Consumer Goods'
  PROFILE = '{"display_name": "Retail Customer Intelligence Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de empresa de retail/comercio en Espana. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "retail_analyst",
        "description": "Los datos cubren clientes retail, pedidos/compras, casos de soporte/devoluciones e inventario. Cadena de retail en Espana."
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

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.

---

## Prueba el Agente

Preguntas de ejemplo para validar el agente:

- "¿Cuales son los 10 clientes con mayor riesgo de churn?"
- "¿Que categorias de producto tienen mas devoluciones este mes?"
- "¿Cual es el ticket medio de compra por tier de fidelidad?"
- "¿Que clientes llevan mas de 90 dias sin comprar?"

---

## FASE FINAL — App Streamlit en Snowflake

**[CoCo]**
```
Crea una app Streamlit in Snowflake (SiS) en la base de datos RETAIL_COPILOT, schema AI, llamada RETAIL_APP.
La app debe conectarse a Snowflake con la sesion activa (get_active_session).
Incluye 4 secciones usando st.tabs:
1. KPI — tarjetas con st.metric con los totales y ratios principales de CUSTOMER_360
2. Distribucion — grafico de barras (st.bar_chart) por categoria/region desde SALES_PERFORMANCE
3. Top 20 — tabla interactiva (st.dataframe) ordenada por score de riesgo desde CHURN_RISK
4. Tendencia — grafico de lineas (st.line_chart) con evolucion temporal de los eventos principales
Usa titulos y etiquetas descriptivos en espanol. Aplica un tema oscuro con st.set_page_config.
```
