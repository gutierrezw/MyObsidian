---
tipo: design
modulo: report-center
version: 2.7
fecha: 2026-07-13
status: implementado (Fase 1)
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

**Acceso — Cloudflare Access + Google (activo desde 2026-07-05):** protege `/reports/*` completo en `api-main.wildaga.com` — un solo setup de auth sirve para todos los reportes presentes y futuros, no uno por tipo. Login vía Google (cuenta `gutierrez.madrid.wilmer@gmail.com`), policy creada y guardada en el dashboard de Cloudflare (Zero Trust → Access → Applications). No requiere manejo de contraseñas propio ni sesión separada del server-api — Cloudflare intercepta antes de llegar a Express. Setup fue un paso único, no necesita repetirse al sumar reportes nuevos (ver [[report-center-status]] en memoria para más contexto operativo).

[[design-schema-monitor]] (primer consumidor) usa el camino externo: Windows Task Scheduler ya corre lunes 8am (antes disparaba `mysql-weekly-report/report.js` con su propia lógica) — se repunta para que solo dispare `POST /internal/reports/schema_health/run`, sin cómputo propio. `mysql-weekly-report` se retira.

## Tabla `reportes_historial`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | BIGINT AUTO_INCREMENT PK | |
| `tipo_reporte` | VARCHAR(32) | Ej: `schema_health`, `ib_reconcile` (futuro) |
| `fecha_ejecucion` | DATETIME | Cuándo corrió |
| `categoria` | VARCHAR(32) | Subtipo dentro del reporte (ej: `full_scan`, `indice_sin_uso`) |
| `referencia` | VARCHAR(64) | Identifica el "caso" entre corridas para armar serie histórica (ej: `digest_query`, un símbolo, una tabla) |
| `tablas_afectadas` | VARCHAR(255) NULL | Nombre(s) de tabla MySQL involucrados en el hallazgo, separados por coma si hay más de uno (JOINs). Permite estadísticas "¿qué tabla concentra más hallazgos?" sin parsear `reporte`. `NULL` cuando no se puede determinar con confianza |
| `reporte` | LONGBLOB | Payload completo de esa corrida — libre, no atado a columnas fijas |
| `estado` | ENUM('detectado','propuesto','resuelto','descartado') | Ciclo de vida del hallazgo |
| `propuesta_correccion` | TEXT NULL | Qué se propuso |
| `fecha_resolucion` | DATETIME NULL | Cuándo se marcó resuelto |

Índice: `(tipo_reporte, referencia, fecha_ejecucion)`.

## Clase `ReportManager` (`server-api/lib/ReportManager.js`)

Dueña del dominio "reportes" — cada módulo que genera un reporte la llama, no reimplementa el INSERT/SELECT.

| Método | Uso |
|--------|-----|
| `registrar(tipoReporte, categoria, referencia, reporteBuffer, tablasAfectadas)` | Si ya existe un caso activo (mismo `tipo_reporte`+`referencia`, `estado` no resuelto/descartado), lo actualiza en el lugar (refresca `fecha_ejecucion`, `categoria`, `reporte`, `tablas_afectadas`). Si no existe, inserta fila nueva. Evita filas duplicadas para el mismo hallazgo en cada corrida — la tabla crece solo cuando aparece un caso nuevo o cuando uno ya resuelto vuelve a ocurrir |
| `ultimo(tipoReporte)` | Hallazgos activos de la corrida más reciente. Además calcula `no_reproducido` por fila (ver sección abajo) |
| `historico(tipoReporte, referencia)` | Serie histórica de un caso puntual |
| `marcarResuelto(id, propuestaCorreccion)` | Cierra un hallazgo — **afirma** que se confirmó/aplicó un fix |
| `marcarPropuesto(id)` | Pasa el hallazgo a `estado='propuesto'` — bandeja de pendientes manual, no ejecuta nada |
| `marcarDescartado(id, nota)` | Cierra un hallazgo `no_reproducido` sin afirmar un fix — para falsos positivos o casos no reproducidos. Distinto de `marcarResuelto`: `estado='descartado'`, no `'resuelto'` |

