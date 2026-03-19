# Guia del Hackathon: Telco Operations Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos  
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE  
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent

---

## Antes de empezar

### Verificar acceso
Abre Cortex Code y comprueba que puedes conectar con las credenciales de tu equipo.
Ejecuta esto directamente en CoCo para confirmar:

```
Ejecuta: SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```

### Datos disponibles

Los 4 ficheros JSON estan en un repositorio Git de GitHub que conectaremos a Snowflake:

```
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon/
  └── data/
        ├── customers.json        (350 clientes)
        ├── network_events.json   (10,000 eventos de red)
        ├── tickets.json          (750 tickets de soporte)
        └── billing_usage.json    (2,000 registros de facturacion)
```

Una vez creada la integracion Git en Snowflake, los ficheros estaran disponibles en:
```
@TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/
```

### Arquitectura objetivo

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
         │
   ┌─────▼─────────────────────────────────────────────┐
   │  BRONZE (raw JSON en tablas VARIANT)               │
   │  ↓                                                 │
   │  SILVER (normalizado, flatten, cast, dedup)        │
   │  ↓                                                 │
   │  GOLD (KPIs, metricas, vistas analiticas)          │
   │  ↓                                                 │
   │  AI (3 casos de uso con Cortex)                    │
   │  ↓                                                 │
   │  SEMANTIC VIEW (modelo semantico para NL2SQL)      │
   │  ↓                                                 │
   │  AGENT (Copilot en Snowflake Intelligence)         │
   └────────────────────────────────────────────────────┘
```

### Roles sugeridos en el equipo (5 personas)

| Rol | Que hace | Cuando |
|-----|----------|--------|
| **Ingeniero de ingesta** | Conecta el repo Git y carga los 4 JSON a Bronze | Fase 1 |
| **Ingeniero dbt 1** | Modelos Silver (flatten + cast) | Fase 2 |
| **Ingeniero dbt 2** | Modelos Gold (KPIs + metricas) | Fase 2 |
| **Ingeniero AI** | 3 prompts Cortex + Semantic View + Agent | Fase 3-5 |
| **Demo lead** | Prepara la presentacion desde el minuto 60 | Todo |

---

## FASE 1: BRONZE — Ingesta de JSON via Git Integration (25 minutos)

> **Objetivo:** Conectar el repositorio Git de GitHub a Snowflake y cargar los 4 ficheros JSON como tablas raw en Bronze.

### Paso 1.1 — Crear la base de datos y esquemas

Copia y pega este prompt en Cortex Code:

```
Crea una base de datos llamada TELCO_COPILOT con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse SNOWADHOC.
```

> **Resultado esperado:** CoCo generara el SQL con CREATE DATABASE y CREATE SCHEMA. Ejecutalo.

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon

Crea una API Integration de tipo git_https_api que permita acceder a 
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en TELCO_COPILOT.PUBLIC llamado HACKATHON_REPO 
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/Snowflake-Spain-SE-Demos-Sandbox')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git';
```

> **Nota:** Si no tienes permisos para crear la API Integration, pide al mentor que la ejecute
> con un rol ACCOUNTADMIN y te otorgue USAGE sobre ella.

### Paso 1.3 — Sincronizar el repositorio y verificar los ficheros

```
Sincroniza el repositorio Git TELCO_COPILOT.PUBLIC.HACKATHON_REPO con ALTER GIT REPOSITORY ... FETCH
y luego lista los ficheros disponibles en la rama main con:
LIST @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/
```

> **SQL directo:**

```sql
ALTER GIT REPOSITORY TELCO_COPILOT.PUBLIC.HACKATHON_REPO FETCH;

LIST @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/;
```

> **Resultado esperado:** Deberias ver los 4 ficheros JSON listados (customers.json, network_events.json, tickets.json, billing_usage.json).

### Paso 1.4 — Crear el file format para JSON

```
En la base de datos TELCO_COPILOT, esquema BRONZE, crea un file format de tipo JSON 
llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE para poder leer arrays de objetos JSON.
```

### Paso 1.5 — Crear las tablas Bronze y cargar desde el repositorio Git

```
Crea 4 tablas en el esquema TELCO_COPILOT.BRONZE y carga los datos JSON 
desde el repositorio Git @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/
usando el file format JSON_FORMAT.

Las tablas deben ser:
- RAW_CUSTOMERS: carga desde @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/customers.json
- RAW_NETWORK_EVENTS: carga desde @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/network_events.json
- RAW_TICKETS: carga desde @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/tickets.json
- RAW_BILLING_USAGE: carga desde @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/billing_usage.json

Cada tabla debe tener:
- columna SRC de tipo VARIANT con el objeto JSON
- columna LOAD_TS con CURRENT_TIMESTAMP()
- columna FILENAME con METADATA$FILENAME

Usa COPY INTO para cargar los datos.
```

