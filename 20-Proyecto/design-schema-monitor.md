---
tipo: design
modulo: schema-monitor
version: 2.1
fecha: 2026-07-12
status: borrador
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

> Corrección 2026-07-12: no hace falta un cron nuevo. `MyNode/mysql-weekly-report/report.js` ya corre lunes 8am vía Windows Task Scheduler ("Reporte Semanal MySQL") — es el mismo reporte que ya menciona el debate de origen (ver nota abajo). Se extiende ese script en vez de construir un cron dentro de `server-api`.

```
mysql-weekly-report/report.js (Task Scheduler, YA corre lunes 8am)
    │  arma su HTML local como siempre
    ▼
POST /internal/report  (mismo patrón que AppOO → /internal/update)
    ▼
ReportManager.registrar('schema_health', categoria, referencia, reporteBuffer)   ← ver [[design-report-center]]
    ▼
GET /reports/schema_health  →  página genérica de Report Center
```

**Dos caminos distintos a la misma info — no son redundantes:**
- **Vivo/on-demand:** MCP `get_schema_health` / `get_slow_queries` (Fase 1, ya implementado en `routes/mcp.js`) — corre la query en el momento, sin histórico.
- **Histórico/snapshot:** este doc (Fase 2) — persiste la corrida semanal de `mysql-weekly-report` en `reportes_historial`, permite ver evolución y marcar resuelto.

## Mapeo a los campos genéricos de `reportes_historial`

| Campo genérico | Valor en schema-monitor |
|-----------------|--------------------------|
| `tipo_reporte` | `schema_health` |
| `categoria` | `full_scan`, `indice_sin_uso`, `tabla_grande`, `buffer_pool` |
| `referencia` | `digest_query` (`SUBSTRING(digest_text,1,80)`) o nombre de tabla, según categoría |
| `reporte` | Salida completa de `getSchemaHealth()`/`getSlowQueries()` para esa corrida |

## Parámetros de esta instancia

- **Frecuencia:** 1x/semana, lunes 8am — la que ya tiene "Reporte Semanal MySQL" en Task Scheduler, sin cambios
- **Umbral de severidad:** un hallazgo aparece como "activo" si `rows_examined` total de esa corrida supera 10M filas (el caso de `market_sentiment` está en ~1.38B — bien por encima; se ajusta con datos reales más adelante)
- **Quién marca `estado = resuelto`:** manual (vos o Code en sesión), sin automatización
- **Botón "🔧 Proponer corrección":** abre el caso en chat — propuesta manual, no ejecuta nada solo (`run_schema_fix` sigue pausado)

---

## Pendientes de decisión

- [x] Auth — resuelto a nivel Report Center (Cloudflare Access, ver [[design-report-center]])
- [x] Frecuencia — 1x/semana, lunes 8am (la ya existente, sin cron nuevo)
- [x] Umbral de severidad — `rows_examined` > 10M por corrida
- [x] Quién marca `resuelto` — manual (vos/Code)

Sin pendientes abiertos. Implementación = extender `mysql-weekly-report/report.js` con el POST a `/internal/report` — no requiere levantar infraestructura de scheduling nueva.

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-12 | Versión inicial — diseño Fase 2 (cron + tabla histórica + página) |
| 1.1 | 2026-07-12 | `metricas JSON` → `reporte LONGBLOB` — permite acumular varios reportes por caso sin esquema fijo |
| 1.2 | 2026-07-12 | Auth decidido: login liviano con `api_key` en `localStorage`, se descarta OAuth |
| 1.3 | 2026-07-12 | Auth revertido a **Cloudflare Access** — pegar API key no era viable desde celular |
| 2.0 | 2026-07-12 | Generalizado: tabla/página/auth extraídos a [[design-report-center]] (motor reutilizable). Este doc queda como su primer consumidor. Cerrados los 3 pendientes restantes (frecuencia, umbral, resolución manual) |
| 2.1 | 2026-07-12 | Frecuencia del cron: 1x/día → 1x/semana |
| 2.2 | 2026-07-12 | Unificado con `MyNode/mysql-weekly-report/report.js` (ya corría lunes 8am vía Task Scheduler, no detectado hasta ahora). Se descarta el node-cron propio: el script existente hace POST a `/internal/report` en vez de duplicar el disparador. Aclarada la relación con los MCP tools (vivo/on-demand) vs este doc (histórico/snapshot) |
