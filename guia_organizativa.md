# Guia Organizativa - Hackathon Telco Operations Copilot
## Snowflake Cortex Code Experience | Madrid, 28 mayo 2026

---

## 1. Datos Generales

| Campo | Detalle |
|-------|---------|
| Fecha | 28 de mayo de 2026 |
| Horario | 10:00 - 13:30 (recepcion desde 09:30) |
| Lugar | Oficinas de Snowflake, Madrid |
| Asistentes | 30 clientes + equipo Snowflake |
| Formato | 6 equipos de 5 personas |
| Catering | Comida servida a partir de las 13:00 |

---

## 2. Checklist Pre-Evento (T-4 semanas)

### Logistica y Espacio
- [ ] Reservar sala principal (capacidad 40+ personas, 6 mesas de equipo)
- [ ] Confirmar disponibilidad de pantalla/proyector principal para keynote
- [ ] Confirmar WiFi de alta capacidad (30+ dispositivos simultaneos)
- [ ] Reservar 6 monitores/pantallas auxiliares (uno por mesa de equipo)
- [ ] Preparar senalizacion: entrada, mesas de equipo, zona catering
- [ ] Verificar enchufes y regletas suficientes (minimo 2 por mesa)

### Catering
- [ ] Contratar servicio de catering para 35 personas (margen de seguridad)
- [ ] Cafe de bienvenida (09:30): cafe, te, zumos, bolleria
- [ ] Coffee break (12:20): cafe, refrescos, snacks
- [ ] Comida (13:00): menu tipo finger food / standing lunch
- [ ] Confirmar alergias/restricciones dieteticas de asistentes
- [ ] Pedir agua embotellada en cada mesa de equipo

### Comunicaciones
- [ ] Enviar save-the-date (T-4 semanas)
- [ ] Enviar invitacion formal con agenda y requisitos (T-2 semanas)
- [ ] Recordatorio con instrucciones de acceso al edificio (T-3 dias)
- [ ] Confirmar asistencia final (T-1 semana)
- [ ] Preparar email post-evento con fotos y recursos

---

## 3. Checklist Pre-Evento (T-1 semana)

### Entornos Snowflake
- [ ] Crear 6 cuentas/entornos Snowflake (uno por equipo)
- [ ] Cargar dataset telco JSON en stage de cada entorno (customers, network_events, tickets, billing_usage)
- [ ] Verificar que Cortex Code esta habilitado en todos los entornos
- [ ] Verificar acceso a funciones Cortex AI (COMPLETE, SUMMARIZE, CLASSIFY)
- [ ] Verificar acceso a CREATE SEMANTIC VIEW y CREATE AGENT
- [ ] Crear base de datos SNOWFLAKE_INTELLIGENCE con esquema AGENTS en cada entorno
- [ ] Probar el flujo completo end-to-end en un entorno de prueba (hasta Agent)
- [ ] Preparar credenciales de acceso en tarjetas individuales por equipo

### Materiales por Equipo
- [ ] Guia de equipo impresa (flujo Bronze > Silver > Gold > AI > Semantic View > Agent + prompts CoCo)
- [ ] Tarjeta de credenciales Snowflake (usuario, password, URL)
- [ ] Quick reference card: funciones Cortex AI + sintaxis CREATE SEMANTIC VIEW + CREATE AGENT
- [ ] Hoja de evaluacion para jueces (por equipo)

### Materiales Generales
- [ ] Presentacion PPTX cargada y probada en el portatil de presentacion
- [ ] Cronometro visible para las demos (5 min por equipo)
- [ ] Tarjetas de votacion o formulario online para jueces
- [ ] Premios para el equipo ganador

---

## 4. Equipo Snowflake - Roles y Responsabilidades

| Rol | Persona | Responsabilidades |
|-----|---------|-------------------|
| **MC / Presentador** | TBD | Apertura, keynote, transiciones, premios |
| **Demo Lead** | TBD | Demo en vivo de Cortex Code durante la keynote |
| **Mentor Mesa 1-2** | TBD | Soporte tecnico para equipos 1 y 2 |
| **Mentor Mesa 3-4** | TBD | Soporte tecnico para equipos 3 y 4 |
| **Mentor Mesa 5-6** | TBD | Soporte tecnico para equipos 5 y 6 |
| **Juez 1** | TBD | Evaluacion de demos |
| **Juez 2** | TBD | Evaluacion de demos |
| **Juez 3** | TBD | Evaluacion de demos |
| **Coordinador Logistica** | TBD | Catering, espacio, materiales, timing |
| **Fotografo/Social** | TBD | Fotos, video highlights, publicacion en redes |