> **Para ir mas rapido**, si CoCo tarda en generar, copia y ejecuta directamente:

```sql
USE DATABASE TELCO_COPILOT;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_CUSTOMERS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_CUSTOMERS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/customers.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_NETWORK_EVENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_NETWORK_EVENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/network_events.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_TICKETS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_TICKETS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/tickets.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_BILLING_USAGE (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_BILLING_USAGE (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @TELCO_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/billing_usage.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en TELCO_COPILOT.BRONZE
y muestra una fila de ejemplo de cada una para verificar que el JSON se cargo correctamente.
```

> **Checkpoint:** Deberias ver:
> - RAW_CUSTOMERS: 350 filas
> - RAW_NETWORK_EVENTS: 10,000 filas
> - RAW_TICKETS: 750 filas
> - RAW_BILLING_USAGE: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado telco_copilot que conecte a la base de datos TELCO_COPILOT.
El proyecto debe tener la siguiente estructura de carpetas:
- models/staging/ (para los modelos Silver)
- models/marts/ (para los modelos Gold)

El profile debe usar el warehouse SNOWADHOC, la base de datos TELCO_COPILOT 
y el esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Modelos Silver: Sources

```
En el proyecto dbt telco_copilot, crea un fichero models/staging/sources.yml 
que defina las 4 tablas Bronze como sources:
- database: TELCO_COPILOT
- schema: BRONZE
- tables: RAW_CUSTOMERS, RAW_NETWORK_EVENTS, RAW_TICKETS, RAW_BILLING_USAGE
```

### Paso 2.3 — Modelo Silver: stg_customers

```
Crea un modelo dbt models/staging/stg_customers.sql que haga flatten del JSON 
de la tabla RAW_CUSTOMERS (source Bronze). Extrae estos campos del VARIANT SRC:

- customer_id (VARCHAR)
- company_name (VARCHAR)
- contact_name (VARCHAR)
- email (VARCHAR)
- region (VARCHAR)
- segment (VARCHAR)
- industry (VARCHAR)
- plan (VARCHAR)
- monthly_revenue_eur (NUMBER(10,2))
- contract_start (TIMESTAMP)
- contract_months (INTEGER)
- status (VARCHAR)
- nps_score (INTEGER)
- assigned_account_manager (VARCHAR)

Materializa como table en el esquema SILVER.
```

### Paso 2.4 — Modelo Silver: stg_network_events

```
Crea un modelo dbt models/staging/stg_network_events.sql que extraiga del JSON 
RAW_NETWORK_EVENTS estos campos:

- event_id (VARCHAR)
- customer_id (VARCHAR)
- timestamp (TIMESTAMP)
- event_type (VARCHAR)
- severity (VARCHAR)
- cell_tower_id (VARCHAR)
- device_type (VARCHAR)
- duration_ms (INTEGER)
- metadata (VARIANT) — mantener como VARIANT para consultas flexibles

Materializa como table en el esquema SILVER.
```

### Paso 2.5 — Modelo Silver: stg_tickets

```
Crea un modelo dbt models/staging/stg_tickets.sql que extraiga del JSON RAW_TICKETS:

- ticket_id (VARCHAR)
- customer_id (VARCHAR)
- created_at (TIMESTAMP)
- category (VARCHAR)
- priority (VARCHAR)
- status (VARCHAR)
- channel (VARCHAR)
- subject (VARCHAR)
- description (VARCHAR)
- assigned_agent (VARCHAR)
- sla_hours (INTEGER)
- sla_breached (BOOLEAN)
- resolved_at (TIMESTAMP)
- resolution_notes (VARCHAR)
- satisfaction_score (INTEGER)

Materializa como table en el esquema SILVER.
```

### Paso 2.6 — Modelo Silver: stg_billing_usage

```
Crea un modelo dbt models/staging/stg_billing_usage.sql que extraiga del JSON 
RAW_BILLING_USAGE:

- billing_id (VARCHAR)
- customer_id (VARCHAR)
- period_start (TIMESTAMP)
- period_end (TIMESTAMP)
- service_type (VARCHAR)
- usage (VARIANT) — mantener como VARIANT
- amount_eur (NUMBER(10,2))
- currency (VARCHAR)
- payment_status (VARCHAR)
- invoice_id (VARCHAR)

Materializa como table en el esquema SILVER.
```

### Paso 2.7 — Ejecutar modelos Silver

```
Ejecuta los modelos de staging del proyecto dbt telco_copilot: dbt run --select staging
```

> **Checkpoint:** 4 tablas en TELCO_COPILOT.SILVER con datos tipados y limpios.

### Paso 2.8 — Tests Silver

```
Anade tests dbt al proyecto telco_copilot para los modelos Silver:
- stg_customers: unique y not_null en customer_id
- stg_network_events: unique y not_null en event_id, not_null en customer_id
- stg_tickets: unique y not_null en ticket_id, not_null en customer_id
- stg_billing_usage: unique y not_null en billing_id

Crea el fichero models/staging/schema.yml con estos tests y ejecuta dbt test.
```

