---
tipo: design
modulo: schema-monitor
version: 1.1
fecha: 2026-07-12
status: borrador
---

# Design — Monitoreo y corrección de schema MySQL

> Origen: [[30-Gestion/debates/debate-2026-07-12-schema-monitoring]] (Fase 1 read-only ya implementada: `get_schema_health`, `get_slow_queries` en `routes/mcp.js`, lógica compartida en `lib/schemaHealth.js`).
> Este doc formaliza la Fase 2: cómo se programa, persiste y visualiza ese chequeo — sin reabrir [[feedback_cowork_propio_shelved|la idea de "cowork propio"]] descartada el mismo día.

---

## Motivación

Fase 1 dejó los datos accesibles (MCP + REST `/db/diagnostics`) pero on-demand: hay que pedirlos activamente en una sesión Code/Desktop. Evidencia real encontrada al probarlo (2026-07-12): la query de `market_sentiment` (`MAX(CASE WHEN ms.fecha_hora = latest.max_f...`) corrió 51,521 veces, examinando ~1.38B filas acumuladas (~261h). Eso justifica tener el dato siempre disponible y poder ver su evolución tras aplicar una corrección.

## Por qué esto NO es "cowork propio"

La idea descartada era un scheduler general dependiente de `mcp__scheduled-tasks` / sesión Claude activa, con 5 riesgos identificados (confiabilidad, resiliencia, colisión de mecanismos, `PushNotification` no apto para desatendido, alto costo para bajo dolor). Esta propuesta usa un único cron nativo de Node dentro de un proceso que **ya corre 24/7** (`server-api` en PM2) — no agrega un mecanismo nuevo de "programar cosas", reutiliza el que ya existe.

---

## Arquitectura

```
node-cron (dentro de server-api, PM2)
    │  1x/día
    ▼
schemaHealth.js → getSchemaHealth() + getSlowQueries()
    │
    ▼
INSERT en tabla schema_health_history (nueva)
    │
    ▼
GET /schema-report  →  página HTML lee el último snapshot + histórico por digest
    │
    ▼
Botón "🔧 Proponer corrección"  →  sendPrompt-equivalente: abre caso en chat con Code
                                     (propuesta manual, no ejecución automática — run_schema_fix sigue pausado)
```

## Tabla `schema_health_history`

Permite ver evolución de una misma query/índice entre chequeos (antes/después de una corrección), no solo el último snapshot.

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | BIGINT AUTO_INCREMENT PK | |
| `fecha_ejecucion` | DATETIME | Cuándo corrió el chequeo |
| `tipo_hallazgo` | ENUM('full_scan','indice_sin_uso','tabla_grande','buffer_pool') | Categoría |
| `tabla_afectada` | VARCHAR(64) | Ej: `market_sentiment` |
| `digest_query` | VARCHAR(80) | `SUBSTRING(digest_text,1,80)` — identifica la misma query entre corridas |
| `reporte` | LONGBLOB | Reporte completo de esa corrida (JSON serializado o HTML) — payload libre, no atado a un esquema fijo de columnas |
| `estado` | ENUM('detectado','propuesto','resuelto','descartado') | Ciclo de vida del hallazgo |
| `propuesta_correccion` | TEXT NULL | Qué se propuso (índice, rewrite, etc.) |
| `fecha_resolucion` | DATETIME NULL | Cuándo se marcó resuelto |

Índice sugerido: `(digest_query, fecha_ejecucion)` para traer la serie histórica de una query puntual.

`reporte` en BLOB en vez de JSON tipado: cada fila guarda el reporte completo de esa corrida sin depender de columnas fijas — así se pueden acumular varios reportes por fila/caso (o incluso tipos de reporte distintos a futuro, no solo schema health) sin migrar el esquema cada vez que cambia lo que se mide. Las columnas livianas (`tipo_hallazgo`, `tabla_afectada`, `digest_query`, `estado`) siguen siendo las que se filtran/indexan; el BLOB es solo el detalle.

## Página `/schema-report`

- Última corrida: hallazgos activos (estado != resuelto), ordenados por severidad (executions × avg_time)
- Por hallazgo: mini-serie histórica (¿mejoró o empeoró desde la última corrección?)
- Botón "🔄 Actualizar" — dispara corrida manual fuera del horario del cron (llama al mismo endpoint que usa el cron)
- Botón "🔧 Proponer corrección" por hallazgo — abre el caso en chat, no ejecuta nada solo
- Auth: pendiente definir (login liviano vs. reusar OAuth existente — ver sección Pendientes)

---

## Pendientes de decisión

- [ ] Auth de la página para navegador (no puede mandar `X-API-Key` por header simple) — login liviano vs. OAuth existente
- [ ] Frecuencia del cron (propuesta inicial: 1x/día)
- [ ] Umbral de severidad para listar un hallazgo como "activo" en la página
- [ ] Quién marca `estado = resuelto` — manual (vos/Code) por ahora, sin automatización

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-12 | Versión inicial — diseño Fase 2 (cron + tabla histórica + página) |
| 1.1 | 2026-07-12 | `metricas JSON` → `reporte LONGBLOB` — permite acumular varios reportes por caso sin esquema fijo |
