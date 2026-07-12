---
tipo: design
modulo: schema-monitor
version: 3.0
fecha: 2026-07-12
status: implementado (backend + Task Scheduler, falta UI)
---

# Design — Monitoreo y corrección de schema MySQL

> Origen: [[30-Gestion/debates/debate-2026-07-12-schema-monitoring]] (Fase 1 read-only ya implementada: `get_schema_health`, `get_slow_queries` en `routes/mcp.js`, lógica compartida en `lib/schemaHealth.js`).
> Fase 2 (este doc) es el **primer consumidor** de [[design-report-center|Report Center]] — el motor genérico de tabla histórica + página + auth. Acá solo va lo específico de schema health; lo genérico (tabla `reportes_historial`, `ReportManager`, Cloudflare Access) vive en ese doc.

---

## Motivación

Fase 1 dejó los datos accesibles (MCP + REST `/db/diagnostics`) pero on-demand: hay que pedirlos activamente en una sesión Code/Desktop. Evidencia real encontrada al probarlo (2026-07-12): la query de `market_sentiment` (`MAX(CASE WHEN ms.fecha_hora = latest.max_f...`) corrió 51,521 veces, examinando ~1.38B filas acumuladas (~261h). Eso justifica tener el dato siempre disponible y poder ver su evolución tras aplicar una corrección.

## Por qué esto NO es "cowork propio"

La idea descartada era un scheduler general dependiente de `mcp__scheduled-tasks` / sesión Claude activa, con 5 riesgos identificados (confiabilidad, resiliencia, colisión de mecanismos, `PushNotification` no apto para desatendido, alto costo para bajo dolor). Esta propuesta usa un único cron nativo de Node dentro de un proceso que **ya corre 24/7** (`server-api` en PM2) — no agrega un mecanismo nuevo de "programar cosas", reutiliza el que ya existe.

---

## Arquitectura (específica de schema health)

> Decisión final 2026-07-12: **una sola versión, un solo dueño de la lógica.** `server-api/lib/schemaHealth.js` es la única implementación de análisis (full scans, índices sin uso, buffer pool, tamaño de tablas) — la usan tanto los MCP tools (vivo) como la corrida semanal persistida (histórico). `MyNode/mysql-weekly-report/` (script Node standalone con sus propias queries + HTML local) **se retira** — quedaría duplicando exactamente esta lógica en otro lugar.

```
Windows Task Scheduler "Reporte Semanal MySQL" (lunes 8am, YA existe)
    │  se repuntan sus Arguments: de `node report.js` a un trigger delgado (curl/HTTP), sin lógica propia
    ▼
POST /internal/reports/schema_health/run   (server-api, requiere X-API-Key)
    ▼
schemaHealth.js → getSchemaHealth() + getSlowQueries()   ← ÚNICA implementación, misma que usan los MCP tools
    ▼
ReportManager.registrar('schema_health', categoria, referencia, reporteBuffer)   ← ver [[design-report-center]]
    ▼
GET /reports/schema_health  →  página genérica de Report Center (Cloudflare Access, celular)
```

**Dos consumidores, un solo cómputo — no hay duplicación:**
- **Vivo/on-demand:** MCP `get_schema_health` / `get_slow_queries` (Fase 1, ya implementado en `routes/mcp.js`) — llama `schemaHealth.js` directo, sin pasar por la tabla.
- **Histórico/snapshot:** este doc (Fase 2) — el trigger semanal llama el mismo `schemaHealth.js` y persiste el resultado en `reportes_historial`, permite ver evolución y marcar resuelto.

**Migración de `mysql-weekly-report`:**
1. Repuntar la acción de la tarea "Reporte Semanal MySQL" (`schtasks /change`) de `node .../report.js` a un trigger HTTP contra `/internal/reports/schema_health/run`
2. Portar a `schemaHealth.js` cualquier chequeo que `report.js`/`analyze-fullscan.js` tuviera y `schemaHealth.js` todavía no cubra (ej. sugerencias de índice de `analyze-fullscan.js` → alimentan `propuesta_correccion`)
3. Una vez validada 2-3 semanas la corrida vía server-api, archivar `MyNode/mysql-weekly-report/` (no borrar de una — mantiene su HTML histórico ya generado como referencia)

**Trade-off aceptado:** la corrida semanal ahora depende de que `server-api`/PM2 esté arriba (antes `mysql-weekly-report` conectaba directo a MySQL, independiente de server-api). Se acepta porque server-api ya es la base 24/7 del resto del co-work (MCP, `/db/*`) — si eso cae, ya hay impacto mayor de todos modos.

## Mapeo a los campos genéricos de `reportes_historial`

