# Build an AI-Powered Telco Operations Copilot
## Hackathon Cortex Code Experience

**Fecha:** 28 de mayo de 2026  
**Horario:** 10:00 - 13:30  
**Lugar:** Oficinas de Snowflake, Madrid  
**Asistentes:** 30 clientes (6 equipos de 5 personas)  
**Catering:** Comida servida a partir de las 13:00

---

## Objetivo

Cada equipo construira una mini plataforma telco end-to-end usando Snowflake Cortex Code como unico IDE:
- Ingestar datos semi-estructurados JSON (customers, network_events, tickets, billing_usage)
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
| 10:15 - 10:35 | 20 min | **Keynote: Cortex Code** | Demo en vivo de Cortex Code (CoCo). Mostrar el flujo end-to-end: ingesta JSON, transformacion medallion, AI prompts, Semantic View y Copilot Agent |
| 10:35 - 10:45 | 10 min | **Presentacion del reto** | Explicacion del dataset telco (JSON), la arquitectura Bronze/Silver/Gold/AI/SV/Agent y los 5 fases del hackathon |
| 10:45 - 10:50 | 5 min | **Formacion de equipos** | Asignacion de mesas, entrega de guias por equipo, acceso a entornos Snowflake |
| 10:50 - 12:20 | 90 min | **Hackathon - Manos a la obra** | Trabajo en equipos. Cada equipo sigue su guia usando Cortex Code como herramienta principal |
| | | *Fase 1 - Bronze (25 min)* | Ingesta de JSON semi-estructurado desde @TELCO.FICHEROS y creacion de la capa Bronze (raw) |
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
@TELCO.FICHEROS (JSON)
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
| Bronze | Tablas raw con JSON semi-estructurado cargado desde @TELCO.FICHEROS |
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

Ficheros JSON semi-estructurados disponibles en el stage `@TELCO.FICHEROS`:
- **customers.json**: 350 clientes telco (customer_id, company_name, plan, region, segment, monthly_revenue_eur, nps_score)
- **network_events.json**: 10,000 eventos de red (event_id, customer_id, event_type, severity, cell_tower_id, duration_ms)
- **tickets.json**: 750 tickets de soporte (ticket_id, customer_id, category, priority, description, sla_breached)
- **billing_usage.json**: 2,000 registros de facturacion (billing_id, customer_id, service_type, amount_eur, payment_status)