## Bandeja de pendientes — botón "🔧 Proponer corrección"

Implementado 2026-07-13, exactamente como preveía el diagrama de Arquitectura (sección arriba): el botón marca el hallazgo como `propuesto` (`POST /:tipo/:id/proponer` → `ReportManager.marcarPropuesto`) y ahí termina — **no ejecuta nada automáticamente**. Es una bandeja de pendientes manual: el usuario decide cuándo llevar ese caso a una sesión de Code para resolverlo. Se descartó explícitamente construir análisis o corrección programada/automática (ver `feedback_cowork_propio_shelved` en memoria — decisión de 2026-07-12 reafirmada el 2026-07-13 cuando se propuso extenderla a este caso y el usuario la rechazó).

## Un fix resuelve varios hallazgos — "no reproducido"

Un mismo fix de fondo (ej. connection pooling) puede hacer desaparecer varios hallazgos de una corrida a la siguiente (varios `full_scan` + `buffer_pool` a la vez). En vez de resolver eso automáticamente, `ReportManager.ultimo()` compara la `fecha_ejecucion` de cada fila contra la fecha de la corrida más reciente del mismo `tipo_reporte`: si quedó atrás, marca `no_reproducido=true` sin cambiar `estado` ni tocar ninguna tabla nueva (no requirió cambio de schema). La página (`reportPage.js`) atenúa esa fila (`opacity: 0.6`), le agrega el badge "no reproducido", y `marcarResuelto` pre-llena la propuesta de corrección con la fecha en que se vio por última vez — pero sigue requiriendo confirmación humana antes de cerrar el caso. Verificado end-to-end 2026-07-13 con un caso simulado (backdate de `fecha_ejecucion` + revert).

`routes/reports.js` expone `GET /reports/:tipo` y `POST /internal/reports/:tipo/run` (trigger genérico, requiere `X-API-Key`) — el módulo dueño del dominio (ej. `schemaHealth.js`) hace su chequeo y llama `ReportManager.registrar(...)` al final; coordinador simple, la lógica de qué medir vive únicamente en ese módulo, nunca fuera de `server-api`.

