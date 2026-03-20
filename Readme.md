# Build an AI-Powered Telco Operations Copilot
## Hackathon Cortex Code Experience

**Fecha:** 28 de mayo de 2026  
**Horario:** 10:00 - 13:30  
**Lugar:** Oficinas de Snowflake, Madrid  
**Asistentes:** 30 clientes (6 equipos de 5 personas)  
**Catering:** Comida servida a partir de las 13:00

---

## Requisito Previo: Cuenta Trial de Snowflake

Cada participante debe crear su cuenta trial **antes del evento** (o al inicio del hackathon):

1. Ir a **https://signup.snowflake.com/**
2. Rellenar el formulario con email corporativo
3. Seleccionar:
   - **Cloud Provider:** AWS
   - **Edition:** Enterprise
   - **Region:** US West (Oregon)
4. Activar la cuenta desde el email de confirmacion
5. La cuenta incluye **$400 en creditos** y 30 dias de validez

> Los mentores ayudaran a configurar Cortex Code en las cuentas al inicio del hackathon.

---

## Objetivo

Cada equipo construira una mini plataforma telco end-to-end usando Snowflake Cortex Code como unico IDE:
- Conectar un repositorio Git de GitHub a Snowflake e ingestar datos JSON (customers, network_events, tickets, billing_usage)
- Transformar con patron medallion Bronze / Silver / Gold usando dbt
- Aplicar 3 casos de uso AI con Cortex sobre la capa Gold
- Crear una Semantic View para consultas en lenguaje natural (NL2SQL)
- Desplegar un Cortex Agent como Copilot en Snowflake Intelligence

---

## Agenda Detallada

| Hora | Duracion | Bloque | Descripcion |
|------|----------|--------|-------------|
| 09:30 - 10:00 | 30 min | **Recepcion y registro** | Acreditacion, entrega de materiales, cafe de bienvenida |
| 10:00 - 10:15 | 15 min | **Bienvenida** | Apertura institucional Snowflake. Presentacion del evento, objetivos y mecanica |
| 10:15 - 10:35 | 20 min | **Keynote: Cortex Code** | Demo en vivo de Cortex Code (CoCo). Mostrar el flujo end-to-end: Git integration, transformacion medallion, AI prompts, Semantic View y Copilot Agent |
| 10:35 - 10:45 | 10 min | **Presentacion del reto** | Explicacion del dataset telco (GitHub repo), la arquitectura Git > Bronze/Silver/Gold/AI/SV/Agent y las 5 fases del hackathon |
| 10:45 - 10:50 | 5 min | **Formacion de equipos** | Asignacion de mesas, entrega de guias por equipo, acceso a entornos Snowflake |
| 10:50 - 12:20 | 90 min | **Hackathon - Manos a la obra** | Trabajo en equipos. Cada equipo sigue su guia usando Cortex Code como herramienta principal |
| | | *Fase 1 - Bronze (25 min)* | Conectar repo Git de GitHub a Snowflake e ingestar JSON como capa Bronze (raw) |
| | | *Fase 2 - Silver/Gold (35 min)* | Transformacion con dbt: Silver (tablas normalizadas) y Gold (customer_360, network_health, revenue_at_risk) |
| | | *Fase 3 - AI (20 min)* | 3 prompts Cortex AI: clasificacion de tickets, resumen ejecutivo de cuenta, recomendacion de retencion |
| | | *Fase 4 - Semantic View (5 min)* | Crear Semantic View sobre Gold/Silver con dimensiones, metricas e instrucciones NL2SQL |
| | | *Fase 5 - Cortex Agent (5 min)* | Crear Cortex Agent en Snowflake Intelligence conectado a la Semantic View |
| 12:20 - 12:30 | 10 min | **Pausa / Coffee break** | Descanso breve, preparacion de demos |
| 12:30 - 13:00 | 30 min | **Presentaciones de equipos** | Cada equipo presenta su solucion (5 min por equipo) |
| 13:00 - 13:15 | 15 min | **Deliberacion y premios** | Votacion y entrega de premios al mejor proyecto |
| 13:15 - 13:30+ | 15 min+ | **Cierre y networking** | Palabras de cierre, comida tipo catering y networking |