| Campo genérico | Valor en schema-monitor |
|-----------------|--------------------------|
| `tipo_reporte` | `schema_health` |
| `categoria` | `full_scan`, `indice_sin_uso`, `tabla_grande`, `buffer_pool` |
| `referencia` | `digest_query` (`SUBSTRING(digest_text,1,80)`) o nombre de tabla, según categoría |
| `reporte` | Salida completa de `getSchemaHealth()`/`getSlowQueries()` para esa corrida |

## Parámetros de esta instancia

- **Frecuencia:** 1x/semana, lunes 8am — la que ya tiene "Reporte Semanal MySQL" en Task Scheduler, sin cambios
- **Umbral de severidad:** un hallazgo aparece como "activo" si `rows_examined` total de esa corrida supera 10M filas (el caso de `market_sentiment` está en ~1.38B — bien por encima; se ajusta con datos reales más adelante). **⚠️ No implementado aún** — `routes/reports.js` hoy registra los 10 `full_scan` que devuelve `schemaHealth.js` sin filtrar por umbral; pendiente decidir si el filtro va en `schemaHealth.js` o al registrar en `ReportManager`
- **Quién marca `estado = resuelto`:** manual (vos o Code en sesión), sin automatización
- **Botón "🔧 Proponer corrección":** abre el caso en chat — propuesta manual, no ejecuta nada solo (`run_schema_fix` sigue pausado)

---

## Pendientes de decisión

- [x] Auth — resuelto a nivel Report Center (Cloudflare Access, ver [[design-report-center]])
- [x] Frecuencia — 1x/semana, lunes 8am (la ya existente, sin cron nuevo)
- [x] Umbral de severidad — `rows_examined` > 10M por corrida
- [x] Quién marca `resuelto` — manual (vos/Code)

Sin pendientes abiertos. Implementación = (a) repuntar Task Scheduler a un trigger HTTP delgado, (b) portar lo que falte de `mysql-weekly-report` a `schemaHealth.js`, (c) `/reports/schema_health` vía Report Center. Sin infraestructura de scheduling nueva — se reutiliza tanto el Task Scheduler existente como el proceso 24/7 de `server-api`.

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-12 | Versión inicial — diseño Fase 2 (cron + tabla histórica + página) |
| 1.1 | 2026-07-12 | `metricas JSON` → `reporte LONGBLOB` — permite acumular varios reportes por caso sin esquema fijo |
| 1.2 | 2026-07-12 | Auth decidido: login liviano con `api_key` en `localStorage`, se descarta OAuth |
| 1.3 | 2026-07-12 | Auth revertido a **Cloudflare Access** — pegar API key no era viable desde celular |
| 2.0 | 2026-07-12 | Generalizado: tabla/página/auth extraídos a [[design-report-center]] (motor reutilizable). Este doc queda como su primer consumidor. Cerrados los 3 pendientes restantes (frecuencia, umbral, resolución manual) |
| 2.1 | 2026-07-12 | Frecuencia del cron: 1x/día → 1x/semana |
| 2.2 | 2026-07-12 | Unificado con `MyNode/mysql-weekly-report/report.js` (ya corría lunes 8am vía Task Scheduler, no detectado hasta ahora) |
| 3.0 | 2026-07-12 | **Una sola versión, no dos.** Decidido que `server-api/lib/schemaHealth.js` es el único dueño de la lógica de análisis (usuario eligió esta opción sobre mantener `mysql-weekly-report` independiente). `mysql-weekly-report` se retira — Task Scheduler pasa a ser un trigger HTTP delgado sin lógica propia. Trade-off aceptado: la corrida semanal ahora depende de que `server-api` esté arriba |
| 3.1 | 2026-07-12 | **Backend probado end-to-end contra `bdinv` real**: `POST /internal/reports/schema_health/run` persistió 40 hallazgos (10 full_scan / 24 indice_sin_uso / 5 tabla_grande / 1 buffer_pool), incluyendo `market_sentiment` (1.46B filas examinadas, 54,649 veces). `GET /reports/schema_health` los leyó correctamente. Detectado al revisar: el umbral de severidad (10M filas) documentado arriba **no está implementado** todavía. Pendiente real: repuntar Task Scheduler y construir la página `/reports/:tipo` |
| 3.2 | 2026-07-12 | **Task Scheduler migrada.** "Reporte Semanal MySQL" repuntada de `node report.js` a `scripts/trigger-schema-health.ps1` (curl delgado, sin lógica propia). Verificado con ejecución manual: `LastTaskResult=0`, 120 filas acumuladas en `reportes_historial`. `mysql-weekly-report` queda desconectado del scheduler (carpeta aún no archivada). Nota operativa: `schtasks /change /tr` con comillas anidadas corrompe el argumento sin avisar — usar `Set-ScheduledTask`/`New-ScheduledTaskAction` |
