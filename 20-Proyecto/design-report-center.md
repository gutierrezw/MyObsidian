---
tipo: design
modulo: report-center
version: 1.1
fecha: 2026-07-12
status: borrador
---

# Design — Motor genérico de reportes (Report Center)

> Extraído de [[design-schema-monitor]] al generalizar el patrón: tabla histórica + página con auth transparente + botón de corrección, pensado para servir a cualquier reporte futuro (schema health es el primer consumidor, no el único).

---

## Motivación

`schema-monitor` (Fase 2) definió tabla histórica + `/schema-report` + Cloudflare Access + botón "proponer corrección". Nada de eso es específico de MySQL — el mismo patrón sirve para cualquier chequeo periódico que necesite: guardar resultado, ver evolución, y pedir revisión sin ejecutar nada solo. Generalizarlo ahora evita reconstruirlo cuando aparezca el próximo caso (ej. resumen diario de IB reconcile, salud del pipeline 13F).

## Arquitectura

```
Cualquier dominio (schemaHealth.js, ibReconcile.js, ...)
    │  corre su propia lógica (cron o on-demand)
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

`routes/reports.js` expone `GET /reports/:tipo` — página genérica, misma UI para cualquier `tipo_reporte`. Los crons de cada dominio (ej. `schemaHealth.js`) solo llaman `ReportManager.registrar(...)` al final de su corrida — coordinador simple, la lógica de qué medir sigue viviendo en el módulo dueño de ese dominio.

## Cómo se suma un reporte nuevo

1. El módulo dueño del dominio corre su chequeo (igual que `schemaHealth.js`)
2. Llama `ReportManager.registrar('nuevo_tipo', categoria, referencia, buffer)`
3. Ya aparece en `/reports/nuevo_tipo` — sin tocar auth, tabla ni página

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
