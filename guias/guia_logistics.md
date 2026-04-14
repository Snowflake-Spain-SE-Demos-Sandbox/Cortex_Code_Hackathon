# 🚚 Logistics & Fleet Copilot
**DB:** `LOGISTICS_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Crea una base de datos llamada LOGISTICS_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE LOGISTICS_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Crea una API Integration de tipo git_https_api llamada GIT_HACKATHON_INTEGRATION
que permita acceder a https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y un Git Repository en LOGISTICS_COPILOT.PUBLIC llamado HACKATHON_REPO apuntando a:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No necesita autenticacion (repo publico).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/;
```

**[CoCo]** Tablas Bronze
```
En LOGISTICS_COPILOT.BRONZE crea las siguientes tablas con columna SRC VARIANT
y cargalas desde el Git repo con COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_FLEET desde data/logistics/fleet_vehicles.json
- RAW_ROUTE_EVENTS desde data/logistics/route_events.json
- RAW_DELIVERY_TICKETS desde data/logistics/delivery_tickets.json
- RAW_SHIPMENTS desde data/logistics/shipments.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_FLEET; -- esperado: 350
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_ROUTE_EVENTS; -- esperado: 10,000
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_DELIVERY_TICKETS; -- esperado: 750
SELECT COUNT(*) FROM LOGISTICS_COPILOT.BRONZE.RAW_SHIPMENTS; -- esperado: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Nota:** Asegurate de estar en **Projects > Workspaces** en Snowsight para ejecutar dbt.

**[CoCo]** Init dbt
```
Crea un proyecto dbt llamado logistics_copilot conectado a LOGISTICS_COPILOT.
models/staging/ -> esquema SILVER | models/marts/ -> esquema GOLD | warehouse COMPUTE_WH.
Crea models/staging/sources.yml con las tablas Bronze: RAW_FLEET, RAW_ROUTE_EVENTS, RAW_DELIVERY_TICKETS, RAW_SHIPMENTS
```

**[CoCo]** Silver: STG_FLEET
```
Crea models/staging/stg_fleet.sql que haga flatten de RAW_FLEET.
Campos: vehiculos de flota (vehicle_id, plate_number, type, brand, base_location, max_payload_kg, fuel_type, km_total, status)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_ROUTE_EVENTS
```
Crea models/staging/stg_route_events.sql que haga flatten de RAW_ROUTE_EVENTS.
Campos: eventos de ruta (event_id, vehicle_id, shipment_id, event_type, location, gps_lat, gps_lon, speed_kmh)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_DELIVERY_TICKETS
```
Crea models/staging/stg_delivery_tickets.sql que haga flatten de RAW_DELIVERY_TICKETS.
Campos: tickets de incidencias (ticket_id, shipment_id, vehicle_id, category, priority, description, sla_breached, estimated_cost_eur)
Materializa como table en SILVER.
```

**[CoCo]** Silver: STG_SHIPMENTS
```
Crea models/staging/stg_shipments.sql que haga flatten de RAW_SHIPMENTS.
Campos: envios (shipment_id, client_name, origin, destination, service_type, amount_eur, weight_kg, status)
Materializa como table en SILVER.
```

**[CoCo]** Ejecutar Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_FLEET; -- esperado: 350
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_ROUTE_EVENTS; -- esperado: 10,000
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_DELIVERY_TICKETS; -- esperado: 750
SELECT COUNT(*) FROM LOGISTICS_COPILOT.SILVER.STG_SHIPMENTS; -- esperado: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: VEHICLE_360
```
Crea models/marts/vehicle_360.sql:
Vista 360 de vehiculo: datos basicos + total_shipments + total_route_events + incidents_count + total_km_last_90d + total_revenue_eur + avg_delivery_time_hours + days_until_maintenance. JOIN de todas las Silver.
Materializa como table en GOLD.
```

**[CoCo]** Gold: ROUTE_HEALTH
```
Crea models/marts/route_health.sql:
Salud de rutas por origin, destination y dia: total_shipments, total_events, delay_events, avg_delivery_hours, on_time_rate, total_revenue_eur, distinct_vehicles.
Materializa como table en GOLD.
```

**[CoCo]** Gold: DELIVERY_RISK
```
Crea models/marts/delivery_risk.sql:
Riesgo de incidencias por vehiculo/ruta: pending_tickets + sla_breaches_last_90d + delay_count + damage_claims + km_since_maintenance + delivery_risk_score (0-100).
Materializa como table en GOLD.
```

**[CoCo]** Ejecutar Gold
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

**[CoCo]** Caso 1: Clasificacion de incidencias de entrega
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla LOGISTICS_COPILOT.SILVER.STG_DELIVERY_TICKETS.
Analiza el campo "description" y devuelve un JSON con: tipo_incidencia (retraso|dano|direccion|vehiculo|documentacion|rechazo|temperatura|peso), urgencia (critica|alta|media|baja), departamento (operaciones|flota|calidad|atencion_cliente|legal), impacto_servicio (alto|medio|bajo), resumen_corto (max 15 palabras)
Prompt del sistema: "Eres un sistema de clasificacion de incidencias de una empresa de transporte y logistica."
Crea la vista LOGISTICS_COPILOT.AI.VW_DELIVERY_CLASSIFICATION. Limita a 50 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_DELIVERY_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Caso 2: Resumen ejecutivo de vehiculo/flota
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla LOGISTICS_COPILOT.GOLD.VEHICLE_360.
Genera un resumen ejecutivo en espanol (3-4 frases) con los campos: plate_number, type, base_location, km_total, total_shipments, incidents_count, total_revenue_eur, days_until_maintenance
Prompt del sistema: "Eres un gestor de flota de transporte. Genera un resumen ejecutivo en espanol (3-4 frases) del estado de este vehiculo."
Crea la vista LOGISTICS_COPILOT.AI.VW_FLEET_EXECUTIVE_SUMMARY. Limita a 20 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_FLEET_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Caso 3: Recomendacion de optimizacion de ruta
```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
sobre la tabla LOGISTICS_COPILOT.GOLD.DELIVERY_RISK.
Devuelve un JSON con: nivel_riesgo (alto|medio|bajo), accion_principal (cambio_ruta|mantenimiento_vehiculo|refuerzo_capacidad|formacion_conductor), mejora_estimada_pct, plazo_implementacion_dias, impacto_coste_eur, recomendacion_detallada
Prompt del sistema: "Eres un experto en optimizacion logistica. Analiza esta ruta/vehiculo y devuelve SOLO un JSON valido."
Filtra WHERE delivery_risk_score >= 30.
Crea la vista LOGISTICS_COPILOT.AI.VW_ROUTE_OPTIMIZATION. Limita a 25 registros.
```

**[✓ SQL]**
```sql
SELECT * FROM LOGISTICS_COPILOT.AI.VW_ROUTE_OPTIMIZATION LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Crea una Semantic View en LOGISTICS_COPILOT.GOLD llamada SV_LOGISTICS_COPILOT
sobre las tablas Silver y Gold para consultas en lenguaje natural.
Contexto del negocio: Los datos cubren vehiculos de flota, eventos de ruta, tickets de incidencias de entrega y envios. Empresa de transporte/logistica en Espana.
Incluye dimensiones con comentarios en espanol, metricas con sinonimos
e instrucciones AI_SQL_GENERATION con el contexto de negocio.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Crea un Cortex Agent en Snowflake Intelligence para Transporte, Logistica y Travel.
- Crea el objeto Snowflake Intelligence predeterminado si no existe
- Crea la base de datos SNOWFLAKE_INTELLIGENCE y el schema AGENTS
- Crea el agente LOGISTICS_COPILOT_AGENT conectado a la Semantic View LOGISTICS_COPILOT.GOLD.SV_LOGISTICS_COPILOT
  con modelo de orquestacion claude-4-sonnet, respondiendo siempre en espanol
  y contexto de negocio: empresa de transporte y logistica en Espana
