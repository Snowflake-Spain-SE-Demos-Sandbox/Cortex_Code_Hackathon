# Build an AI-Powered Industry Copilot
## Hackathon Cortex Code Experience

**Fecha:** 27 de mayo de 2026
**Horario:** 09:30 - 13:30
**Lugar:** Oficinas de Snowflake, Madrid
**Asistentes:** 30 personas
**Catering:** Comida servida a partir de las 13:30

**Registro:** [https://www.snowflake.com/event/hackathon-cortex-code-experience-madrid-20260527/](https://www.snowflake.com/event/hackathon-cortex-code-experience-madrid-20260527/)

---

## Elige tu Industria

Cada equipo selecciona una industria y trabaja con sus datos especificos:

| # | Industria | Base de datos | Guia | Dataset |
|---|-----------|---------------|------|---------|
| 1 | 📡 **Telecomunicaciones** | `TELCO_COPILOT` | [guia_telco.md](guias/guia_telco.md) | `data/telco/` |
| 2 | 🏦 **Servicios Financieros** | `FSI_COPILOT` | [guia_fsi.md](guias/guia_fsi.md) | `data/fsi/` |
| 3 | 🛍️ **Retail y Consumer Goods** | `RETAIL_COPILOT` | [guia_retail.md](guias/guia_retail.md) | `data/retail/` |
| 4 | 🏭 **Manufacturing e Industria** | `MANUFACTURING_COPILOT` | [guia_manufacturing.md](guias/guia_manufacturing.md) | `data/manufacturing/` |
| 5 | 🏥 **Healthcare y Life Sciences** | `HEALTHCARE_COPILOT` | [guia_healthcare.md](guias/guia_healthcare.md) | `data/healthcare/` |
| 6 | 📺 **Media, Entertainment y Advertising** | `MEDIA_COPILOT` | [guia_media.md](guias/guia_media.md) | `data/media/` |
| 7 | 🚚 **Transporte, Logistica y Travel** | `LOGISTICS_COPILOT` | [guia_logistics.md](guias/guia_logistics.md) | `data/logistics/` |
| 8 | 🌐 **Sector Publico** | `PUBLIC_SECTOR_COPILOT` | [guia_public_sector.md](guias/guia_public_sector.md) | `data/public_sector/` |
| 9 | 💧 **Utilities y Gestion del Agua** | `UTILITIES_COPILOT` | [guia_utilities.md](guias/guia_utilities.md) | `data/utilities/` |

---

## Requisito Previo: Cuenta Trial de Snowflake

Cada participante debe crear su cuenta trial **antes del evento**:

1. Ir a **https://signup.snowflake.com/**
2. Rellenar el formulario con email corporativo
3. Seleccionar: **AWS** | **Enterprise** | **US West (Oregon)**
4. Activar la cuenta desde el email de confirmacion
5. La cuenta incluye **$400 en creditos** y 30 dias de validez

(ALTER ACCOUNT DAB85420 SET COPILOT_ORCHESTRATOR_PARAM_153 = 'false' parameter_comment = 'Access for Enablement Session in Trial Account';)

---

## Objetivo

Cada equipo construira una mini plataforma end-to-end para su industria usando Snowflake Cortex Code como unico IDE:
- Conectar un repositorio Git de GitHub a Snowflake e ingestar datos JSON
- Transformar con patron medallion Bronze / Silver / Gold usando dbt
- Aplicar 3 casos de uso AI con Cortex sobre la capa Gold
- Crear una Semantic View para consultas en lenguaje natural (NL2SQL)
- Desplegar un Cortex Agent como Copilot en Snowflake Intelligence

---

## Agenda Detallada

| Hora | Duracion | Bloque | Descripcion |
|------|----------|--------|-------------|
| 09:30 - 10:00 | 30 min | **Recepcion y registro** | Acreditacion, entrega de materiales, cafe de bienvenida |
| 10:00 - 10:15 | 15 min | **Bienvenida** | Apertura institucional Snowflake. Presentacion del evento |
| 10:15 - 10:35 | 20 min | **Keynote: Cortex Code** | Demo en vivo de Cortex Code: flujo Git > Bronze > Silver > Gold > AI > SV > Agent |
| 10:35 - 10:45 | 10 min | **Presentacion del reto** | Explicacion de las industrias disponibles y las 5 fases del hackathon |
| 10:45 - 10:50 | 5 min | **Formacion de equipos** | Seleccion de industria, asignacion de mesas, entrega de guias |
| 10:50 - 12:20 | 90 min | **Hackathon** | Trabajo en equipos siguiendo la guia de su industria |
| | | *Fase 1 - Bronze (25 min)* | Conectar repo Git e ingestar JSON como capa Bronze |
| | | *Fase 2 - Silver/Gold (35 min)* | Transformacion con dbt: tablas normalizadas y KPIs |
| | | *Fase 3 - AI (20 min)* | 3 casos de uso AI con Cortex COMPLETE |
| | | *Fase 4 - Semantic View (5 min)* | Modelo semantico NL2SQL |
| | | *Fase 5 - Agent (5 min)* | Copilot en Snowflake Intelligence |
| 12:20 - 12:30 | 10 min | **Coffee break** | Descanso, preparacion de demos |
| 12:30 - 13:00 | 30 min | **Demos de equipos** | 5 min por equipo |
| 13:00 - 13:15 | 15 min | **Premios** | Votacion y entrega de premios |
| 13:15 - 13:30+ | 15 min+ | **Cierre y networking** | Comida tipo catering y networking |

---

## Arquitectura del Reto (comun a todas las industrias)

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
        |
   [ BRONZE ]  Raw JSON (4 ficheros por industria)
        |
   [ SILVER ]  Tablas normalizadas con dbt (flatten, cast, dedup)
        |
   [  GOLD  ]  KPIs y vistas analiticas (3 modelos por industria)
        |
   [   AI   ]  3 Casos de Uso con Cortex COMPLETE
        |
   [SEM.VIEW]  Modelo semantico NL2SQL (Cortex Analyst)
        |
   [ AGENT  ]  Copilot en Snowflake Intelligence
```

---

## Datasets por Industria

Cada industria incluye 4 ficheros JSON con la misma estructura de volumetria:

| Fichero | Registros | Rol |
|---------|-----------|-----|
| Entidades principales | 350 | Clientes, pacientes, equipos, vehiculos, etc. |
| Eventos operativos | 10,000 | Transacciones, lecturas, pedidos, encuentros, etc. |
| Tickets / Incidencias | 750 | Alertas, casos, ordenes de trabajo, notas clinicas, etc. |
| Datos financieros | 2,000 | Facturacion, cuentas, envios, presupuestos, etc. |

**Repositorio:** https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon
**Ruta:** `data/{industria}/`

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

## Revision de Costes Post-Hackathon

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

> Las vistas de ACCOUNT_USAGE tienen un retraso de hasta 2 horas. Revisad al dia siguiente.