### Paso 2.9 — Modelo Gold: customer_360

```
Crea un modelo dbt models/marts/customer_360.sql que genere una vista 360 de cada cliente 
haciendo JOIN de las tablas Silver. Para cada customer_id calcula:

- Datos basicos del cliente (de stg_customers)
- total_tickets: numero total de tickets
- open_tickets: tickets con status 'Abierto' o 'En progreso'
- avg_satisfaction: media de satisfaction_score
- sla_breach_rate: porcentaje de tickets con sla_breached = true
- total_network_events: numero de eventos de red
- critical_events: eventos con severity = 'critical'
- total_billed_eur: suma de amount_eur en billing
- distinct_services: numero de service_types distintos

Materializa como table en el esquema GOLD.
```

### Paso 2.10 — Modelo Gold: network_health

```
Crea un modelo dbt models/marts/network_health.sql que resuma la salud de red 
por cell_tower_id y dia. Para cada torre y fecha calcula:

- total_events
- critical_events
- high_events
- avg_duration_ms
- distinct_customers_affected
- top_event_type (el event_type mas frecuente)

Materializa como table en el esquema GOLD.
```

### Paso 2.11 — Modelo Gold: revenue_at_risk

```
Crea un modelo dbt models/marts/revenue_at_risk.sql que identifique el revenue en riesgo.
Cruza stg_customers con stg_tickets y stg_billing_usage para calcular por cliente:

- monthly_revenue_eur (del cliente)
- status del cliente
- total_tickets_last_90d
- sla_breaches_last_90d
- total_billed_last_90d
- unpaid_amount (billing con payment_status = 'vencido' o 'en_disputa')
- risk_score: un score simple de 0-100 basado en:
    - +30 si status = 'En riesgo'
    - +20 si sla_breaches > 2
    - +20 si open_tickets > 3
    - +15 si unpaid_amount > 0
    - +15 si nps_score < 5

Materializa como table en el esquema GOLD. Ordena por risk_score DESC.
```

### Paso 2.12 — Ejecutar y validar Gold

```
Ejecuta todos los modelos del proyecto dbt telco_copilot con dbt run 
y luego dbt test para validar. Muestra el conteo de filas de las 3 tablas Gold.
```