---

## Arquitectura del Reto

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
        |
   [ BRONZE ]  Raw JSON en Snowflake (customers, network_events, tickets, billing_usage)
        |
   [ SILVER ]  Tablas normalizadas con dbt (flatten, cast, dedup)
        |
   [  GOLD  ]  KPIs y vistas analiticas (customer_360, network_health, revenue_at_risk)
        |
   [   AI   ]  3 Casos de Uso con Cortex (clasificacion, resumen, retencion)
        |
   [SEM.VIEW]  Modelo semantico NL2SQL (17 dimensiones, 13 metricas, Cortex Analyst)
        |
   [ AGENT  ]  Copilot en Snowflake Intelligence (consultas en lenguaje natural)
```

---

## Los 3 Casos de Uso AI (todos los equipos)

| # | Caso de Uso | Funcion Cortex | Descripcion |
|---|-------------|----------------|-------------|
| 1 | Clasificacion de tickets | CORTEX.COMPLETE / CLASSIFY | Clasificar incidencias por tipo, severidad y area a partir del texto libre |
| 2 | Resumen ejecutivo de cuenta | CORTEX.SUMMARIZE / COMPLETE | Generar un resumen ejecutivo del estado de un cliente (incidencias, uso, riesgo) |
| 3 | Recomendacion de retencion | CORTEX.COMPLETE | Recomendar acciones de retencion basadas en senales de churn y perfil del cliente |

---

## Entregables por Equipo

| Capa | Entregable |
|------|-----------|
| Bronze | Tablas raw con JSON semi-estructurado cargado desde repositorio Git de GitHub |
| Silver | Tablas normalizadas (stg_customers, stg_network_events, stg_tickets, stg_billing_usage) con tests dbt |
| Gold | Vistas analiticas con KPIs (customer_360, network_health, revenue_at_risk) |
| AI | 3 vistas Cortex AI funcionando sobre la capa Gold |
| Semantic View | Vista semantica con dimensiones, metricas y sinonimos en espanol |
| Agent | Cortex Agent en Snowflake Intelligence respondiendo preguntas en lenguaje natural |
| Demo | Presentacion de 5 min mostrando el pipeline end-to-end |

---

## Criterios de Evaluacion

| Criterio | Peso |
|----------|------|
| Funcionalidad end-to-end (Bronze a Agent) | 30% |
| Uso efectivo de Cortex Code | 20% |
| Calidad de los 3 casos AI | 20% |
| Semantic View + Cortex Agent | 15% |
| Presentacion y storytelling | 15% |

---

## Dataset Telco (JSON)

Ficheros JSON semi-estructurados en el repositorio Git de GitHub:

**Repositorio:** https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon  
**Ruta de datos:** `data/`

- **customers.json**: 350 clientes telco (customer_id, company_name, plan, region, segment, monthly_revenue_eur, nps_score)
- **network_events.json**: 10,000 eventos de red (event_id, customer_id, event_type, severity, cell_tower_id, duration_ms)
- **tickets.json**: 750 tickets de soporte (ticket_id, customer_id, category, priority, description, sla_breached)
- **billing_usage.json**: 2,000 registros de facturacion (billing_id, customer_id, service_type, amount_eur, payment_status)

Durante el hackathon, los equipos conectaran este repositorio a Snowflake via Git Integration y cargaran los datos directamente.

---

## Revision de Costes Post-Hackathon

Al finalizar el hackathon, cada participante puede revisar el consumo de su cuenta trial:

```sql
-- Creditos consumidos por tipo de servicio
SELECT service_type, ROUND(SUM(credits_used), 2) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY 
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY service_type ORDER BY total_credits DESC;

-- Consumo de funciones Cortex AI
SELECT function_name, model_name, SUM(tokens) AS total_tokens, ROUND(SUM(token_credits), 4) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY 
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY function_name, model_name ORDER BY total_credits DESC;
```

> **Nota:** Las vistas de ACCOUNT_USAGE tienen un retraso de hasta 2 horas. Revisad al dia siguiente para datos completos.