---

## 5. Timeline del Dia - Acciones Equipo Snowflake

| Hora | Accion |
|------|--------|
| 08:30 | Llegada equipo organizador. Montar mesas, probar AV, distribuir materiales |
| 09:00 | Verificar catering de bienvenida. Test final de entornos Snowflake |
| 09:30 | Abrir puertas. Recepcion y registro de asistentes |
| 10:00 | MC: bienvenida. Demo Lead: preparar demo |
| 10:15 | Demo Lead: keynote Cortex Code en vivo |
| 10:35 | MC: presentar el reto telco y la mecanica |
| 10:45 | Coordinador: distribuir guias y credenciales. Mentores: a sus mesas |
| 10:50 | Mentores: soporte activo. MC: anunciar checkpoints cada 30 min |
| 11:20 | Checkpoint 1: "Deberian tener Bronze listo" |
| 11:50 | Checkpoint 2: "Silver/Gold deberian estar en progreso" |
| 12:00 | Checkpoint 3: "AI y Semantic View deberian estar avanzados" |
| 12:10 | MC: "10 minutos para cerrar y preparar demos" |
| 12:20 | Catering: servir coffee break. Equipos preparan presentacion |
| 12:30 | Jueces: listos con hojas de evaluacion. MC: gestionar turnos de 5 min |
| 13:00 | Jueces: deliberar. MC: entretener con Q&A mientras se decide |
| 13:10 | MC: anunciar ganadores. Entrega de premios |
| 13:15 | Catering: servir comida. Networking libre |

---

## 6. Kit de Rescate Tecnico

### Problemas Frecuentes y Soluciones

| Problema | Solucion |
|----------|----------|
| "No puedo acceder a Snowflake" | Verificar credenciales. Mentor usa cuenta admin para resetear |
| "Cortex Code no responde" | Refrescar navegador. Verificar warehouse activo |
| "El JSON no carga" | Verificar formato del stage. Mentor tiene script de carga de backup |
| "dbt da error" | Revisar profile.yml. Mentor tiene proyecto dbt de referencia |
| "Cortex AI devuelve error" | Verificar que el warehouse tiene creditos. Comprobar sintaxis del prompt |
| "CREATE SEMANTIC VIEW falla" | Revisar nombres de tabla/columna. Verificar que las tablas Gold existen |
| "CREATE AGENT falla" | Verificar permisos en SNOWFLAKE_INTELLIGENCE.AGENTS. Pedir ACCOUNTADMIN |
| "Agent no aparece en Intelligence" | Verificar que esta en snowflake_intelligence.agents y tiene GRANT USAGE |
| "No nos da tiempo" | Mentor ayuda a priorizar: Bronze > 1 caso AI > Semantic View > demo basica |

### Entorno de Backup
- [ ] Tener un entorno Snowflake pre-construido con Bronze/Silver/Gold ya listo
- [ ] Script SQL para cargar el dataset rapido si falla la ingesta
- [ ] Proyecto dbt de referencia con modelos Silver y Gold funcionales
- [ ] 3 queries SQL con los prompts Cortex AI ya escritos
- [ ] SQL de CREATE SEMANTIC VIEW y CREATE AGENT pre-escrito por si falla CoCo

---

## 7. Criterios de Evaluacion para Jueces

| Criterio | Peso | Que Evaluar |
|----------|------|-------------|
| Pipeline End-to-End | 30% | Bronze cargado, Silver normalizado, Gold con KPIs |
| Uso de Cortex Code | 20% | Han usado CoCo como IDE principal, no scripts manuales |
| Calidad 3 Casos AI | 20% | Los 3 prompts funcionan y dan resultados coherentes |
| Semantic View + Agent | 15% | Vista semantica funcional y agente respondiendo en Intelligence |
| Presentacion | 15% | Claridad, storytelling, valor de negocio explicado |

**Escala:** 1-5 por criterio. Puntuacion final = suma ponderada.

---

## 8. Post-Evento

- [ ] Enviar email de agradecimiento con fotos (T+1 dia)
- [ ] Compartir repositorio/recursos del hackathon con asistentes
- [ ] Recoger feedback via encuesta rapida (Google Forms / Typeform)
- [ ] Publicar highlights en LinkedIn / redes internas
- [ ] Agendar follow-ups con los equipos mas interesados en Snowflake
- [ ] Informe interno: asistencia, NPS, leads generados
