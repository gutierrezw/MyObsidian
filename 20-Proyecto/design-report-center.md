---
tipo: design
modulo: report-center
version: 2.0
fecha: 2026-07-12
status: borrador
---

# Design — Motor genérico de reportes (Report Center)

> Extraído de [[design-schema-monitor]] al generalizar el patrón: tabla histórica + página con auth transparente + botón de corrección, pensado para servir a cualquier reporte futuro (schema health es el primer consumidor, no el único).

---

## Motivación

`schema-monitor` (Fase 2) definió tabla histórica + `/schema-report` + Cloudflare Access + botón "proponer corrección". Nada de eso es específico de MySQL — el mismo patrón sirve para cualquier chequeo periódico que necesite: guardar resultado, ver evolución, y pedir revisión sin ejecutar nada solo. Generalizarlo ahora evita reconstruirlo cuando aparezca el próximo caso (ej. resumen diario de IB reconcile, salud del pipeline 13F).

## Arquitectura

**Principio: un solo dueño de la lógica por dominio, vive siempre dentro de `server-api`.** Lo único que puede ser externo es el *disparador* (qué dispara cuándo corre) — nunca el *cómputo*. Evita el problema real que motivó esta reconciliación: `mysql-weekly-report` tenía su propia copia de la lógica de schema health corriendo por fuera, duplicando `lib/schemaHealth.js`.

```
Disparador — interno (node-cron dentro de server-api) o externo (Windows Task Scheduler, trigger delgado sin lógica)
    ▼
POST /internal/reports/:tipo/run   (si el disparador es externo; interno llama directo)
    ▼
Módulo dueño del dominio (schemaHealth.js, ibReconcile.js, ...)  ← ÚNICA implementación, la reusan también los MCP tools/REST si aplica
    ▼
ReportManager.registrar(tipoReporte, categoria, referencia, reporteBuffer)
    │
    ▼
INSERT en tabla reportes_historial (genérica, un solo lugar)
    │
    ▼
GET /reports/:tipo  →  página genérica lee ReportManager.ultimo()/historico()
    │
    ▼
Botón "🔧 Proponer corrección"  →  abre caso en chat (genérico, no ejecuta nada solo)
```

Cloudflare Access protege `/reports/*` completo — un solo setup de auth sirve para todos los reportes presentes y futuros, no uno por tipo.

[[design-schema-monitor]] (primer consumidor) usa el camino externo: Windows Task Scheduler ya corre lunes 8am (antes disparaba `mysql-weekly-report/report.js` con su propia lógica) — se repunta para que solo dispare `POST /internal/reports/schema_health/run`, sin cómputo propio. `mysql-weekly-report` se retira.

## Tabla `reportes_historial`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | BIGINT AUTO_INCREMENT PK | |
| `tipo_reporte` | VARCHAR(32) | Ej: `schema_health`, `ib_reconcile` (futuro) |
| `fecha_ejecucion` | DATETIME | Cuándo corrió |
| `categoria` | VARCHAR(32) | Subtipo dentro del reporte (ej: `full_scan`, `indice_sin_uso`) |
| `referencia` | VARCHAR(64) | Identifica el "caso" entre corridas para armar serie histórica (ej: `digest_query`, un símbolo, una tabla) |
| `reporte` | LONGBLOB | Payload completo de esa corrida — libre, no atado a columnas fijas |
| `estado` | ENUM('detectado','propuesto','resuelto','descartado') | Ciclo de vida del hallazgo |
| `propuesta_correccion` | TEXT NULL | Qué se propuso |
| `fecha_resolucion` | DATETIME NULL | Cuándo se marcó resuelto |

Índice: `(tipo_reporte, referencia, fecha_ejecucion)`.

## Clase `ReportManager` (`server-api/lib/ReportManager.js`)

Dueña del dominio "reportes" — cada módulo que genera un reporte la llama, no reimplementa el INSERT/SELECT.

| Método | Uso |
|--------|-----|
| `registrar(tipoReporte, categoria, referencia, reporteBuffer)` | Inserta una corrida nueva |
| `ultimo(tipoReporte)` | Hallazgos activos de la corrida más reciente |
| `historico(tipoReporte, referencia)` | Serie histórica de un caso puntual |
| `marcarResuelto(id, propuestaCorreccion)` | Cierra un hallazgo |

`routes/reports.js` expone `GET /reports/:tipo` y `POST /internal/reports/:tipo/run` (trigger genérico, requiere `X-API-Key`) — el módulo dueño del dominio (ej. `schemaHealth.js`) hace su chequeo y llama `ReportManager.registrar(...)` al final; coordinador simple, la lógica de qué medir vive únicamente en ese módulo, nunca fuera de `server-api`.

## Cómo se suma un reporte nuevo

1. El módulo dueño del dominio se implementa **dentro de `server-api`** (igual que `schemaHealth.js`) — aunque el disparo venga de afuera (Task Scheduler u otro), el cómputo no sale de acá
2. Se registra su `tipo_reporte` en `routes/reports.js` para que `POST /internal/reports/:tipo/run` sepa qué módulo invocar
3. Al terminar, llama `ReportManager.registrar('nuevo_tipo', categoria, referencia, buffer)`
4. Ya aparece en `/reports/nuevo_tipo` — sin tocar auth, tabla ni página

## Estilo visual

La página `/reports/:tipo` es genérica — misma UI para cualquier `tipo_reporte` — así que el estilo se define **una sola vez** y no por reporte.

- **Fase de ajuste:** durante la implementación de `schema_health` (primer consumidor), el estilo queda abierto a tuneo — colores, layout, cómo se ven las mini-series históricas, botones.
- **Congelamiento:** al cerrar esa Fase 1, el estilo aprobado se documenta acá (CSS/paleta/tipografía) como estándar. Los reportes que se sumen después (`ib_reconcile`, etc.) lo heredan automáticamente vía la página genérica — no se rediseña por tipo.
- **Cómo se prueba:** conviene iterar el look en un Artifact (prototipo rápido, sin tocar `server-api`) antes de portarlo a la página real — así se ajusta visualmente sin fricción de deploy.

*(Pendiente: paleta/tipografía definitiva — se completa esta sección al cerrar Fase 1.)*

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-12 | Extraído de design-schema-monitor al generalizar el patrón |
| 1.1 | 2026-07-12 | Agregada sección Estilo visual — se tunea en Fase 1 (schema_health) y se congela como estándar para reportes futuros |
| 1.2 | 2026-07-12 | Aclarado que el disparador puede ser un proceso externo ya agendado (vía `POST /internal/report`), no solo un cron interno de `server-api` — motivado por descubrir que `mysql-weekly-report` ya cubría el primer consumidor |
| 2.0 | 2026-07-12 | **Principio explícito: un solo dueño de la lógica, siempre dentro de `server-api`** — solo el disparador puede ser externo, nunca el cómputo. Usuario eligió esta opción sobre mantener `mysql-weekly-report` independiente. Endpoint de trigger nombrado: `POST /internal/reports/:tipo/run`. `mysql-weekly-report` se retira (ver [[design-schema-monitor]] para el plan de migración) |
