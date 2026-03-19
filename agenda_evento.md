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
- Ingestar datos semi-estructurados JSON (incidencias, clientes, senales de churn)
- Transformar con patron medallion Bronze / Silver / Gold usando dbt
- Aplicar 3 casos de uso AI con Cortex sobre la capa Gold

---

## Agenda Detallada

| Hora | Duracion | Bloque | Descripcion |
|------|----------|--------|-------------|
| 09:30 - 10:00 | 30 min | **Recepcion y registro** | Acreditacion, entrega de materiales, cafe de bienvenida |
| 10:00 - 10:15 | 15 min | **Bienvenida** | Apertura institucional Snowflake. Presentacion del evento, objetivos y mecanica |
| 10:15 - 10:35 | 20 min | **Keynote: Cortex Code** | Demo en vivo de Cortex Code (CoCo). Mostrar el flujo end-to-end: ingesta JSON, transformacion medallion, AI prompts |
| 10:35 - 10:45 | 10 min | **Presentacion del reto** | Explicacion del dataset telco (JSON), la arquitectura Bronze/Silver/Gold y los 3 casos de uso AI |
| 10:45 - 10:50 | 5 min | **Formacion de equipos** | Asignacion de mesas, entrega de guias por equipo, acceso a entornos Snowflake |
| 10:50 - 12:20 | 90 min | **Hackathon - Manos a la obra** | Trabajo en equipos. Cada equipo sigue su guia usando Cortex Code como herramienta principal |
| | | *Fase 1 - Bronze (25 min)* | Ingesta de JSON semi-estructurado y creacion de la capa Bronze (raw) |
| | | *Fase 2 - Silver/Gold (35 min)* | Transformacion con dbt: Silver (eventos normalizados) y Gold (KPIs y vistas analiticas) |
| | | *Fase 3 - AI (30 min)* | 3 prompts Cortex AI: clasificacion de tickets, resumen ejecutivo de cuenta, recomendacion de retencion |
| 12:20 - 12:30 | 10 min | **Pausa / Coffee break** | Descanso breve, preparacion de demos |
| 12:30 - 13:00 | 30 min | **Presentaciones de equipos** | Cada equipo presenta su solucion (5 min por equipo) |
| 13:00 - 13:15 | 15 min | **Deliberacion y premios** | Votacion y entrega de premios al mejor proyecto |
| 13:15 - 13:30+ | 15 min+ | **Cierre y networking** | Palabras de cierre, comida tipo catering y networking |

---

## Arquitectura del Reto

```
JSON Semi-estructurado
        |
   [ BRONZE ]  Raw JSON en Snowflake (incidents, customers, usage, churn_signals)
        |
   [ SILVER ]  Eventos normalizados con dbt (flatten, cast, dedup)
        |
   [  GOLD  ]  KPIs y vistas analiticas (metricas por cuenta, SLA, segmentacion)
        |
   [   AI   ]  3 Casos de Uso con Cortex
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
| Bronze | Tablas raw con JSON semi-estructurado cargado |
| Silver | Tablas normalizadas (incidents, customers, usage) con tests dbt |
| Gold | Vistas analiticas con KPIs (SLA, churn score, revenue at risk) |
| AI | 3 prompts Cortex funcionando sobre la capa Gold |
| Demo | Presentacion de 5 min mostrando el pipeline end-to-end |

---

## Criterios de Evaluacion

| Criterio | Peso |
|----------|------|
| Funcionalidad end-to-end (Bronze a AI) | 30% |
| Uso efectivo de Cortex Code | 25% |
| Calidad de los 3 casos AI | 25% |
| Presentacion y storytelling | 20% |

---

## Dataset Telco (JSON)

Ficheros JSON semi-estructurados con:
- **incidents.json**: Tickets de soporte (id, customer_id, timestamp, description, status, priority)
- **customers.json**: Clientes telco (id, name, plan, region, contract_start, monthly_revenue)
- **usage.json**: Consumo de servicios (customer_id, date, data_gb, voice_min, sms_count)
- **churn_signals.json**: Senales de riesgo (customer_id, signal_type, score, detected_at)