- Concede acceso al rol PUBLIC al agente y a la Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.LOGISTICS_COPILOT_AGENT
  COMMENT = 'Copilot de Transporte, Logistica y Travel'
  PROFILE = '{"display_name": "Logistics & Fleet Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "Eres un asistente de empresa de transporte y logistica en Espana. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "logistics_analyst",
        "description": "Los datos cubren vehiculos de flota, eventos de ruta, tickets de incidencias de entrega y envios. Empresa de transporte/logistica en Espana."
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

> Abre **Snowsight > AI & ML > Snowflake Intelligence** y busca el agente.

---

## Prueba el Agente

Preguntas de ejemplo para validar el agente:

- "¿Que vehiculos de la flota necesitan mantenimiento urgente?"
- "¿Cuales son las rutas con mayor tasa de incidencias y retrasos?"
- "¿Cuantos envios tienen el SLA incumplido esta semana?"
- "¿Cual es el coste estimado de las incidencias abiertas por zona?"

---

## FASE FINAL — App Streamlit en Snowflake

**[CoCo]**
```
Crea una app Streamlit in Snowflake (SiS) en la base de datos LOGISTICS_COPILOT, schema AI, llamada LOGISTICS_APP.
La app debe conectarse a Snowflake con la sesion activa (get_active_session).
Incluye 4 secciones usando st.tabs:
1. KPI — tarjetas con st.metric con los totales y ratios principales de VEHICLE_360
2. Distribucion — grafico de barras (st.bar_chart) por categoria/region desde ROUTE_HEALTH
3. Top 20 — tabla interactiva (st.dataframe) ordenada por score de riesgo desde DELIVERY_RISK
4. Tendencia — grafico de lineas (st.line_chart) con evolucion temporal de los eventos principales
Usa titulos y etiquetas descriptivos en espanol. Aplica un tema oscuro con st.set_page_config.
```