Cuando una fila queda `no_reproducido`, la página ofrece el botón **"Descartar (no reprodujo)"** (`POST /:tipo/:id/descartar` → `marcarDescartado`) además de "Marcar resuelto" — cierra el caso sin afirmar que se aplicó un fix (útil para falsos positivos, ver caso #239 más abajo). Si el pipeline lo vuelve a detectar, reaparece como fila nueva.

## Priorización por peso real — columnas Ejec. / Filas por ejecución

Cada fila expone, cuando la categoría lo provee (`full_scan` hoy; `tabla_grande` usa `filas` como aproximación), la cantidad de ejecuciones (`veces`) y las filas examinadas **por ejecución** (`filas_examinadas_prom = sum_rows_examined / count_star`, calculado en `schemaHealth.js`). `ReportManager.pesoRegistro()` ordena `ultimo()` por ese valor — evita que una query barata-pero-frecuente desplace visualmente a una realmente pesada. Categorías sin este detalle muestran "—". El ID de cada fila (`row.id`) también es visible en la tabla para poder referenciar un caso puntual sin abrir el JSON.

## Umbral por ejecución para `full_scan` — caso #239

La métrica original ordenaba `full_scans` por `sum_rows_examined` **acumulado**, que mezcla "corre muchas veces" con "es realmente cara por ejecución". El hallazgo #239 (una query sobre `market_sentiment` con subquery agregada) acumulaba millones de filas examinadas solo porque corría muy seguido — diagnosticado con `EXPLAIN` + `SHOW INDEX`: el `type: ALL` marcado por `performance_schema` era un artefacto de la materialización de la subquery derivada (11-40 filas reales), no una falta de índice. Restructurar la query (ej. window functions) no lo hubiera eliminado — cualquier subquery agregada produce el mismo artefacto vía otra tabla intermedia.

**Fix (2026-07-13):** `schemaHealth.js` ahora exige `sum_rows_examined / count_star >= 10.000.000` (filas por ejecución, no acumulado) y ordena por ese promedio. Verificado con corrida forzada: `full_scan` pasó de reportar falsos positivos como #239 a **0** hallazgos activos. #239 se descartó manualmente vía "Descartar (no reprodujo)".

## Reiniciar estadísticas — botón "🔄 Reiniciar estadísticas"

Visible solo en `schema_health` (`reportPage.js`, condicional por `tipo`). Llama `POST /reports/schema_health/reset-stats` → `schemaHealth.resetStats()` → `TRUNCATE performance_schema.events_statements_summary_by_digest`. Con `confirm()` explícito: afecta a **todas** las queries, no una sola — sirve para arrancar una medición limpia al validar un fix puntual, pero queries pesadas-pero-poco-frecuentes (ej. batch 13F) no reaparecen en los reportes hasta que vuelvan a ejecutarse.

## Falsos positivos de `indices_sin_uso` — tres patrones distintos (2026-07-13)

`performance_schema.table_io_waits_summary_by_index_usage` acumula lecturas/escrituras por índice **desde el último reinicio de MySQL** (no desde el botón "Reiniciar estadísticas" — ese trunca `events_statements_summary_by_digest`, una tabla distinta, usada solo para `full_scan`/slow queries). Un índice con `count_read=0` no necesariamente está muerto: puede ser real pero de uso esporádico. Depuración de la bandeja de 9 hallazgos pendientes de `indice_sin_uso` encontró tres causas distintas, cada una con su propio fix en `schemaHealth.js`:

1. **Índices UNIQUE de constraint** (ej. `ib_flex_trades.uq_transaction_id`) — MySQL no siempre los marca como "usados" en este contador aunque sí se usan para validar duplicados en cada INSERT. Fix estructural: excluir del query cualquier índice donde `information_schema.STATISTICS.NON_UNIQUE = 0` (son constraints, no índices de performance opcionales — nunca se deben poder "dropear por desuso").
2. **Verificados en uso por código real pero de baja frecuencia** (`market.idx_market_account_symbol`, `market.idx_market_ACCOUNT_tipo`) — confirmado con `EXPLAIN` + grep de `Modulos_Mysql.py` (líneas 1568 y 1580, queries `WHERE account+symbol` / `WHERE account+tipo`). El contador estaba en 0 solo porque el query path que los usa no corrió aún desde el último restart de MySQL. Fix: lista blanca hardcodeada `INDICES_USO_VERIFICADO` (`schemaHealth.js`), cada entrada con comentario apuntando a la línea de código que lo justifica — mecanismo pensado para casos puntuales verificados a mano, no para volumen.
3. **Real pero esporádico, sin verificación de código caso-por-caso viable** (7 índices `fin_*`/`ia_mejoras` del módulo Finanzas — uso manual del usuario cuando revisa gastos puntuales, "muy esporádico", no built-in a ningún batch). Validado en vivo durante la sesión: varios de estos índices pasaron de `count_read=0` a cientos de lecturas en cuestión de horas, mientras se seguía trabajando. Fix de raíz (no lista blanca — no escala): `UMBRAL_UPTIME_DIAS = 30` gatea el query completo de `indices_sin_uso` — no reporta nada si `SHOW GLOBAL STATUS LIKE 'Uptime'` indica menos de 30 días corridos, dándole tiempo a procesos de baja frecuencia a dejar rastro real antes de sospechar de un índice.

Como consecuencia de esta depuración también se dropearon en `bdinv` 6 índices confirmados como genuinamente sin uso en tablas pequeñas (`ia_trace.idx_estado/idx_timestamp/idx_vehiculo_decision`, `split.split_ticket`, `trazaplan.idx_idcuenta`, `youtube_candidatos.idx_ultima_vez`) — en tablas de pocas filas MySQL nunca los elige (full scan siempre gana), sin uso real ni potencial a ese volumen.

## `buffer_pool` — umbral de reporte (2026-07-13)

A diferencia de `full_scan`/`indice_sin_uso`/`tabla_grande` (hallazgos discretos, aparecen solo si hay un problema puntual), `buffer_pool` es una métrica continua (`en_uso_gb` / `configurado_gb`) que se registraba en **cada corrida sin condición** — ruido permanente inevitable en la bandeja aunque la DB estuviera sana. Fix: `UMBRAL_BUFFER_POOL = 0.6` en `routes/reports.js` — solo se llama a `ReportManager.registrar()` para `buffer_pool` cuando el uso real supera 60% del buffer pool configurado (señal real de presión de memoria). Estado verificado el 2026-07-13: 0.37GB en uso / 2.00GB configurado (18%) — DB bien entonada, sin ajuste de memoria pendiente.

## Cómo se suma un reporte nuevo

1. El módulo dueño del dominio se implementa **dentro de `server-api`** (igual que `schemaHealth.js`) — aunque el disparo venga de afuera (Task Scheduler u otro), el cómputo no sale de acá
2. Se registra su `tipo_reporte` en `routes/reports.js` para que `POST /internal/reports/:tipo/run` sepa qué módulo invocar
3. Al terminar, llama `ReportManager.registrar('nuevo_tipo', categoria, referencia, buffer)`
4. Ya aparece en `/reports/nuevo_tipo` — sin tocar auth, tabla ni página

## Estilo visual

La página `/reports/:tipo` es genérica — misma UI para cualquier `tipo_reporte` — así que el estilo se define **una sola vez** y no por reporte.

- **Fase de ajuste:** durante la implementación de `schema_health` (primer consumidor), el estilo quedó abierto a tuneo — colores, layout, cómo se ven las mini-series históricas, botones.
- **Congelamiento (2026-07-13):** estilo aprobado, vive en `lib/reportPage.js` (`<style>` inline, no hoja de estilos separada — la página es un único módulo autocontenido). Los reportes que se sumen después (`ib_reconcile`, etc.) lo heredan automáticamente vía la página genérica — no se rediseña por tipo.
  - **Base:** dark mode fijo (`color-scheme: dark`), fondo `#0d1117`, texto `#e6edf3`, fuente sistema (`-apple-system, Segoe UI, sans-serif`).
  - **Tabla:** filas con hover `#161b22`, separador `1px solid #21262d`, headers en mayúsculas `#8b949e` 0.72rem.
  - **Badges de estado** (`estado` del hallazgo): `detectado` = naranja (`#4d2d00`/`#f0883e`), `propuesto` = azul (`#1c3a5e`/`#58a6ff`). `no_reproducido` = badge verde adicional (`ausente-tag`, `#1a2f1a`/`#3fb950`) + fila atenuada (`opacity: 0.6`) — señal visual de "esto puede estar resuelto, confirmar antes de cerrar".
  - **Detalle JSON:** `<details>` colapsable con `<pre>` monoespaciado sobre fondo `#010409` — evita que el JSON crudo abrume la tabla por default.
  - **Botones:** estilo uniforme `#21262d` con hover `#30363d`, sin distinción visual entre acciones (todas requieren confirm() antes de disparar el POST correspondiente).
- **Cómo se prueba:** conviene iterar el look en un Artifact (prototipo rápido, sin tocar `server-api`) antes de portarlo a la página real — así se ajusta visualmente sin fricción de deploy.

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-12 | Extraído de design-schema-monitor al generalizar el patrón |
| 1.1 | 2026-07-12 | Agregada sección Estilo visual — se tunea en Fase 1 (schema_health) y se congela como estándar para reportes futuros |
| 1.2 | 2026-07-12 | Aclarado que el disparador puede ser un proceso externo ya agendado (vía `POST /internal/report`), no solo un cron interno de `server-api` — motivado por descubrir que `mysql-weekly-report` ya cubría el primer consumidor |
| 2.0 | 2026-07-12 | **Principio explícito: un solo dueño de la lógica, siempre dentro de `server-api`** — solo el disparador puede ser externo, nunca el cómputo. Usuario eligió esta opción sobre mantener `mysql-weekly-report` independiente. Endpoint de trigger nombrado: `POST /internal/reports/:tipo/run`. `mysql-weekly-report` se retira (ver [[design-schema-monitor]] para el plan de migración) |
| 2.1 | 2026-07-12 | **Backend implementado y probado end-to-end**: tabla `reportes_historial` creada en `bdinv` real, `lib/ReportManager.js` (registrar/ultimo/historico/marcarResuelto), `routes/reports.js` (`POST /:tipo/run`, `GET /:tipo`, `GET /:tipo/historico`, `POST /:tipo/:id/resolver`), montado en `server.js` bajo `requireApiKey`. Verificado vía curl contra `server-api` en PM2: trigger de `schema_health` persistió 40 hallazgos reales, lectura los devolvió correctamente. Falta: repuntar Task Scheduler, página HTML de `/reports/:tipo` (hoy JSON crudo), Cloudflare Access |
| 2.2 | 2026-07-12 | Task Scheduler repuntada (ver [[design-schema-monitor]]). Agregada columna `tablas_afectadas` — para `full_scan` se parsea `FROM`/`JOIN` sobre `QUERY_SAMPLE_TEXT` (muestra real de la query, no el `digest_text` normalizado/truncado) porque el catálogo de MySQL no expone tabla a nivel de digest; probado con el caso real de `market_sentiment` (3 tablas vía JOIN, correctamente separadas) |
| 2.3 | 2026-07-12 | `registrar()` cambia de INSERT ciego a upsert por caso activo — feedback del usuario: insertar una fila nueva en cada corrida aunque el hallazgo no cambie no tenía sentido. Probado: dos corridas seguidas dejan `total_filas = referencias_unicas = 39` (antes hubiera dado 78). Solo se inserta fila nueva para un caso realmente nuevo, o para uno que ya estaba `resuelto`/`descartado` y reaparece |
| 2.4 | 2026-07-13 | Botón "🔧 Proponer corrección" implementado (`marcarPropuesto` + `POST /:tipo/:id/proponer`) — bandeja de pendientes manual, sin auto-ejecución. Agregado `no_reproducido` en `ultimo()` — detecta hallazgos que un fix de fondo resolvió sin que el usuario tuviera que cerrarlos uno por uno; badge + fila atenuada + sugerencia pre-llenada, requiere confirmación humana. Verificado end-to-end con caso simulado. **Estilo visual congelado** (sección Estilo visual completa, ya no queda pendiente) |
| 2.5 | 2026-07-13 | Botón "Descartar (no reprodujo)" (`marcarDescartado` + `POST /:tipo/:id/descartar`) — cierra un caso `no_reproducido` sin afirmar un fix (`estado='descartado'`, distinto de `'resuelto'`). Solo visible cuando la fila ya está marcada `no_reproducido` |
| 2.6 | 2026-07-13 | ID de fila visible en la tabla. Botón "🔄 Reiniciar estadísticas" (trunca `performance_schema.events_statements_summary_by_digest`, solo en `schema_health`). Columnas Ejec./Filas por ejecución (`pesoRegistro()` prioriza por peso real, no por orden de categoría). **Umbral por ejecución para `full_scan`**: `sum_rows_examined/count_star >= 10M` en vez de acumulado — resolvió el falso positivo #239 (artefacto de materialización de subquery derivada, no falta de índice) |
| 2.7 | 2026-07-13 | Depuración completa de la bandeja `schema_health` (32 hallazgos cerrados, 6 índices dropeados en `bdinv`). Tres fixes de raíz para falsos positivos de `indice_sin_uso` (exclusión UNIQUE constraint, lista blanca `INDICES_USO_VERIFICADO`, umbral `UMBRAL_UPTIME_DIAS=30` — ver sección dedicada arriba). Umbral `UMBRAL_BUFFER_POOL=0.6` para dejar de reportar `buffer_pool` en cada corrida sin condición. Expandido detalle de Cloudflare Access + Google login en sección Arquitectura |