> **Checkpoint:** 3 tablas en TELCO_COPILOT.GOLD listas para la capa AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (30 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando funciones Cortex sobre las capas Silver y Gold.

### Caso de Uso 1: Clasificacion automatica de tickets

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b' 
para clasificar automaticamente los tickets de soporte de la tabla 
TELCO_COPILOT.SILVER.STG_TICKETS.

Para cada ticket, el LLM debe analizar el campo "description" y devolver un JSON con:
- clasificacion_tecnica: una de [conectividad, hardware, software, cobertura, facturacion, comercial]
- severidad_estimada: una de [critica, alta, media, baja]
- departamento_responsable: una de [ingenieria_red, soporte_nivel2, facturacion, comercial, campo]
- resumen_corto: una frase de maximo 15 palabras resumiendo el problema

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de tickets de soporte de una empresa de telecomunicaciones.
Analiza la descripcion del ticket y devuelve SOLO un JSON valido con los campos: 
clasificacion_tecnica, severidad_estimada, departamento_responsable, resumen_corto."

Limita a 20 tickets como ejemplo (usa LIMIT 20) y crea una vista en el esquema 
TELCO_COPILOT.AI llamada VW_TICKET_CLASSIFICATION.
```

> **Para ir mas rapido:** si CoCo tarda en generar, puedes copiar y ejecutar directamente:

```sql
CREATE OR REPLACE VIEW TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION AS
SELECT 
    ticket_id,
    customer_id,
    category,
    priority,
    description,
    SNOWFLAKE.CORTEX.COMPLETE(
        'llama3.1-70b',
        CONCAT(
            'Eres un sistema de clasificacion de tickets de soporte de una empresa de telecomunicaciones. ',
            'Analiza la siguiente descripcion y devuelve SOLO un JSON valido con estos campos: ',
            'clasificacion_tecnica (conectividad|hardware|software|cobertura|facturacion|comercial), ',
            'severidad_estimada (critica|alta|media|baja), ',
            'departamento_responsable (ingenieria_red|soporte_nivel2|facturacion|comercial|campo), ',
            'resumen_corto (max 15 palabras). ',
            'Descripcion del ticket: ', description
        )
    ) AS ai_classification
FROM TELCO_COPILOT.SILVER.STG_TICKETS
LIMIT 50;
```

### Caso de Uso 2: Resumen ejecutivo de cuenta

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que genere un resumen ejecutivo para cada cliente usando 
SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'.

Usa la tabla TELCO_COPILOT.GOLD.CUSTOMER_360 y construye un prompt que incluya 
los datos del cliente: company_name, plan, region, segment, monthly_revenue_eur, 
total_tickets, open_tickets, avg_satisfaction, sla_breach_rate, critical_events, 
total_billed_eur.

El LLM debe generar un resumen ejecutivo en espanol de 3-4 frases que cubra:
1. Perfil del cliente y valor economico
2. Estado operativo (tickets, SLA, eventos criticos)
3. Nivel de satisfaccion y riesgo

Crea una vista en TELCO_COPILOT.AI llamada VW_ACCOUNT_EXECUTIVE_SUMMARY.
Limita a los 20 clientes con mayor monthly_revenue_eur.
```

> **SQL directo:**

```sql
CREATE OR REPLACE VIEW TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY AS
SELECT 
    customer_id,
    company_name,
    plan,
    region,
    monthly_revenue_eur,
    total_tickets,
    critical_events,
    SNOWFLAKE.CORTEX.COMPLETE(
        'llama3.1-70b',
        CONCAT(
            'Eres un analista de cuentas de una empresa de telecomunicaciones. ',
            'Genera un resumen ejecutivo en espanol (3-4 frases) de este cliente. ',
            'Datos: Empresa: ', company_name,
            ', Plan: ', plan,
            ', Region: ', region,
            ', Segmento: ', segment,
            ', Revenue mensual: ', monthly_revenue_eur::VARCHAR, ' EUR',
            ', Tickets totales: ', total_tickets::VARCHAR,
            ', Tickets abiertos: ', open_tickets::VARCHAR,
            ', Satisfaccion media: ', COALESCE(avg_satisfaction::VARCHAR, 'N/A'),
            ', Tasa incumplimiento SLA: ', ROUND(sla_breach_rate, 1)::VARCHAR, '%',
            ', Eventos criticos de red: ', critical_events::VARCHAR,
            ', Facturacion total: ', total_billed_eur::VARCHAR, ' EUR',
            '. Cubre: 1) Perfil y valor economico, 2) Estado operativo, 3) Riesgo y satisfaccion.'
        )
    ) AS executive_summary
FROM TELCO_COPILOT.GOLD.CUSTOMER_360
ORDER BY monthly_revenue_eur DESC
LIMIT 20;
```

### Caso de Uso 3: Recomendacion de acciones de retencion

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b' 
para recomendar acciones de retencion personalizadas.

Usa la tabla TELCO_COPILOT.GOLD.REVENUE_AT_RISK y filtra clientes con risk_score >= 30.

El prompt debe incluir los datos del cliente y pedir al LLM que devuelva un JSON con:
- nivel_urgencia: alta/media/baja
- accion_principal: la accion mas importante a tomar
- acciones_complementarias: lista de 2-3 acciones adicionales
- oferta_sugerida: una propuesta comercial concreta
- canal_contacto: el mejor canal para contactar (telefono/email/visita_presencial)
- mensaje_personalizado: un mensaje corto para el account manager

Crea una vista en TELCO_COPILOT.AI llamada VW_RETENTION_RECOMMENDATIONS.
```

> **SQL directo:**

```sql
CREATE OR REPLACE VIEW TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS AS
SELECT 
    r.customer_id,
    c.company_name,
    c.plan,
    c.region,
    r.monthly_revenue_eur,
    r.risk_score,
    r.total_tickets_last_90d,
    r.sla_breaches_last_90d,
    r.unpaid_amount,
    SNOWFLAKE.CORTEX.COMPLETE(
        'llama3.1-70b',
        CONCAT(
            'Eres un experto en retencion de clientes de telecomunicaciones. ',
            'Analiza este cliente en riesgo y devuelve SOLO un JSON valido con: ',
            'nivel_urgencia (alta|media|baja), accion_principal (string), ',
            'acciones_complementarias (array de 2-3 strings), ',
            'oferta_sugerida (string con propuesta comercial concreta), ',
            'canal_contacto (telefono|email|visita_presencial), ',
            'mensaje_personalizado (string corto para el account manager). ',
            'Datos del cliente: Empresa: ', c.company_name,
            ', Plan: ', c.plan,
            ', Region: ', c.region,
            ', Segmento: ', c.segment,
            ', Revenue mensual: ', r.monthly_revenue_eur::VARCHAR, ' EUR',
            ', Risk score: ', r.risk_score::VARCHAR, '/100',
            ', Tickets ultimos 90d: ', r.total_tickets_last_90d::VARCHAR,
            ', SLA breaches: ', r.sla_breaches_last_90d::VARCHAR,
            ', Importe impagado: ', COALESCE(r.unpaid_amount::VARCHAR, '0'), ' EUR',
            ', Status: ', c.status,
            ', NPS: ', COALESCE(c.nps_score::VARCHAR, 'N/A')
        )
    ) AS retention_recommendation
FROM TELCO_COPILOT.GOLD.REVENUE_AT_RISK r
JOIN TELCO_COPILOT.SILVER.STG_CUSTOMERS c ON r.customer_id = c.customer_id
WHERE r.risk_score >= 30
ORDER BY r.risk_score DESC
LIMIT 25;
```

### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM TELCO_COPILOT.AI.VW_TICKET_CLASSIFICATION LIMIT 3;
- SELECT * FROM TELCO_COPILOT.AI.VW_ACCOUNT_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM TELCO_COPILOT.AI.VW_RETENTION_RECOMMENDATIONS LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en TELCO_COPILOT.AI con resultados coherentes del LLM.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (15 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold que permita hacer preguntas en lenguaje natural y obtener SQL automaticamente via Cortex Analyst.

### Paso 4.1 — Crear la Semantic View

```
Crea una Semantic View en TELCO_COPILOT.GOLD llamada SV_TELCO_COPILOT 
que conecte las tablas Gold y Silver para permitir consultas en lenguaje natural.

Las tablas logicas deben ser:
- customers: TELCO_COPILOT.SILVER.STG_CUSTOMERS (primary key: customer_id)
- tickets: TELCO_COPILOT.SILVER.STG_TICKETS (primary key: ticket_id)
- network_events: TELCO_COPILOT.SILVER.STG_NETWORK_EVENTS (primary key: event_id)
- billing: TELCO_COPILOT.SILVER.STG_BILLING_USAGE (primary key: billing_id)
- customer360: TELCO_COPILOT.GOLD.CUSTOMER_360 (primary key: customer_id)
- revenue_risk: TELCO_COPILOT.GOLD.REVENUE_AT_RISK (primary key: customer_id)

Relaciones:
- tickets.customer_id -> customers.customer_id
- network_events.customer_id -> customers.customer_id
- billing.customer_id -> customers.customer_id
- customer360.customer_id -> customers.customer_id
- revenue_risk.customer_id -> customers.customer_id

Dimensiones (con comentarios descriptivos en espanol):
- customers: customer_id, company_name, region, segment, industry, plan, status
- tickets: ticket_id, category, priority, status (como ticket_status), channel, sla_breached
- network_events: event_type, severity, cell_tower_id, device_type
- billing: service_type, payment_status

Metricas:
- customers: monthly_revenue_eur (SUM), nps_score (AVG), total_customers (COUNT)
- tickets: total_tickets (COUNT), avg_satisfaction (AVG de satisfaction_score), 
  sla_breach_count (COUNT con filtro sla_breached = true)
- network_events: total_events (COUNT), avg_duration_ms (AVG)
- billing: total_billed_eur (SUM de amount_eur)
- customer360: total_open_tickets (SUM de open_tickets), total_critical_events (SUM de critical_events)
- revenue_risk: avg_risk_score (AVG de risk_score), revenue_at_risk (SUM de monthly_revenue_eur)

Anade instrucciones AI_SQL_GENERATION para que Cortex Analyst sepa que es una empresa telco
y que las preguntas seran sobre clientes, tickets, eventos de red y facturacion.
```

> **SQL directo:**

```sql
CREATE OR REPLACE SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT
  TABLES (
    customers AS TELCO_COPILOT.SILVER.STG_CUSTOMERS
      PRIMARY KEY (customer_id),
    tickets AS TELCO_COPILOT.SILVER.STG_TICKETS
      PRIMARY KEY (ticket_id),
    network_events AS TELCO_COPILOT.SILVER.STG_NETWORK_EVENTS
      PRIMARY KEY (event_id),
    billing AS TELCO_COPILOT.SILVER.STG_BILLING_USAGE
      PRIMARY KEY (billing_id),
    customer360 AS TELCO_COPILOT.GOLD.CUSTOMER_360
      PRIMARY KEY (customer_id),
    revenue_risk AS TELCO_COPILOT.GOLD.REVENUE_AT_RISK
      PRIMARY KEY (customer_id)
  )
  RELATIONSHIPS (
    tickets(customer_id) REFERENCES customers,
    network_events(customer_id) REFERENCES customers,
    billing(customer_id) REFERENCES customers,
    customer360(customer_id) REFERENCES customers,
    revenue_risk(customer_id) REFERENCES customers
  )
  DIMENSIONS (
    -- Dimensiones de clientes
    customers.customer_id AS customer_id
      COMMENT = 'Identificador unico del cliente',
    customers.company_name AS company_name
      COMMENT = 'Nombre de la empresa cliente',
    customers.region AS region
      WITH SYNONYMS = ('ciudad', 'zona', 'ubicacion')
      COMMENT = 'Region geografica del cliente (Madrid, Barcelona, Valencia, etc.)',
    customers.segment AS segment
      WITH SYNONYMS = ('tipo de cliente', 'segmento de mercado')
      COMMENT = 'Segmento comercial: PYME, Corporativo, Autonomo, Gran Cuenta',
    customers.industry AS industry
      WITH SYNONYMS = ('sector', 'industria', 'vertical')
      COMMENT = 'Sector de actividad del cliente',
    customers.plan AS plan
      WITH SYNONYMS = ('tarifa', 'producto', 'plan contratado')
      COMMENT = 'Plan contratado: Basico, Estandar, Premium, Empresa',
    customers.status AS customer_status
      WITH SYNONYMS = ('estado del cliente')
      COMMENT = 'Estado del cliente: Activo, En riesgo, Churned',

    -- Dimensiones de tickets
    tickets.ticket_id AS ticket_id
      COMMENT = 'Identificador unico del ticket de soporte',
    tickets.category AS ticket_category
      WITH SYNONYMS = ('tipo de incidencia', 'categoria del ticket')
      COMMENT = 'Categoria del ticket: Red, Facturacion, Servicio, Hardware, Cobertura, Roaming, Velocidad',
    tickets.priority AS ticket_priority
      WITH SYNONYMS = ('urgencia', 'prioridad del ticket')
      COMMENT = 'Prioridad: baja, media, alta, urgente',
    tickets.status AS ticket_status
      WITH SYNONYMS = ('estado del ticket')
      COMMENT = 'Estado: Abierto, En progreso, Resuelto, Cerrado, Escalado',
    tickets.channel AS ticket_channel
      WITH SYNONYMS = ('canal de contacto', 'via de apertura')
      COMMENT = 'Canal por el que se abrio el ticket: telefono, email, chat, app_movil, portal_web',
    tickets.sla_breached AS sla_breached
      COMMENT = 'Indica si el ticket incumplio el SLA (true/false)',

    -- Dimensiones de eventos de red
    network_events.event_type AS event_type
      WITH SYNONYMS = ('tipo de evento', 'tipo de incidencia de red')
      COMMENT = 'Tipo de evento de red: call_drop, latency_spike, packet_loss, handover_fail, etc.',
    network_events.severity AS event_severity
      WITH SYNONYMS = ('gravedad', 'criticidad')
      COMMENT = 'Severidad del evento: low, medium, high, critical',
    network_events.cell_tower_id AS cell_tower_id
      WITH SYNONYMS = ('torre', 'antena', 'estacion base')
      COMMENT = 'Identificador de la torre de celda afectada',
    network_events.device_type AS device_type
      WITH SYNONYMS = ('tipo de dispositivo', 'equipo')
      COMMENT = 'Tipo de dispositivo: smartphone, router_4g, router_5g, iot_sensor, tablet, laptop_dongle',

    -- Dimensiones de facturacion
    billing.service_type AS service_type
      WITH SYNONYMS = ('tipo de servicio', 'producto facturado')
      COMMENT = 'Tipo de servicio facturado: datos_movil, voz, sms, datos_fijo, centralita_virtual, iot, roaming',
    billing.payment_status AS payment_status
      WITH SYNONYMS = ('estado del pago', 'estado de factura')
      COMMENT = 'Estado del pago: pagado, pendiente, vencido, en_disputa'
  )
  METRICS (
    -- Metricas de clientes
    customers.total_customers AS COUNT(customer_id)
      WITH SYNONYMS = ('numero de clientes', 'cantidad de clientes', 'cuantos clientes')
      COMMENT = 'Numero total de clientes',
    customers.sum_monthly_revenue AS SUM(monthly_revenue_eur)
      WITH SYNONYMS = ('revenue mensual', 'ingresos mensuales', 'facturacion mensual')
      COMMENT = 'Suma del revenue mensual de los clientes en EUR',
    customers.avg_nps AS AVG(nps_score)
      WITH SYNONYMS = ('satisfaccion media', 'NPS medio', 'nota media')
      COMMENT = 'Puntuacion NPS media de los clientes (0-10)',

    -- Metricas de tickets
    tickets.total_tickets AS COUNT(ticket_id)
      WITH SYNONYMS = ('numero de tickets', 'incidencias totales', 'cuantos tickets')
      COMMENT = 'Numero total de tickets de soporte',
    tickets.avg_satisfaction AS AVG(satisfaction_score)
      WITH SYNONYMS = ('satisfaccion media del ticket', 'nota media de resolucion')
      COMMENT = 'Puntuacion media de satisfaccion tras la resolucion (1-5)',
    tickets.sla_breach_count AS COUNT_IF(sla_breached = TRUE)
      WITH SYNONYMS = ('incumplimientos SLA', 'SLA rotos', 'tickets fuera de SLA')
      COMMENT = 'Numero de tickets que incumplieron el SLA',

    -- Metricas de eventos de red
    network_events.total_events AS COUNT(event_id)
      WITH SYNONYMS = ('eventos de red', 'incidencias de red', 'cuantos eventos')
      COMMENT = 'Numero total de eventos de red',
    network_events.avg_duration_ms AS AVG(duration_ms)
      WITH SYNONYMS = ('duracion media', 'tiempo medio del evento')
      COMMENT = 'Duracion media de los eventos de red en milisegundos',

    -- Metricas de facturacion
    billing.total_billed AS SUM(amount_eur)
      WITH SYNONYMS = ('total facturado', 'importe total', 'facturacion total')
      COMMENT = 'Importe total facturado en EUR',

    -- Metricas de customer360
    customer360.sum_open_tickets AS SUM(open_tickets)
      WITH SYNONYMS = ('tickets abiertos totales')
      COMMENT = 'Suma de tickets abiertos de todos los clientes',
    customer360.sum_critical_events AS SUM(critical_events)
      WITH SYNONYMS = ('eventos criticos totales')
      COMMENT = 'Suma de eventos criticos de red de todos los clientes',

    -- Metricas de revenue at risk
    revenue_risk.avg_risk_score AS AVG(risk_score)
      WITH SYNONYMS = ('riesgo medio', 'score de riesgo medio')
      COMMENT = 'Puntuacion media de riesgo de churn (0-100)',
    revenue_risk.total_revenue_at_risk AS SUM(monthly_revenue_eur)
      WITH SYNONYMS = ('revenue en riesgo', 'ingresos en riesgo', 'dinero en riesgo')
      COMMENT = 'Revenue mensual total de clientes en riesgo en EUR'
  )
  COMMENT = 'Vista semantica del Copilot Telco: clientes, tickets, eventos de red, facturacion y metricas de riesgo'
  AI_SQL_GENERATION 'Esta vista semantica describe los datos operativos de una empresa de telecomunicaciones en Espana. Los datos cubren clientes, tickets de soporte, eventos de red (incidencias tecnicas) y facturacion. Las regiones son ciudades espanolas. Los planes van de Basico a Empresa. Cuando el usuario pregunte por ingresos o revenue, usa las metricas en EUR. Cuando pregunte por riesgo de churn, usa la tabla revenue_risk y el campo risk_score.';
```

### Paso 4.2 — Probar la Semantic View con consultas de ejemplo

Ejecuta estas queries para verificar que la vista semantica funciona:

```sql
-- Consulta 1: Revenue por region
SELECT * FROM SEMANTIC VIEW (
  TELCO_COPILOT.GOLD.SV_TELCO_COPILOT
  DIMENSIONS (customers.region)
  METRICS (customers.sum_monthly_revenue, customers.total_customers)
);

-- Consulta 2: Tickets por categoria y prioridad
SELECT * FROM SEMANTIC VIEW (
  TELCO_COPILOT.GOLD.SV_TELCO_COPILOT
  DIMENSIONS (tickets.ticket_category, tickets.ticket_priority)
  METRICS (tickets.total_tickets, tickets.sla_breach_count)
);

-- Consulta 3: Eventos de red por severidad y tipo
SELECT * FROM SEMANTIC VIEW (
  TELCO_COPILOT.GOLD.SV_TELCO_COPILOT
  DIMENSIONS (network_events.event_severity, network_events.event_type)
  METRICS (network_events.total_events, network_events.avg_duration_ms)
);
```

> **Checkpoint:** Las 3 queries devuelven datos agregados correctamente desde la Semantic View.

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (10 minutos)

> **Objetivo:** Crear un Cortex Agent que use la Semantic View para responder preguntas en lenguaje natural desde Snowflake Intelligence.

### Paso 5.1 — Preparar la base de datos para Snowflake Intelligence

Los agentes de Snowflake Intelligence deben crearse en la base de datos `SNOWFLAKE_INTELLIGENCE`:

```
Crea la base de datos SNOWFLAKE_INTELLIGENCE si no existe, con un esquema AGENTS.
Da permisos de USAGE a PUBLIC en ambos. Luego dame permiso de CREATE AGENT 
en el esquema SNOWFLAKE_INTELLIGENCE.AGENTS para mi rol actual.
```

> **SQL directo:**

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
GRANT USAGE ON DATABASE SNOWFLAKE_INTELLIGENCE TO ROLE PUBLIC;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;
GRANT USAGE ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE PUBLIC;
```

> **Nota:** Si no tienes permisos para crear la base de datos, pide al mentor que ejecute
> estos comandos con un rol ACCOUNTADMIN y te otorgue `CREATE AGENT` en el esquema.

### Paso 5.2 — Crear el Cortex Agent

```
Crea un Cortex Agent en SNOWFLAKE_INTELLIGENCE.AGENTS llamado TELCO_COPILOT_AGENT
que use la Semantic View TELCO_COPILOT.GOLD.SV_TELCO_COPILOT para responder 
preguntas en lenguaje natural sobre los datos de la telco.

El agente debe:
- Usar el modelo claude-4-sonnet para orquestacion
- Tener una herramienta de tipo cortex_analyst_text_to_sql conectada a la Semantic View
- Usar el warehouse SNOWADHOC para ejecutar queries
- Tener instrucciones en espanol explicando que es un asistente de operaciones telco

El display_name debe ser "Telco Operations Copilot".
```

> **SQL directo:**

```sql
CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT
  COMMENT = 'Agente de operaciones telco para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Telco Operations Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una empresa de telecomunicaciones en Espana. Usa la herramienta de analisis de datos para responder preguntas sobre clientes, tickets de soporte, eventos de red, facturacion y riesgo de churn. Responde siempre en espanol. Cuando el usuario pregunte por metricas, usa la herramienta telco_analyst para generar y ejecutar la query SQL adecuada.",
      "response": "Responde de forma clara y concisa en espanol. Cuando muestres datos, anade contexto de negocio. Si detectas algo preocupante (alto riesgo, muchos SLA rotos, etc.), mencionalo proactivamente. Usa formato de tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "telco_analyst",
          "description": "Consulta datos operativos de la telco: clientes, tickets de soporte, eventos de red, facturacion y metricas de riesgo de churn. Permite hacer preguntas en lenguaje natural sobre los datos."
        }
      }
    ],
    "tool_resources": {
      "telco_analyst": {
        "semantic_view": "TELCO_COPILOT.GOLD.SV_TELCO_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "SNOWADHOC"
        },
        "query_timeout": 60
      }
    }
  }
  $$;
```

### Paso 5.3 — Dar acceso al agente

```sql
GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.TELCO_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW TELCO_COPILOT.GOLD.SV_TELCO_COPILOT TO ROLE PUBLIC;
```

### Paso 5.4 — Probar el agente desde Snowflake Intelligence

1. Abre **Snowsight** en el navegador
2. Ve a **AI & ML > Snowflake Intelligence**
3. Busca el agente **"Telco Operations Copilot"**
4. Hazle estas preguntas de prueba:

```
Pregunta 1: "Cuantos clientes tenemos por region?"
Pregunta 2: "Cuales son las 5 categorias de tickets con mas incumplimientos de SLA?"
Pregunta 3: "Cual es el revenue mensual total en riesgo de churn?"
Pregunta 4: "Dame los eventos criticos de red agrupados por tipo de evento"
Pregunta 5: "Que regiones tienen peor satisfaccion media en tickets?"
```

> **Checkpoint:** El agente responde en espanol con datos reales, genera SQL contra la Semantic View y muestra resultados coherentes.

---

## BONUS: Streamlit App (si os sobra tiempo)

Si terminais antes, podeis construir un dashboard Streamlit:

```
Crea una aplicacion Streamlit en Snowflake que se conecte a TELCO_COPILOT y muestre:
1. Pagina principal con KPIs: total clientes, tickets abiertos, revenue at risk, SLA breach rate
2. Tabla interactiva de CUSTOMER_360 con filtros por region y segmento
3. Para cada cliente seleccionado, mostrar su resumen ejecutivo AI y recomendacion de retencion
4. Grafico de barras con los top 10 clientes por risk_score

Usa st.metric para los KPIs, st.dataframe para las tablas y st.bar_chart para el grafico.
```

---

## Tips para la Demo (5 minutos)

### Estructura recomendada

| Minuto | Contenido |
|--------|-----------|
| 0:00 - 0:30 | Presentar al equipo y el enfoque |
| 0:30 - 1:00 | Mostrar el pipeline Bronze > Silver > Gold (queries en Snowsight) |
| 1:00 - 2:30 | Demo de los 3 casos AI en vivo (ejecutar las vistas) |
| 2:30 - 3:30 | Mostrar el agente en Snowflake Intelligence respondiendo preguntas |
| 3:30 - 4:30 | Mostrar Streamlit si lo teneis, o explicar el valor de negocio |
| 4:30 - 5:00 | Conclusiones y que haria el equipo con mas tiempo |

### Frases clave para los jueces

- "Todo lo hemos construido usando solo prompts en Cortex Code"
- "El pipeline procesa X mil registros desde JSON raw hasta insights AI"
- "Nuestro modelo de retencion identifica X clientes en riesgo con Y EUR de revenue"
- "La clasificacion automatica de tickets ahorra X horas de triaje manual al dia"
- "Cualquier usuario de negocio puede preguntar al agente en lenguaje natural desde Snowflake Intelligence"

### Errores comunes a evitar

- No intentar hacer todo perfecto — es mejor tener el pipeline completo que una fase impecable
- No gastar mas de 25 min en Bronze — la ingesta es el medio, no el fin
- No olvidar preparar la demo — asignad a alguien desde el minuto 60
- No ejecutar las vistas AI sin LIMIT — puede tardar mucho con todos los registros
