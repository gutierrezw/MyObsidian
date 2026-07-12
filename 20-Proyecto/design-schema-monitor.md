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

```
node-cron (dentro de server-api, PM2)  — 1x/semana
    ▼
schemaHealth.js → getSchemaHealth() + getSlowQueries()
    ▼
ReportManager.registrar('schema_health', categoria, referencia, reporteBuffer)   ← ver [[design-report-center]]
    ▼
GET /reports/schema_health  →  página genérica de Report Center
```

## Mapeo a los campos genéricos de `reportes_historial`

| Campo genérico | Valor en schema-monitor |
|-----------------|--------------------------|
| `tipo_reporte` | `schema_health` |
| `categoria` | `full_scan`, `indice_sin_uso`, `tabla_grande`, `buffer_pool` |
| `referencia` | `digest_query` (`SUBSTRING(digest_text,1,80)`) o nombre de tabla, según categoría |
| `reporte` | Salida completa de `getSchemaHealth()`/`getSlowQueries()` para esa corrida |

## Parámetros de esta instancia

- **Frecuencia del cron:** 1x/semana, de madrugada (ej. lunes 3am) — no compite con el pipeline 13F ni el uso normal de la app
- **Umbral de severidad:** un hallazgo aparece como "activo" si `rows_examined` total de esa corrida supera 10M filas (el caso de `market_sentiment` está en ~1.38B — bien por encima; se ajusta con datos reales más adelante)
- **Quién marca `estado = resuelto`:** manual (vos o Code en sesión), sin automatización
- **Botón "🔧 Proponer corrección":** abre el caso en chat — propuesta manual, no ejecuta nada solo (`run_schema_fix` sigue pausado)

---

## Pendientes de decisión

- [x] Auth — resuelto a nivel Report Center (Cloudflare Access, ver [[design-report-center]])
- [x] Frecuencia del cron — 1x/semana, madrugada
- [x] Umbral de severidad — `rows_examined` > 10M por corrida
- [x] Quién marca `resuelto` — manual (vos/Code)

Sin pendientes abiertos — diseño listo para pasar a implementación.

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-12 | Versión inicial — diseño Fase 2 (cron + tabla histórica + página) |
| 1.1 | 2026-07-12 | `metricas JSON` → `reporte LONGBLOB` — permite acumular varios reportes por caso sin esquema fijo |
| 1.2 | 2026-07-12 | Auth decidido: login liviano con `api_key` en `localStorage`, se descarta OAuth |
| 1.3 | 2026-07-12 | Auth revertido a **Cloudflare Access** — pegar API key no era viable desde celular |
| 2.0 | 2026-07-12 | Generalizado: tabla/página/auth extraídos a [[design-report-center]] (motor reutilizable). Este doc queda como su primer consumidor. Cerrados los 3 pendientes restantes (frecuencia, umbral, resolución manual) |
| 2.1 | 2026-07-12 | Frecuencia del cron: 1x/día → 1x/semana |
