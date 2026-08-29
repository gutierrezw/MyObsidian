---
tipo: backlog
tags: [gestion, pendientes]
fuente_de_verdad: true
---

# BACKLOG — AppOO Trading Platform

Historial de versiones al final del archivo.

---

## Pendiente

| # | Módulo | Tarea | Prioridad |
|---|--------|-------|-----------|
| 🧪 75 | **Stock/Reconcile** | **`Agente_LotesReconcile` — detección automática de desfase lotes vs posición IB** — Detectar cuando `SUM(cantidad WHERE activa='Y' AND codigo='O')` ≠ `inversion.position` por símbolo. Motivado por VGSH (2026-07-22): el campo `booktrading.stock` final era correcto (14) pero la suma de lotes activos daba 18 — el desfase solo se notó por casualidad en una captura de pantalla. **Contexto del bug VGSH:** SELL -4 del 2026-03-20 no marcó el lote Jan-27 (5 shares) como `activa='N'`; hubo que partir manualmente el lote en 4-cerrado + 1-abierto. **Dos chequeos necesarios (el segundo es el que faltaba):** (1) `booktrading.stock` último registro vs `inversion.position` — ya existe en `AppTest/run_monitor_booktrading.py`; (2) `SUM(cantidad) WHERE activa='Y' AND codigo='O'` vs `inversion.position` — el que detecta el bug de lotes. **Implementación:** lógica en `Class_customer.py` o `Modulos_Mysql.py` método `check_lotes_vs_position(account, divisa)` → retorna lista de dicts `{simbolo, lotes_sum, ib_position, delta}`. Agente en `Class_AgentManager.py` `@wait_rate(86400, persist=True, desc="Reconcile lotes vs IB (24h)", nivel=2)` → llama al método → si hay deltas ≠ 0 → log WARNING con detalle por símbolo + `DataHub.system_alerts`. **Excluir:** `divisa='ARS'` (FCI — lotes no aplican igual), `delisted=1`, `codigo='C'`. **Reutilizar:** lógica de `run_monitor_booktrading.py` para obtener `inversion.position`; el chequeo (1) puede foldearse en el mismo agente para unificar. | **Alta** |
| 74 | **Stock/Performance** | **TWR real para cartera Stock** — reemplazar `CumPort = (performa / raiz) - 1` (ratio simple) por TWR con flujos reales de capital desde `booktrading`. El ratio simple sobreestima rendimiento cuando se inyecta capital nuevo. Cashflows = compras netas en USD identificadas en `booktrading` (compras de nuevas posiciones con capital fresco). Complejidades a resolver: dividendos (¿retorno o capital?), multi-divisa (CAD/EUR), splits. Mismo patrón que la corrección BBVA.ARS en `Modulos_Comunes.py` rama `tipo == "Stock"`. Crypto BotCrypto excluido — capital fijo desde el inicio, distorsión mínima. | Media |
| 69 | **Infraestructura/Salud** | Tools MCP schema MySQL Fase 1 — agregar `get_schema_health` + `get_slow_queries` (read-only) en `routes/mcp.js`. Resultado del debate `debate-2026-07-12-schema-monitoring.md`. `run_schema_fix` pausado hasta caso de uso real. Logear en `mcp_audit.jsonl`. | Media |
| 70 | **Infraestructura/Salud** | Report Center — motor genérico `reportes_historial` + `ReportManager` + `GET /reports/:tipo` (página HTML) + Cloudflare Access. Primer consumidor: `schema_health` (extiende Fase 1 del ítem #69, no la duplica). Diseño en `design-report-center.md`. **Backend + página + Cloudflare Access implementados 2026-07-12.** **2026-07-13:** agregado botón "🔧 Proponer corrección" (estado `propuesto`, `ReportManager.marcarPropuesto` + endpoint `POST /:tipo/:id/proponer`) — bandeja de pendientes manual, sin auto-ejecución (se evaluó y descartó automatización programada, ver `feedback_cowork_propio_shelved`). Agregada detección de hallazgos "no reproducidos" (cubre el caso de un fix que resuelve varios hallazgos a la vez): `ReportManager.ultimo()` marca `no_reproducido` cuando `fecha_ejecucion` < última corrida del mismo `tipo_reporte`; la fila se atenúa visualmente y `marcarResuelto` sugiere una resolución pre-llenada. **Verificado 2026-07-13:** "no reproducido" probado end-to-end (restart PM2 + caso simulado con backdate/revert de `fecha_ejecucion`) — funciona correctamente. `design-report-center.md` actualizado con el botón `propuesto`, la detección de ausencia, y Estilo visual congelado. **2026-07-13 (b):** ✅ umbral de severidad por ejecución implementado — `full_scan` ahora exige `sum_rows_examined / count_star >= 10.000.000` (antes ordenaba por `sum_rows_examined` acumulado, que mezclaba "corre muchas veces" con "es realmente cara"). Motivado por hallazgo #239 (falso positivo: artefacto de materialización de subquery derivada en MySQL, 11-40 filas reales, no falta de índice). Verificado con corrida forzada: `full_scan` pasó de reportar falsos positivos a `0` hallazgos activos, #239 quedó `no_reproducido`. Además: columna ID visible en la tabla (`#id`), botón "🔄 Reiniciar estadísticas" (`TRUNCATE performance_schema.events_statements_summary_by_digest`, solo visible en `schema_health`, con confirm() explicando el impacto global), y columnas "Ejec."/"Filas por ejecución" (peso real del hallazgo, `ReportManager.pesoRegistro()` ordena por `filas_examinadas_prom` en vez de mezclar categorías). **Pendiente:** (a) archivar `mysql-weekly-report/` tras validar 2-3 semanas. | Media |
| 68 | **Infraestructura/MCP** | Tools MCP bajo demanda + detección de recurrencia — agregar tools específicos en `routes/mcp.js` cuando una consulta se repite. Evitar SQL libre (riesgo Tunnel). **Criterio manual:** si Claude hizo el mismo script externo más de una vez → candidato a tool. **Criterio automático (idea):** loguear queries recurrentes en sesiones de co-work (VS Code + Desktop) → cuando un patrón se repite N veces → sugerir o auto-crear el endpoint. Objetivo: reducir fricción progresivamente sin intervención manual cada vez. | Baja |
| 18 | **Auto-Remediación** | BD: crear tablas `fallos`, `app_metrics` (`bd_metrics` se retiró — no era del dominio app, era BD/infraestructura) | Media |
| 19 | **Auto-Remediación** | Agentes: `Agente_FallosLog` + `Agente_MetricasCodigo` (`Agente_MetricasBD` se retiró — no era del dominio app) | Media |
| 22 | **Auto-Remediación** | UI tab System: panel "Fallos & Métricas" — treeview fallos + resumen calidad de código | Media |
| 38 | **Gmail/Productividad** | Depuración bandeja Gmail con Claude Desktop: clasificar, etiquetar y archivar correos masivos; definir reglas de limpieza recurrente; usar MCP Gmail tools de Claude Desktop | Media |
| 39 | **BotCrypto/Analytics** | `run_bot_analytics.py`: parsear JSON técnico en `booktrading` (RSI/MACD/ATR/Fibonacci × 3 timeframes diaria/semanal/mensual) de trades cerrados → tabla correlación condición→WR; identificar patrones de entrada ganadores/perdedores para mejorar reglas del scoring | Media |
| 51 | **IA/Investigación** | Revisar agentes financieros que ofrece Claude (Managed Agents API) y evaluar integración al proyecto de inversiones — análisis de portafolio, señales, alertas u otros casos de uso relevantes | Media |
| 52 | **Infraestructura/Agentes** | Panel Agentes — niveles jerárquicos: N1 Datos (MarketScreener, EdgarFunds, FundFilings, 13FHoldings, Sentimiento, YouTubeScanner), N2 Señales (13FScores, InstitucionalScore, ConsensoCache, InterpreteSentimiento, ClasificadorETF, StockBeta), N3 Decisiones (ManagerPreservation, SyncOrders, LtvControl, OrderEodCleanup), N4 Soporte (PriceSync, PerformaValidator, SplitsControl, ExtractosWatcher, AuditPortfolio, ApiCostTracker). Agregar campo `nivel` en `AGENTES_SCHEDULE`, agrupar por nivel en treeview con separador visual, ajustar columnas del panel (ancho + orden). | Media |
| 🧪 53 | **Stock/GainsCapture** | `Agente_GainsCapture` — espíritu especulativo (distinto a Preservation que es defensivo). Opera sobre `categoriaActivo='N'` (volátiles): venta parcial por niveles de ROI (50%/100%/150%), validación Claude de técnicos (RSI_d/RSI_w/EMA) antes de cada nivel. Dos modos: `automatico` (LMT directo a IB + notif Telegram) y `autorizado` (propuesta Telegram /ok /no, timeout 30min → cancela sin ejecutar). **En prueba desde 2026-08-06.** **✅ DESBLOQUEADO 2026-08-29.** El último hallazgo abierto de la revisión ([[resultado-revision-opus-preservation-gainscapture]]) era H5: Preservation con un STOP y GainsCapture con un LMT SELL sobre el mismo símbolo, sin consulta cruzada — hasta 133% de las acciones en ganancia comprometidas. **Cerrado con el gate cruzado**: los dos agentes consultan `DataHub.qty_comprometida_sell()` sobre `order_trader` y exigen `qty_propuesta + comprometida <= position`. En GainsCapture está en sus **dos** puertas — el loop recorta, la aprobación de Telegram rechaza (entre una y otra pueden pasar horas). OCA se evaluó y se descartó: enlaza órdenes ya enviadas y Binance no lo tiene. Diseño en [[design-gains-capture]] § "Gate cruzado con Preservation"; commits `4803517`, `e258c3d`, `049b18e`, `1328647`, `61f61e2`. **Queda en observación antes de PROD**, no en prueba bloqueada. **Corrección 2026-08-26:** este ítem decía que `maximiza_sell_lotes()` repartía mal y que "25%/33%/100%" eran etiquetas falsas — **H1 se reencuadró el 2026-08-21 como no-bug** (el reparto por conteo de lotes es el diseño original, confirmado con el usuario) y el texto quedó sin actualizar cinco días. `maximiza_sell_lotes()` está OK. **Avances 2026-08-22:** lotes en orden ROI DESC (antes entregaba los más antiguos), `lotesGain()` descuenta lo ya vendido, tope duro `vender_qty <= position` con ganancia prorrateada — esta última es la pieza que hace a H5 implementable. `min_roi` filtra lotes y `min_ganancia` filtra la clase vía `DataHub.gains_config()`. **2026-08-26:** las propuestas vencidas ahora se expiran y borran su mensaje de Telegram aunque el símbolo deje de ser candidato. Pendiente calibrar `min_ganancia` (200 sobre la clase deja solo escenario "100%"). | Alta |
| 55 | **IA/Consenso** | Tech Alignment — 7º voto Consenso: RSS TechCrunch+MIT Review → `voto_tech_alignment(symbol)`. Señal para **nuevas oportunidades** según tesis de inversión, no para gestión de cartera activa. **Diseño revisado 2026-07-23:** los feeds RSS ya vienen organizados por tema (no hay que clasificar con Haiku). Feeds directos: `techcrunch.com/category/artificial-intelligence/feed/`, `/biotech-health/feed/`, `/fintech/feed/`; MIT: `/topic/artificial-intelligence/feed/`, `/biotechnology/feed/`, `/climate/feed/`. Sin suscripción, gratis. **Lógica simplificada:** si feed temático tiene ≥ N artículos nuevos hoy → tema "caliente" → símbolos mapeados a ese tema reciben +1. Haiku no necesario para clasificar artículos. **Pendiente clave:** mapeo símbolo→tema — no sale directo de `sector`/`industry` Yahoo. Opciones: (a) tabla manual, (b) Haiku infiere una vez por símbolo y se persiste en BD. `Scanner_Tecnologias.py` en `ConvergIA/` aún no existe. | Baja |
| 56 | **Stock/Oportunidades** | Re-entry Scanner — detectar ex-activos con condiciones mejores que la entrada original: ROI histórico > 0 + consenso favorable + precio actual < avgcost histórico. `get_reentry_candidates()` en `PlanInversion` + `Agente_ReentryScanner` @24h → `DataHub.system_alerts`. Diseño en `Doc/reentry_scanner_design.md`. | Media |
| 58 | **Infraestructura/IB** | Migrar `Class_Ibrks.py` legacy a librería oficial de IB — intento anterior revertido por complejidad. Arrancar en rama separada, validar con `AppTest/run_ib_websocket.py` antes de tocar main. La reconexión automática (`_tickle_loop`) debe quedar igual que hoy. | Baja |
| 71 | **FCI/BrowserFCI** | **Primera automatización ejecución FCI** — Santander: traspaso directo entre fondos vía Playwright. BBVA: rescate + suscripción separados. Canal Telegram como aprobador (propuesta → /ok → ejecuta). Candidato destino por `select_fci_rf_candidato` (más deprimido). Todo en `Class_BrowserFCI.py`, reutilizando perfiles Chrome y credenciales existentes. **Pendiente:** guide & capture de ambos bancos + separar señal RV-captura vs RF-acumulación. Idea en `10-Memoria/idea-fci-browser-ejecucion.md`. | Alta |
| 72 | **FCI/IA** | **Contexto FCI en Claude IA** — agregar bloque FCI en `_armar_contexto_ia()` (`Class_DashBot.py`): spread actual por fondo (vs_piso, vs_techo, señal), candidato RF más deprimido (`select_fci_rf_candidato`), señales RV con ganancia + destino de rotación sugerido. Requiere método en `AnalisisFCI` que devuelva dict de contexto sin abrir ventana. Objetivo: Claude puede recomendar rotación específica ("rotá hacia FBA Renta Pesos, está 24.5% bajo el piso"). Primer producto para cuenta supervisada. | Alta |
| 62 | **Infraestructura/Migración** | Mantenimiento versiones Node.js — definir proceso equivalente a Python: evaluar `nvm-windows` como gestor de versiones, documentar versión mínima requerida por `server-api`, validar PM2 + dependencias npm tras cada upgrade. Ref: [[ref-instalacion]] | Baja |
| 73 | **FCI/DiariaCNV** | **Bug: días faltantes en `diaria_cnv` cuando CNV publica tarde** — El `Agente_DiariaCNV` corre durante el día; si CNV publica la planilla de Fecha N después del horario de corrida, ese día queda con datos parciales o sin datos para algunos fondos. `backfill_historico_cnv` solo aplica para fondos nuevos, no para gaps de fondos existentes. **Fix temporal:** threshold 30% en TWR (Modulos_Comunes.py) anula saltos imposibles causados por precios faltantes. **Fix definitivo:** al terminar el download diario, verificar si algún CAFCI activo tiene gaps en los últimos 3 días; si los hay, descargar la planilla de esa fecha y usar `update()` en vez de `insert_CNV()` (para que sobreescriba registros parciales). Ver `insert_diaria_CNV()` — usa `if not found → INSERT`, ignorando registros parciales. | Media |
| 79 | **UI/Style Guide** | **Migrar resto de la app a `Flat.TButton` + Segoe UI 9 + Entry con bg/fg explícitos + `ui_section_bar()` para sesiones** — hoy migradas Editar Plan, Editar Vehículo, botón Cancel de Diversificación y Variables de Entorno completa (ver `spec-style-guide.md`). Migrar el resto (Screener, Consenso, popups de órdenes, Setup general, etc.) de a una, cuando se toquen por otro motivo — no es un sprint de refactor dedicado. | Baja |
| 80 | **UI/Style Guide** | **Mover `font_base`/`btn`/`entry` al JSON de `DataHub.colors` en BD** — hoy `Flat.TButton` y el font Segoe UI 9 son constantes en `style_app()` (`Modulos_Utilitarios.py`), no forman parte del JSON configurable de la sesión `DataHub` (ver [[ref-datahub]]). Requiere `UPDATE` sobre esa sesión en producción — evaluar antes de hacerlo. | Baja |
| 81 | **IA/Agente_ClaudeIA** | **Correcciones previas a integrar riesgos/`salida_emergencia` en Capa 4** — bloqueado hasta resolver, tras revisión crítica independiente ([[prompt-revision-opus-riesgos-salida-emergencia]], ver [[debate-2026-08-19-integracion-tabs-datahub]]). (1) **Corregir modelo mental del rollout** — `Agente_ClaudeIA` corre solo en Stock (`Class_DashBot.py:93,100-102,545,570,861,2615`), no hay 3 vehículos entre los que "propagar"; el plan de rollout debe partir de eso. (2) **Verificar ancho real de columna `variablesplan.ditem`/`observaciones`** — hoy truncados a 50 chars en `Modulos_Mysql.py` (`update_variablesplan_item()` líneas 3766-3772, `insert_variablesplan_item()` líneas 3783-3790); insuficiente para un criterio de decisión (~250 tokens máx sumando ambos campos), evaluar subir el límite (posible migración de columna). (3) **Definir el gate ejecutable de `salida_emergencia`** — sobre las columnas `valor`/`unidad` de `variablesplan` (hoy hardcodeadas a `0`/`''` en el insert), no solo como texto en el prompt; debe decidirse antes de tocar `_armar_contexto_ia()`/`_claude_ia_eval()`, con vista a dónde engancha (antes de `_claude_ia_eval()` en Fase 1, antes de `execute_order()` en Fase 3). (4) **Trazabilidad real, no fake** — `gates_ok` hoy es `{}` literal (`Class_DashBot.py:575`); si se agrega, debe ser un campo que el propio modelo declare en el JSON de salida (ej. `riesgos_aplicados`, contrato en líneas 878-880), no algo recalculado en Python post-hoc a partir de lo que se envió al prompt. **Colateral, no bloqueante:** verificar si el callback Telegram `ia_ejecutar\|{trace_id}` (Fase 2 "Voz", `Class_DashBot.py:589`) realmente ejecuta algo o el botón "✅ Ejecutar" no hace nada todavía. | **Alta** |
| 🧪 76 | **Stock/Preservation** | **Preservation Fase 1 mejorada — configuración dinámica + validaciones de mercado** — Implementado 2026-08-03: (1) **gainInversion en $** — usar campo `sesion.gainInversion` (dólares absolutos, no %) como ROI mínimo requerido antes de vender. Stock=$70, Crypto=$20 (valores vigentes en la columna; el $100 original se bajó). Garantiza ganancia mínima despues de comisiones. (2) **Registro de cancelaciones** — cuando Preservation cancela una orden por ROI < roi_minimo, actualizar `order_trader` con status=CANCELED para que aparezca en lista de órdenes. (3) **Validación RSI semanal** — solo permitir venta si RSI_semanal >= 70 (overbought). (4) **Validación precio mínimo** — evitar venta forzada durante correcciones: Stock=$50, Crypto=$0.001. (5) **Reducción de logs** — comentar mensajes [CLAUDE] y [ATR-CAP] intermedios que no resultan en orden. **Estado:** En observación. Ahora ejecutando correctamente (2026-08-13 07:31): Preservation(Stock) roi_min=10% prot=40%, Preservation(Crypto) roi_min=18% prot=40%. Validar 1-2 semanas en producción. **✅ DESBLOQUEADO 2026-08-27.** La revisión crítica independiente ([[resultado-revision-opus-preservation-gainscapture]]) abrió cinco hallazgos sobre este ítem (H3 ratchet, H4 cancel+send, H6 Crypto en DRY-RUN, H7 `PRECIO_MINIMO`, H8 fallback SMA20). **Los cinco están cerrados** — ver la tabla "Cerrados" del resultado. Preservation queda sin bloqueantes ni advertencias, acotado a Stock. **Decisión H7 (2026-08-26):** el punto (4) de este ítem — validación de precio mínimo — queda **eliminado**, no reconfigurado. `PRECIO_MINIMO = 50.0` salió de `_preservation_run_vehiculo`. El filtro económico de Preservation es el punto (1), `unrealizedpnl < sesion.gainInversion` (Stock $70, Crypto $20): mide dólares en juego, no precio unitario, y ya es configurable por vehículo. El piso de $50 era el mismo criterio mal medido y dejaba sin protección al perfil de precio bajo que GainsCapture sí opera. **Decisión H6 (2026-08-21):** Crypto sacado explícitamente del loop de `Agente_ManagerPreservation` (`Class_AgentManager.py:782`, `for vehiculo in ("Stock",):`) — antes corría siempre en `[DRY-RUN]` sin protección real; se prefiere no simular en silencio. **Prueba en curso queda acotada a Stock únicamente.** Retomar Crypto cuando Stock esté sólido. GainsCapture (#53) sí cubre Stock+Crypto — no confundir alcance entre los dos agentes. | **Alta** |

| 82 | **Crypto/Performance** | **`diaria_performance` no refleja las ganancias realizadas de Crypto/BotCrypto — diagnóstico previo a cualquier reproceso** — Detectado 2026-08-22 al medir el impacto del fix de comisiones. Cruce de las 181 combinaciones fecha+símbolo con ventas contra `diaria_performance`: **82 no tienen fila** (4 Crypto + 78 BotCrypto), 78 tienen fila con `gyp_dia` distinto, solo 21 coinciden. GyP que nunca llegó a la diaria: **+$186.95 Crypto / −$78.25 BotCrypto**. Ejemplos: ADAUSDT 2025-09-18 tiene $75.00 realizados y `gyp_dia = 0`; ICPUSDT 2026-06-03 tiene −$8.96 y `gyp_dia = 0`; las ventas de BNB (2024-04-19 y 2026-08-22) no tienen fila. **Causa probable:** `acumula_igual_date()` en `Modulos_Comunes.py:275-284` solo acumula `gprealizadas` cuando la fecha del booktrading coincide con una fecha presente en los datos de mercado del símbolo — si ese día no vino de Yahoo (hueco, o el símbolo dejó de descargarse), la venta se pierde: no hay fila donde escribirla. **NO es consecuencia del fix de comisiones** (ese movió $1.50 en total sobre 485 filas). **Alcance de este ítem: solo diagnosticar** (¿huecos de Yahoo? ¿símbolos que dejaron de descargarse? ¿la diaria arrancó después de la venta?) sin tocar datos. **Por qué no se reprocesa de una:** rehacer esta historia arrastra **extractos, diaria y performance** — son tres capas encadenadas, no una tabla. Además 82 combinaciones no tienen fila que actualizar y crearla exige reconstruir `AdjClose`/`value`/`costo_base` de ese día. Y no tiene sentido reconstruir con el mismo código que produjo el hueco: primero arreglar `detalle_book`, o el hueco se repite. **Reutilizar:** `IPerformance.purgar_desde(account, vehiculo, desde)` y `AppTest/run_purgar_performance_desde.py` ya existen para la fase de reproceso, cuando se llegue. | Media |

| 83 | **Venta/Umbrales** | **Calibración de umbrales de venta contra `booktrading` — medición por ventana, no histórica** — Herramienta: `AppTest/run_booktrading_roi.py` (solo lectura), acepta ventana `6m` / `ytd` / `12m` / fecha y reporta por vehículo la distribución de ROI y ganancia **por orden** (símbolo+día, que es lo que mide `min_ganancia`) y cuántas habrían pasado cada umbral. Ver sección 3.5 de [[design-autonomo]]. **Automatizado 2026-08-25:** `Agente_RoiVentas` (`Class_AgentManager.py`, `@wait_rate(2592000)`, ventana fija 6m) corre la misma medición para Stock y Crypto con los umbrales vigentes de `DataHub.gains_config()` y la deja en el log `Agente.Infra`; lógica en `PlanInversion.roi_ordenes_venta()`. Qué se hace con la serie queda pendiente. **Regla que queda asentada:** ningún umbral de venta se mueve sin correr esta medición sobre la ventana vigente — el histórico completo arranca en 2020 y describe una operación que ya no existe (otro capital, otro tamaño de posición, ninguna de las herramientas actuales). **Estado 2026-08-25:** *Stock* medido sobre 2026 (44 órdenes) — ganancia mediana 110 USD, `min_ganancia`=90 deja pasar 64% de las órdenes y `min_ganancia`=200 cae justo en el p90 (4 de 44): **los dos valores quedan validados**, y con esto se corrige —no se borra— la nota del ítem #53 sobre que 200 dejaba solo el escenario 100%, que venía de leer el histórico. *Crypto* no tiene muestra: 2 órdenes en todo 2026 (BTCUSDT y BNBUSDT, 22-23 de agosto, menos de 30 USD cada una) — `min_ganancia` 150/300 queda **sin evidencia ni a favor ni en contra**, se revisa cuando haya ventas del año, no por analogía con Stock. *Escalón de ROI* (×2.2 Stock / ×1.5 Crypto): medido y **cerrado, queda como está**. **Siguiente paso natural:** cuando Oportunidades/GainsCapture/Preservation acumulen ventas propias, la misma medición pasa de describir lo que hizo el usuario a describir lo que hicieron los módulos — que es el dato con el que se calibra el modo autónomo. | Media |

| 🧪 84 | **Infraestructura/Reloj** | **Deriva del reloj del sistema — 690 ppm sin causa identificada; dos capas de compensación en observación** — Detectado 2026-08-27. El reloj de la máquina se desincronizaba al punto de invalidar las firmas de Binance (`-1021 INVALID_TIMESTAMP`), cortando la operación. **Diagnóstico:** tres servidores NTP independientes coincidiendo dentro de 5 ms establecieron un offset real de −240 ms y una deriva de **~690 ppm ≈ 59 s/día**. La causa de que ningún ajuste previo funcionara: **el slew máximo de W32Time es ~500 ppm**, así que a 690 ppm el servicio no puede converger por corrección gradual, sin importar el intervalo de sondeo. Evidencia de apoyo: evento 134 (fallo DNS al arranque con backoff de 15 min), tarea nativa `SynchronizeTime` semanal fallando con 1056, `VMICTimeProvider` en loop sobre hardware no-Hyper-V. **Capa 1 (SO) — cerrada:** `MaxAllowedPhaseOffset = 0` fuerza corrección **por salto** en vez de slew; sondeo cada 64 s contra 3 IPs literales (evita el backoff por DNS); `ResolvePeerBackoffMinutes=2`, `VMICTimeProvider` desactivado; servicio a `delayed-auto` vía `sc.exe config` (`Set-Service -StartupType AutomaticDelayedStart` no existe en PowerShell 5.1). Watchdog `Scripts/time_watchdog.ps1` como red de seguridad, registrado en Task Scheduler como `AppOO-TimeWatchdog` (cada 10 min + al arranque, como SYSTEM), log en `C:\Users\InversionesWildaga\Documents\deploy\tmp\time_watchdog.csv`. **Capa 2 (app) — cerrada:** clase `BinanceTime` en `Class_ApiBinnace.py` — offset contra `GET /api/v3/time` compensado por round-trip, caché perezosa de 300 s con `requests.Session`, reintento diferido 30 s si no hay red, e invalidación automática al recibir `-1021`. Los 12 puntos de firma migrados. Costo medido: 0.755 µs con caché, 321 ms al refrescar, 12 requests/hora por `base_url` (~0.003% del rate limit). **Efecto verificado 2026-08-27 07:00:** offset estabilizado en **47 ms**, que calza con la aritmética del diente de sierra (0.69 ms/s × 64 s = 44 ms) — la deriva sigue intacta, pero W32Time la resetea cada sondeo y nunca se acumula. Contra el `recvWindow` de Binance (5000–8000 ms) eso es 0.6% del presupuesto. **Lo que queda abierto — la causa física.** Candidatos: pila CMOS agotada, C-states del CPU afectando el temporizador, HPET defectuoso (`bcdedit /set useplatformclock true`). Hoy está compensada, no resuelta: si el watchdog o W32Time se caen, el problema vuelve entero. **Qué observar:** filas `RESYNC` en el CSV. Con umbral de 0.25 s contra un pico esperado de 44 ms hay 5× de margen, así que un `RESYNC` significa que W32Time falló de verdad (red caída, servicio detenido, DNS), no ruido normal. Revisar en 1–2 semanas: si el CSV es todo `OK`, la capa 1 se sostiene sola y la causa física puede quedar como deuda conocida; si aparecen `RESYNC` recurrentes, escalar al hardware. Documentación completa en `AppOO/Scripts/README_time_watchdog.md`. | Media |

---

## Historial

### v5.0 — 2026-08-20
**"Riesgos del Plan" — CERRADO:**
- ✅ ítem 57 — Panel UI editable para restricciones de cartera del agente: implementado vía `edit_plan()`/`_edit_riesgos()` en `Class_gestion.py`, persistido en `sesion.parameters.agente_ia.plan` (no `llave_privada`, nombre de columna corregido respecto al ítem original), editable desde UI sin reiniciar app. Sesión 2026-08-19 extendió esto a plan compartido entre Stock/Crypto/BotCrypto (`_sync_plan_restricciones()`, commit `9a1a1b3`) y conectó las barras KPI Leverage/Deuda Total del panel superior de `DashMain.py` a `deuda_max_pct` (commit `71a4aa7`). `leverage_max` y `risk_real_max` quedan documentados como pendientes de aclarar semántica (ver `10-Memoria/sesion-2026-08-19-kpi-panel-deuda-max-pct.md`) — no bloquean el cierre de este ítem, que era sobre la existencia del panel editable.

### v4.9 — 2026-08-15
**Gate Crypto 24/7 en performa_inversion — CERRADO:**
- ✅ ítem 78 — Desfase performa_inversion Crypto: root cause definitivo era `get_ultimo_dia_mercado(market="Crypto")` (Class_DataFrame.py:626) dependiendo de la serie `BTC-USD` de Yahoo para decidir "hay día nuevo" — un hueco en esa serie (falta fila 08-14) dejaba el gate trabado, aunque las diarias de booktrading ya estaban listas. Fix (commit ad786aa): Crypto ya no descarga el índice para el gate, retorna `hoy - 1 día` directo — el índice de referencia real (`p_referencia`) se sigue descargando aparte, sin cambios, en `crea_dataframe_index()`. Mismo bug duplicado en `Class_BotCryptoUI.py:1819`, corregido por ser la misma función compartida. Validado: usuario reinició la app y confirmó "funciono".

### v4.8 — 2026-08-04
**Gráficos Diversificación — fix NaN en lugar correcto:**
- 🧪 ítem 77 — Gráficos de rentabilidad (sesión 2026-08-04):
  - 🐛 **Root cause identificado:** `draw_rentabilidad()` en Class_DataFrame.py usaba `datos["Close"].iloc[-1]` sin validar NaN. Cuando yfinance devuelve último registro vacío (común en PLUG), falla silenciosamente.
  - ✅ **Fix aplicado (commit 3ff3892):** Filtrar `datos_valid = datos[datos["Close"].notna()]` antes de usar price_now. Valida que datos_valid no esté vacío.
  - ✅ **Stock (PLUG):** Gráfico de rentabilidad ahora se muestra ✓
  - ❌ **FCI (FBA HORIZONTE):** Aún no muestra gráfico. Investigando: `get_yf_CNV()` retorna DataFrame vacío. Pendiente verificar si hay registros en `diaria_cnv` para ese símbolo.
  - 📝 **Cambios innecesarios:** Commits a345dc3, a5aa55c, 4480dac (debug/fixes en Class_customer) no eran necesarios — el problema estaba en Class_DataFrame.

### v4.7 — 2026-08-03
**Preservation Fase 1 mejorada + Gráficos Diversificación — en observación:**
- 🧪 ítem 76 — `Agente_ManagerPreservation` enhancements (2026-08-03):
  - ✅ **Configuración dinámica:** usar `sesion.gainInversion` (dólares) en lugar de `parameters.preservation.roi_minimo`. Stock=$100/Crypto=$20 — garantiza ganancia mínima absoluta antes de comisiones.
  - ✅ **Registro de cancelaciones:** cuando Preservation cancela orden por ROI < threshold, ejecutar `update_order_trader_by_client_id()` con status=CANCELED. Ahora cancelaciones aparecen en lista de órdenes (rojo, CANCELED).
  - ✅ **Validación RSI semanal:** agregar gate `RSI_semanal >= 70` (overbought) antes de crear orden de venta. Evita venta en debilidad.
  - ✅ **Validación precio mínimo:** agregar floor por vehiculo — Stock=$50, Crypto=$0.001. Cancela órdenes pendientes si precio cae por debajo.
  - ✅ **Reducción de ruido:** comentar logs [CLAUDE] y [ATR-CAP] intermedios (solo show cuando orden aceptada).
  - 🧪 **Estado:** En observación — implementado y testeado (BP venta, cancelación con precio floor), pero NO committeado. Validar 1-2 semanas antes de hacer commit. Changes en `Class_DashBot.py` líneas 1287-1482.

- 🧪 ítem 77 — Gráficos de rentabilidad (Diversificación popup) — normalización vehiculo params (sesión 2026-08-02, varias iteraciones):
  - 🐛 **Bug raíz:** parámetros vehiculo inconsistentes — algunos módulos usaban "hist"/"download" (inválidos), otros "Stock"/"Crypto". Gráficos no se mostraban para Stock/Crypto.
  - ✅ **Fixes aplicados:** `dividends_rendimiento.py:30` ("hist"→"Stock"), `Modulos_Comunes.py:573` (typo "donwload"→"download"), `Class_customer.py` líneas 4821/4823/4754 (mapeos correctos Stock/Crypto).
  - ✅ **Validación yfinance:** confirmado que `yf.Ticker().history()` SÍ retorna Dividends; `yf.download()` NO retorna Dividends (behavior esperado).
  - ❌ **Rejected approach:** agregación de validación `_vehiculo` en `ts_yfinance_symbol()` → causó cache duplicates. Descartado.
  - ✅ **Datos:** `diaria_cnv` contiene 412 registros — información presente.
  - 🧪 **Estado:** En observación — múltiples intentos/reversiones durante debug. Cambios NO committeados ("no confirmes tengo otro ajuste"). **Pendiente:** (a) validar que 3 gráficos se renderizan (rentabilidad, dividendos, acumulado) para Stock/Crypto/FCI en próxima sesión; (b) si falla, diagnosticar con `get_yfinance()` access pattern en `Class_AnalisisDiversificacion.window_estrategia()`.

### v4.6 — 2026-07-23
- ✅ ítem 75 — `Agente_LotesReconcile` @24h — detecta desfase `SUM(cantidad WHERE activa='Y' AND codigo='O')` vs `inversion.position` por símbolo. Diagnóstico: cruce con `ib_flex_trades` (últimos 90d). Log WARNING por delta + `DataHub.system_alerts`. `check_lotes_vs_position(account)` en `PlanInversion` (`Modulos_Mysql.py`). Excluye FCI (`divisa='ARS'`), delisted, `codigo<>'O'`.
- ✅ ítem 67 — Cerrado sin implementación de panel UI: cubierto por `Agente_LotesReconcile` (#75) que detecta desfases y envía diagnóstico (Flex 90d) directo a Alertas + Telegram. Panel visual redundante.
- ✅ Tab "Alertas" en System Status — visualiza `DataHub.system_alerts` en tiempo real (refresh 5s), botón Limpiar. Complementa el botón Telegram "🚨 Alertas".
- ✅ Preservation fixes (2026-07-22): `write_json_tmp` loguea error en vez de falla silenciosa; snapshot unificado preserva `_last_run_` en cada write; `Decimal` → `float` en audit trail `insert_preservation_order`.
- ✅ RSI en mensajes Telegram — BUY y SELL incluyen `RSI d/w` con emoji semáforo (BUY: 💚<30/🟢<45/🟡<60/🟠≥60; SELL: 💚≥70/🟢≥60/🟡≥50/🟠≥40/🔴<40). SELL agrega ⚠️ si RSI<45 (vendiendo en debilidad).
- ✅ Importe en mensajes Telegram — `cantidad × precio` en BUY y SELL (útil para FCI en ARS).

### v4.5 — 2026-07-13
- ✅ ítem 70(b) — umbral de severidad por ejecución para `full_scan`: `sum_rows_examined / count_star >= 10.000.000` en vez de `sum_rows_examined` acumulado (`schemaHealth.js`). Motivado por el hallazgo #239, diagnosticado con `EXPLAIN`+`SHOW INDEX` como falso positivo (artefacto de materialización de subquery derivada, no falta de índice real). Verificado con corrida forzada: `full_scan: 0` post-fix, #239 pasó a `no_reproducido`.
- ✅ Report Center UI — columna ID visible (`#id`), botón "🔄 Reiniciar estadísticas" (trunca `performance_schema.events_statements_summary_by_digest`, solo en `schema_health`, confirm() explícito), columnas "Ejec."/"Filas por ejecución" para priorizar atención por peso real de la query (`ReportManager.pesoRegistro()`), no por recurrencia entre corridas.

### v4.4 — 2026-07-13
- 🐛 **Root cause `full_scan` en `diaria_performance` resuelto — no era falta de índice.** El índice ya existía y el optimizer lo usaba correctamente; el diagnóstico real fue ausencia de connection pooling en `Modulos_Mysql.py:connect_dbase()` (~150 callsites abriendo/cerrando conexión cruda por llamada, evidenciado por millones de `SET NAMES`/`SET AUTOCOMMIT` acumulados). Fix aplicado en sesión VS Code separada con `DBUtils.PooledDB` (pymysql no trae pooling nativo), sin tocar los ~150 callsites (`.cursor()`/`.close()` siguen funcionando igual, `.close()` devuelve la conexión al pool). Verificado con medición en vivo (delta de `SHOW GLOBAL STATUS LIKE 'Connections'` en ventana de 30s, no promedios acumulados de `performance_schema` que tardan en reflejar la mejora): antes ~25 conexiones nuevas/seg, después 0/seg sosteniendo 129 queries/seg reales. `performance_schema.events_statements_summary_by_digest` reseteado para baseline limpio post-fix. Cerrado en Report Center vía `marcarResuelto` (id 241) con la narrativa completa en `propuesta_correccion`.
- ✅ ítem 70 — Report Center: botón "🔧 Proponer corrección" (bandeja de pendientes manual) + detección de hallazgos "no reproducidos" (ver detalle en la fila del ítem). Diseño ya preveía este botón exactamente así (`design-report-center.md` v1.0) — se reusó sin inventar mecanismo nuevo.
- 🚫 Automatización de análisis programado — evaluada y **descartada explícitamente por el usuario** ("no!!") tras recordarle el precedente `feedback_cowork_propio_shelved` (2026-07-12). Se mantiene la bandeja manual como único mecanismo; no reabrir esta idea sin que aparezca una razón nueva.
- ⏳ Pendiente inmediato: verificar "no reproducido" end-to-end (falta `pm2 restart server-api` + prueba con datos reales) y actualizar `design-report-center.md` (documentar botón + detección de ausencia + congelar sección Estilo visual, pendiente desde v1.1).

### v4.3 — 2026-07-12
- 🏷️ Taxonomía de dominio establecida (creciendo rápido, necesario para no re-mezclar): **Auto-Remediación** (fallas de AppOO app) / **Infraestructura/Salud** (monitoreo continuo — BD hoy, ítems 69/70) / **Infraestructura/Migración** (cambios puntuales de entorno — Python/Node/MySQL/máquina, ítem 62). Categoría de 69/70 renombrada de `Infraestructura/MCP` → `Infraestructura/Salud`; 62 renombrada de `Infraestructura` → `Infraestructura/Migración`.

### v4.2 — 2026-07-12
- ✅ ítem 21 — Depuración imports `Modulos_python.py`: vulture 69 → 0 hallazgos (commit 777ea82)
- ✅ ítem 28 — `Agente_NtpCheck` @5min en AgentManager. Alerta Telegram si deriva >500ms.
- ✅ ítem 65 — Cloudflare Tunnel activo como servicio Windows. Dominio `wildaga.com`. Rutas `api-main.wildaga.com` + `api-son.wildaga.com` → `localhost:8050`.
- ✅ ítem 66 — OAuth en server-api: Authorization Code + PKCE. Discovery `/.well-known/oauth-authorization-server`. Registrado en claude.ai Connectors.
- ✅ ítem 64 — Retrain modelos BUY/SELL con sentimiento: BUY reentrenado semana 2026-07-07 (33 filas, prerequisito ≥30 cumplido). SELL con 24 filas / 1 señal sell — reentrenar cuando acumule más ejemplos positivos.

### v4.1 — 2026-07-10
**Documentación centralizada en Obsidian + modelos IA:**
- ✅ ítem 63 — Docs conectadas a Obsidian vía URI scheme (`obsidian://open?vault=...`). Reemplazada función `documentar_estructura` (popup+BD BLOB) por apertura directa en Obsidian. Estructuras: DataHub, Cache, BuySell, Rebalanceo, Screener, modelos IA (Buy/Sell). Vault configurable en `profiles/*.json` → `APPOO_OBSIDIAN_VAULT` env var. Hijo con vault vacío = botón silencioso.
- ✅ Toggle "Modelo ON / Etiquetando" en tabs Buy IA y Sell IA — botón rojo cuando modo etiquetado activo. Persiste en `paramts` JSON de BD.
- ✅ "Ver Docs" en Buy IA / Sell IA → abre `ref-modelos-ia` en Obsidian.
- ✅ Ventana parámetros modelo simplificada — solo Modelo (readonly) + checkbox Modo Etiquetado + JSON parámetros.
- ✅ `documents` column eliminada de `insert_modelo_ia` / `update_modelo_ia` (`Modulos_Mysql.py`). SQL: `ALTER TABLE bdinv.modelos_ia DROP COLUMN documents;`

### v4.0 — 2026-07-03
**server-api Fase 3 (ítems 61 + 20) — MCP server operativo:**
- ✅ ítem 61 / ítem 20 — MCP server en `routes/mcp.js` (`@modelcontextprotocol/sdk` v1.29). 6 tools: `query_portfolio`, `get_consenso`, `get_market_data`, `get_booktrading`, `execute_order` (simula por default), `get_agent_status`. Audit log en `logs/mcp_audit.jsonl`. Endpoint `/mcp` con API key en `server.js`.
- ✅ `routes/tv.js` exporta `state` — compartido con `mcp.js` para `get_agent_status`
- ✅ Userscript de contexto eliminado — primer intento de co-work, superado por MCP

### v3.9 — 2026-07-03
**server-api Fase 2 (ítem 60) + fixes TradingView bridge:**
- ✅ ítem 60 — `Class_BrowserBridge.py` reescrito: AppOO empuja a Node via `POST /internal/update`. Mini-server Python en 5051 recibe callbacks de órdenes desde Node. Puerto 5050 deprecado.
- ✅ `routes/tv.js` — nuevos endpoints `/tv/*` (position, current, price, ping, contexto, symbols, balance, order, current POST). `/internal/update` sin rate limit (valida IP). Rate limit 120 req/min solo en `/tv/*`.
- ✅ `_push()` — `json.dumps(default=str)` para serializar datetime/Decimal de MySQL (fix silencioso que bloqueaba el push de lotes)
- ✅ Script Tampermonkey tv_panel actualizado a v2.2.
- ✅ `Class_SystemStatus.py` — endpoint TradingView Server actualizado a 5051

### v3.8 — 2026-07-02
**server-api Fase 1 + fixes freeze UI + retiros FCI:**
- ✅ ítem 59 — `server-api` Fase 1 operativo en `MyNode/server-api/` (PM2, puerto 8050). Express + mysql2 + API key + rate limit. Endpoints: `/health`, `/db/portfolio`, `/db/market`, `/db/consenso`, `/db/booktrading`, `/db/extractos`, `/db/query`, `/db/diagnostics`. Config en `Claude-Cowork-Scripts/mysql_config.json`.
- ✅ `MyMessageBox.grab_release()` — fix freeze app tras cerrar diálogo (grab_set no se liberaba con withdraw)
- ✅ `start_stock` — carga BD primero, `run()` (yfinance×41) en hilo background `InitStock_BG` — mismo patrón que Crypto
- ✅ `on_treeview_select` — guard `if self.select_activo:` evita crash con selección vacía
- ✅ `construir_extracto_fci` — retiros con `abs()` para evitar negativos cuando `cantidad` es negativa (rescates)

### v3.7 — 2026-07-01
**Sentimiento integrado en modelos IA + fixes SELL/BUY:**
- ✅ ítem 48 — Módulo Sentimiento operativo: `Agente_Sentimiento` @8h + `Agente_InterpreteSentimiento` @24h. Voto `Sent` activo en Consenso. Depuración extendida a 5 meses. 40 días de histórico, 39 símbolos, 22K lecturas.
- ✅ ítem 50 — Sentimiento como feature en modelos BUY/SELL: `enriquecer_con_sentimiento()` en flujo de entrenamiento. Modelos reentrenados (SELL: 81%±7 / BUY: 81%±8 precisión). `predecir_modelo()` con guard para columnas faltantes. 2do retrain programado agosto → ítem 64.
- ✅ `Agente_ClasificadorCrypto` — reclasifica activos crypto con estrategia NULL o vieja vía Claude Haiku
- ✅ `MyMessageBox` — fix TclError bad window path (withdraw+wait_variable)
- ✅ Fix raíz SWK/CRNT: `filtrar=False` en ManagerSell/ManagerBuy + gate bypass por confianza IA
- ✅ EMA tendencia SELL corregida: `e20>e50>e100` (pico alcista = momento de toma de ganancias)
- ✅ Entrenamiento en hilo de fondo — UI no se bloquea durante RF fit
- ✅ `ref-consenso.md` creado en Obsidian — referencia simple votos/tags/screener

### v3.6 — 2026-06-23
**IB WebSocket + colores panel APIs + rename automático tickers:**
- ✅ `Class_SystemStatus.py` — colores en panel APIs: verde=conectado, rojo=inactivo (lista principal + ventana detalle). Simplificado a solo verde/rojo (sin naranja intermedio)
- ✅ `Class_ApiIBrks.py` — override `create_session()` no-interactivo: loguea banner con timestamp al logger `IBroks_Client` pero nunca llama `input()` ni bloquea el thread daemon
- ✅ `Class_websocket_IB.py` legacy movido a `AppTest/run_ib_websocket.py` — credenciales desde `BDsystem.get_sesion_by_vehiculo("Stock")`, puerto 5501, `import ssl` agregado; eliminado del raíz para evitar sesiones IB competidoras desde VS Code
- ✅ `Modulos_Mysql.py` — `MarketScreen.rename_symbol(old, new, account)`: renombra ticker en 7 tablas en una sola transacción: `market`, `booktrading`, `oportunidadesbuysell`, `order_trader`, `trazaplan`, `youtube_candidatos`, `ia_trace`
- ✅ `Class_Screener.py` — `cleanup_market` Phase 2: consulta Yahoo individual para cada `not_found`; si Yahoo redirige a nuevo ticker → `rename_symbol()` en lugar de eliminar; si no → `market.delete()` como antes
- 📋 ítem 58 anotado — migración IB legacy a nueva librería (intento anterior revertido)

### v3.5 — 2026-06-15
**FCI — Descarga automática + cierre residuales + Telegram menu:**
- ✅ `Class_BrowserFCI.py` — módulo async Playwright: `download_bbva()` + `download_santander()` con `launch_persistent_context` (sesión persistente entre runs), shadow DOM traversal para DNI BBVA, `download.suggested_filename` para conservar nombre original Santander (`movimientos-de-superfondos-*.xlsx`)
- ✅ Bloqueo en primera falla: `_check_blocked()` / `_set_blocked()` / `reset_blocked()` vía `browser_fci_blocked.json` — impide reintentos automáticos hasta intervención manual
- ✅ `Agente_BrowserFCI` en `Class_AgentManager.py` — `@wait_rate(3600, persist=True)`, ventana L-V 8:30–9:59; si bloqueado → agrega a `DataHub.system_alerts` (no push directo)
- ✅ `FinanceScreen.get_bank_credentials(bank_name)` en `Modulos_Mysql.py` — lee `login_user`/`login_pass` de tabla `fin_banks`
- ✅ `AppTest/run_test_browser_fci.py` + `run_test_browser_santander.py` — scripts de prueba con env var `APPOO_PROFILE` + `BDsystem.configure()`
- ✅ `PlanInversion.close_residual_fci(account, symbol)` en `Modulos_Mysql.py` — cierra posición FCI residual (<$5): `activa='N'` + `stock=0` en booktrading; `iactiva='N'` + `position=0` en inversion
- ✅ `Agente_MonitorBooktrading` — auto-llama `close_residual_fci()` para alertas `residual_fci`; solo notifica manualmente las de `book_stock≈0` (stocks/crypto)
- ✅ `update_FCI_en_positions()` — filtra `costobase<=5 or position<=0` en `self.ars.positions` para no renderizar residuales en el panel
- ✅ `schedule_diaria_performace` — mensaje "CNV gate bloqueado" bajado de WARNING → DEBUG (eliminado ruido cada 90s)
- ✅ Telegram menú — botón dinámico "🚨 Alertas (N)" / "🔕 Alertas"; handler `"alertas"` muestra y limpia `DataHub.system_alerts`; `/start` y `/menu` muestran el menú con botones; menú auto-enviado al iniciar el bot (sin necesidad de escribir `/menu`)

### v3.4 — 2026-06-03
**Escalonamiento de salida — diseño completo:**
- 📋 ítem 53 anotado — escalonamiento activos volátiles (categoriaActivo='N')
- ✅ Fix `otros_activos.symbol` CHAR(25) → VARCHAR(100) + `descripcion` CHAR(80) → VARCHAR(120) — evita error 1406 en insert_otros_activos con nombres de fondos FCI
- ✅ `Doc/preservation_claude_dynamic_design.md` — Fase 2 diseñada: tipo orden LMT (last×0.995), validación Claude técnicos (RSI_d/RSI_w/EMA), modos automatico/autorizado, estados normal/escalon_pendiente/esperando_reset, botón toggle panel Agentes, timeout autorizado 30min → cancela sin ejecutar, json_detalle para aprendizaje futuro

### v3.3 — 2026-06-02
**Preservation activo + fixes Lista de Ordenes + alertas IB Gateway:**
- ✅ ítem 42 — `Agente_ManagerPreservation` activo y validado: 5 STOPs colocados en primera corrida
- ✅ Fix preservation first-run skip — `_preservation_get_config` ya no retorna `False` en primera llamada; evalúa inmediatamente
- ✅ Fix duplicate STOP orders — `preservation_last_run` ahora persiste como `_last_run_{vehiculo}` en `preservation_state.json`; sobrevive cierre/reapertura del Chatbot
- ✅ Fix ventana Diversificación — tamaño ampliado `847×780` + `resizable(True,True)`; botón Cancel ya no queda cortado
- ✅ Lista de Ordenes — columna `Stop` (auxPrice de IB) para órdenes STP LMT; columna `id_enviar` eliminada de la vista; `displaycolumns` para ocultar columnas internas sin líneas separadoras fantasma; sort automático por symbol dentro de cada grupo vehiculo
- ✅ Alerta IB Gateway caído → Telegram vía `DataHub.system_alerts` (class-level queue); `_flush_system_alerts()` async en loop agentesIA; alerta reconexión exitosa en `_ib_on_reconnect`
- ✅ Preservation dinámica con Claude Haiku — `_claude_preservation_eval` + `_build_preservation_context` (DataHub tiempo real) + `insert_preservation_order`; key `ClaudeAPIP` separada; `json_detalle` en `order_trader`; `select_preservation_context` (market + sentiment, sin oportunidades)
- ✅ Lista de Ordenes — columna `IA` (🤖) con doble-click abre popup análisis Claude: stop_final, urgencia, razón, consenso, inst_score, RSI, MACD
- ✅ `Class_SystemStatus.py` — fix canvas matplotlib `fill=tk.X` para que `self.connect` (panel API) sea visible debajo del área de recursos
- ✅ `AppTest/run_preservation_eval.py` — script standalone de validación preservation con Claude; validado contra TradingView (BP, CVS, CRNT, PLUG, UUUU, VALE)
- 📋 ítem 52 anotado — niveles jerárquicos en panel Agentes + ajuste columnas

### v3.2 — 2026-05-25
**YouTube Scanner + AgentManager + Cache tab:**
- ✅ `ConvergIA/Scanner_YouTube.py` — RSS 6 canales hispanos → Claude extrae nombres → `yf.Search()` ticker → `yf.fast_info` valida → `youtube_candidatos` BD
- ✅ Tablas `youtube_canales` (score, detecciones, validados, last_scan) + `youtube_candidatos` (apariciones, confidence, canales, status: pending/approved/rejected)
- ✅ `Agente_YouTubeScanner` @wait_rate(86400) registrado en `AgentManager.register_threads()`
- ✅ Popup "Candidatos" en Screener: tabla con En Market / En Cartera; colores verde=cartera / gris=market; Comprar → market(T) + status=approved; Rechazar → status=rejected
- ✅ Deduplicación `seen_ids` = solo IDs del RSS actual (máx ~90), no acumulado histórico
- ✅ Cleanup automático cada scan: rechaza candidatos expirados (apariciones=1 AND >15d ó <3 AND >30d)
- ✅ `ApiCostTracker` filtrado por `workspace_id` — muestra costos del workspace AppOO, no org total
- ✅ Cache tab rediseñado: árbol agrupado por vehiculo (Stock/Crypto/Referencia), collapsed por defecto, OHLCV últimas 15 filas, preserva selección en refresh
- ✅ `AgentManager` 4 domain loggers (Agente.Stock/Crypto/IA/Infra) + `YouTubeScanner` registrado en `Class_debugging.py`
- ✅ `version.py` → 10.3.0 / 2026-05-25
- ✅ ítem 51 — fixes producción: `date` → `datetime.now().date()`, rate-limit Yahoo (`time.sleep(0.3)` + eliminar `yf.fast_info`), `canal_origen` busca por nombre empresa en lugar de ticker, filtra símbolos >10 chars. Pendiente: `market_cap` (Yahoo Search no lo devuelve).

### v3.1 — 2026-05-23
**Panel Agentes + fixes:**
- ✅ ítem 49 — Panel "Agentes" en SystemStatus: treeview con Intervalo/Estado/Run/Próxima/Descripción; doble-click toggle activo/inactivo; clic derecho Activar/Desactivar; "Activar todos"; persistencia en `tmp/agents_config.json`; auto-refresh cada 10s (una sola lectura de `agents_schedule.json` por tick)
- ✅ Fix Run(0): `task_name = name` alineado con nombre de hilo — contadores ahora muestran iteraciones reales
- ✅ `Agente_ManagerPreservation`: SKIP por IB offline bajado de WARNING → DEBUG (estado normal, no error)
- ✅ Docs branch limpiada: eliminados 142 archivos `.py/.bat` — `docs` ahora solo contiene `.md` y `Doc/`; resuelve conflictos al cambiar de rama
- ✅ ítem 48 fixes: voto `Sent` como columna explícita en popup Consenso; weekend guard en `Agente_Sentimiento` e `Agente_InterpreteSentimiento` (sáb/dom = SKIP)
- 🧪 ítem 50 infra: `load_sentiment_features` en `MarketScreen` + `enriquecer_con_sentimiento` en modelos sell/buy — retrain pendiente ~1 mes de histórico

### v3.0 — 2026-05-22
**Módulo Sentimiento + Keys Claude por módulo:**
- ✅ ítem 48 — `ConvergIA/Scanner_Sentimiento.py`: fetch noticias por símbolo vía `yf.Ticker.news` + clasificación Claude Haiku en batches → {sym: +1/0/-1} → `market_sentiment` BD
- ✅ `ConvergIA/Interprete_Sentimiento.py`: análisis histórico 7 días por símbolo → patrón (acumulacion/distribucion/neutro/inflexion) + interpretación en español → `market_sentiment_analysis` BD
- ✅ `ConvergIA/ThemeMapper.py`: lee BD → `voto_tech_alignment(sym, sentiment, analysis)` → prioridad patrón histórico > sentimiento puntual
- ✅ 7 métodos nuevos en `MarketScreen`: `bulk_save_sentiment`, `load_latest_sentiment`, `load_sentiment_history`, `save_sentiment_analysis`, `load_sentiment_analysis`, `sentiment_already_run_today`, `cleanup_sentiment`
- ✅ `SchemasSQL/market_sentiment.sql` — DDL tablas `market_sentiment` + `market_sentiment_analysis` (creadas en BD)
- ✅ `Agente_Sentimiento` @wait_rate(3600) + `Agente_InterpreteSentimiento` @wait_rate(86400) registrados en loop agentes
- ✅ Voto `"Sent"` agregado al Consenso en `Class_Screener.py` (logger: `Sentimiento`)
- ✅ Keys Claude separadas por módulo en tabla `sesion`: `ClaudeAPIS` (Sentimiento), `ClaudeAPIE` (ETF), `ClaudeAPIC` (Chat) — permite monitorear consumo por módulo en console.anthropic.com
- ✅ `AppTest/run_claude_api_test.py` — valida las 3 keys en una sola corrida
- ✅ `AppTest/run_tech_alignment.py` — prueba end-to-end: Scanner + Intérprete + votos resultantes
- ✅ `AGENTES_SCHEDULE` — `Agente_Sentimiento` + `Agente_InterpreteSentimiento` agregados al dashboard
- ✅ `version.py` — `10.1.0` → `10.2.0`

### v2.9 — 2026-05-21
**Infraestructura + Pipeline 13F + Modelos IA:**
- ✅ ítem 47 — `sync_fund_filings` Opción B corrió 2026-05-21 05:47, OK
- ✅ `load_screener_health` — `pendientes` y `por_renovar` filtrados por account (antes contaban universo global 98K → ahora solo ~7.7K fondos relevantes); números correctos: 389 por_renovar / 4910 pendientes
- ✅ `profiles/main.json` — `tmp_path` absoluto `deploy\tmp` (relativo `"..\tmp"` daba `MyPython\tmp` desde VS Code → log en lugar equivocado)
- ✅ VS Code terminal — venv activo automáticamente + `APPOO_TMP` inyectado vía `terminal.integrated.env.windows`; log siempre en `deploy\logs\`
- ✅ `register_thread` — agrega thread a `DataHub.procesos` e incrementa `Run(N)` en cada iteración del wrapper; todos los agentes muestran contador real
- ✅ `sync_fund_filings` — parámetro `progress_cb` opcional; `Agente_FundFilings` pasa callback que actualiza `Run(i)` cada 100 fondos
- ✅ Modelos BUY/SELL — `rango_13w_pct`, `rango_26w_pct`, `rango_52w_pct` agregados como features en `aplanar_datos_tecnicos()` y `cargar_datos()`; reentrenamiento ejecutado sin novedad

### v2.8 — 2026-05-13
**TV Panel + Consenso + Infraestructura:**
- ✅ ítem 44 — TV panel v2: toggle QTY/USD con estado persistente entre redraws; equivalente siempre visible en ambas direcciones; Crypto usa `toFixed(4)` (no `Math.floor`); minimize no se pierde en el redraw 3s; Crypto incluido en chips cartera
- ✅ `stop_tv_server()` — `shutdown()` antes de `server_close()`: elimina `WinError 10038` al cerrar la app
- ✅ `refresh_consenso_tags` — incluye voto Mod (7 votos unificados con panel Consenso); `_load_csv_signals()` extraída como función de módulo
- ✅ Notificación Telegram: denominador corregido `/6` → `/7`
- ✅ `Panel Crypto mrg (FALLBACK equity)` — bajado de WARNING → DEBUG (no ensucia log mientras LtvControl no corrió)
- ✅ `profiles/main.json` + `profiles/hijo.json` — `tmp_path` corregido a `AppOO\tmp` (era `dist\AppOO\tmp`)
- ✅ `edgar_13f.py` — `_13F_SAVE_DIR` usa `APPOO_TMP` env var (fix path en build PyInstaller)
- ✅ `Class_debugging.py` — log path derivado de `APPOO_TMP` (unifica logs en `MyPython\logs\` siempre)
- ✅ `buildExe.bat` — `%~dp0` + backup/restore `tmp/` antes/después del build

### v2.7 — 2026-05-12
**Releases + UI + Infraestructura:**
- ✅ ítem 45 — `DashMainV9_ia.py` → `DashMain.py`; `version.py` con `APP_NAME="DashMain"`, `VERSION="10.0.0"`, `RELEASE_DATE`; splash y título de ventana leen de `version.py`; `DashMain_hijo.py` y `buildExe.bat` actualizados
- ✅ Splash screen: barra de progreso `ttk.Progressbar` 12 pasos — muestra avance módulo a módulo al iniciar
- ✅ Fix ventana vacía al arrancar: `self.root.withdraw()` en `__init__` justo después de `tk.Tk()`; `state("zoomed")` movido a después de `deiconify()` en `run()` — eliminado el flash de ventana en blanco
- ✅ Fix `Class_FondosInversion` — `self.path` usa `APPOO_TMP` env var con fallback `os.getcwd()/tmp` + `makedirs(exist_ok=True)` — resuelve `WinError 3` al correr desde VS Code
- ✅ Fix `tmp_old/` en git status — agregado a `.gitignore`
- ✅ BUY dialog — "Importe" renombrado a "USD"; `lbl_calc` reubicado en misma fila que entry (column=2) — recupera botón Cancel que quedaba desplazado
- ✅ ítem 43 — Modo "importe USD" implementado: `bt2` readonly Entry (QTY/USD), `lbm` Listbox, `lbl_calc` muestra qty calculado en tiempo real, `_get_qty_final()` calcula qty correcta al enviar; `_get_lot_exp()` nuevo método privado respeta decimales lotSize Crypto

### v2.6 — 2026-05-11
**FCI extractos — corrección NAV real + distribución hijo:**
- ✅ ítem 41 — Ejecutable hijo: `buildExe_hijo.bat` → `dist/AppOO_hijo.exe` onefile; `profiles/hijo.json` (Crypto, Ars, BotCrypto, Finance); tag `v9.1.0-hijo` publicado en GitHub Releases; `AppTest/export_hijo.bat` exporta schema + datos de referencia para setup BD hijo
- ✅ Fix tabs en blanco (`DashMainV9_ia.py`): eliminadas 10 llamadas `.pack()` en frames hijos del Notebook — conflicto con geometry manager de `ttk.Notebook` hacía desaparecer todas las tabs
- ✅ Splash screen minimalista al arranque: `_splash_open()` / `_splash_step()` — ventana borderless centrada, actualiza estado por módulo hasta `mainloop()`
- ✅ `tmp_path` absoluto en `profiles/main.json` — evita que el tmp se cree en `AppOO/` en vez de `dist/AppOO/tmp`
- ✅ `construir_extracto_fci` — parámetro `vehiculo` explícito; SANT0001 pasa `vehiculo="BBVA.ARS"` (único vehiculo en `performa_inversion` para ambas cuentas)
- ✅ `construir_extracto_fci` — `is_month_end` reemplazado por `groupby(to_period("M")).last()` — captura último día de mercado del mes aunque no sea fin de mes calendario; extracto siempre graba fecha fin de mes calendario
- ✅ UPDATE extractos BD — 25 registros: BBVA0001 (Nov-2024→Abr-2026) + SANT0001 (Oct-2025→Abr-2026) actualizados con `navcierre` y `costobase` reales desde `performa_inversion`

### v2.5 — 2026-05-06
**Defensa multicapa precios yfinance + Agente Splits:**
- ✅ ítem 34 — `Agente_SplitsControl` + `sync_splits()`: ya implementado y activo; fix bug timezone `datetime64[ns, America/New_York]` vs Timestamp naive en comparación de índice — `tz_localize` al timezone del índice antes de filtrar
- ✅ fix `auto_adjust=False` en `get_yfinance(vehiculo=Dividends)` — elimina corrupción de precios ADR (ABEV, BP) en raíz
- ✅ Cache quirúrgica: `DataFrameCache.add_bypass(symbol)` — invalida solo el símbolo corrupto, no todo el caché
- ✅ Cuarentena automática: 3+ purgas en 6h → `quarantine_symbols.json` → `detalle_book` saltea símbolo 24h (rompe loop infinito)
- ✅ `Agente_PerformaValidator`: log cache stats + `add_bypass` por anomalía + CRITICAL si cuarentena
- ✅ Guardian `close > basic*200` eliminado de `write_csv` — creaba gaps silenciosos incompatibles con validator
- ✅ `DataFrameCache.stats()`: size, maxsize, ttl, hits, misses, clears, bypass, symbols
- ✅ Decorator no cachea DataFrames vacíos (evita cachear errores transitorios de yfinance)
- ✅ fix `sync_splits` timezone: `pd.Timestamp(primera_compra).tz_localize(splits.index.tz)` cuando índice es tz-aware
- ✅ `AppTest/run_diag_abev_yfinance.py` — script diagnóstico reproducción exacta de detalle_book

### v2.4 — 2026-05-05
**Booktrading monitor + corrección cantidades:**
- 🐛 Root cause identificado y corregido: `insert_booktrading` leía `ustock` antes del commit → segundo insert del mismo símbolo/fecha quedaba con stock incorrecto
- ✅ Fix `Modulos_Mysql.py` — `insert_booktrading` + `insert_booktrading_bottrader`: tras INSERT+commit, UPDATE recalcula `stock = SUM(cantidad)` desde BD usando `cursor.lastrowid` — elimina el desfase sin importar cuántas transacciones del mismo símbolo se inserten en la misma fecha
- ✅ `AppTest/run_monitor_booktrading.py` — compara `booktrading.stock` del último registro vs `inversion.position` (fuente de verdad IB); reporta desvíos; flag `--fix` genera SQLs correctivos
- ✅ 4 cantidades corregidas manualmente en booktrading + inversion: CRNT (+20 fantasma — fila duplicada 2026-03-23), SKLZ (+12 fantasma), BTI (+1 — doble entrada 2026-03-31), CVS (−2 — compras faltantes)
- ✅ KYN corregido — booktrading mostraba stock=150 pero inversion=0 (posición cerrada)
- ✅ Monitor post-corrección: 1 alerta activa restante — CFRXQ (340 en inversion, sin registros en booktrading)

### v2.3 — 2026-05-02
- ✅ ítem 6 — `backup_diario.bat` v2.2: dumps planos en Drive sin subcarpetas, retención 5 últimos `.sql` por conteo, log fijo `backup_diario.log` (sobreescribe cada corrida), dump local se elimina tras copia a Drive

### v2.2 — 2026-05-02
- ✅ ítem 29 — Órdenes desde TV: `POST /order` en `Class_BrowserBridge.py` conecta con broker vía `_order_callback` → `put_order`; userscript muestra estado (Submitted/PreSubmitted/FILLED) y limpia qty; refresh automático del panel post-orden
- ✅ ítem 29 (extensión) — Selector de cartera en TV panel: botón ≡ en header abre lista de chips con todos los símbolos en posición; clic en chip hace `POST /current` → `_switch_callback` → `_abrir_tradingview(symbol)` → TV navega al símbolo; botón flotante 📊 siempre visible para reabrir el panel
- ✅ ítem 25 — primer intento de conectar cartera con co-work (superado por MCP server Fase 3 — 2026-07-03)
- ✅ ítem 33 — rebuild diaria_performance desde 2025-12-20: `run_balance_booktrading.py` sin residuos; `run_fix_hasi_stock.py` corrigió 19 registros HASI (5 acciones fantasma desde sec=25); rebuild completo ejecutado

### v2.1 — 2026-05-01
- ✅ ítem 36 — Guard anti doble-tap Telegram: `put_order_aprovate_telegram()` verifica `estado == "ejecutada"` antes de enviar al broker; `handle_callback()` llama `edit_message_reply_markup(reply_markup=None)` al primer tap para quitar los botones inmediatamente — dos capas de protección evitan ejecución duplicada de órdenes

### v2.0 — 2026-04-30
- 🐛 4 bugs identificados y corregidos en `Modulos_Comunes.py` + `Modulos_Mysql.py` (pipeline diaria_performance → performa_inversion)
- ✅ `detalle_book`: soporte `fecha_deliste` por símbolo
- ✅ `mark_booktrading_delisted(fecha_deliste=None)` — nuevo parámetro opcional
- ✅ `AppTest/run_balance_booktrading.py` + `run_diag_missing_symbols.py` + `run_solo_performa.py` + `run_rebuild_diaria_cartera.py`

### v1.9 — 2026-04-26
- ✅ Guardián precios yfinance en `write_csv()` — descarta precios que difieren >20x del costo promedio
- ✅ `IPerformance.purgar_desde(account, vehiculo, desde)` — reparación limpia de diaria_performance + performa_inversion
- ✅ Pre-carga `DataHub.info` al arranque para IB offline

### v1.8 — 2026-04-24
- ✅ ítem 26 — Fallback yfinance cuando IB offline
- ✅ ítem 27 — Fix sector preservation en `update_inversion_stock()`

### v1.7 — 2026-04-20
- ✅ floatShares + inst_score v2 + Gate Consenso Telegram

### v1.6 — 2026-04-15
- ✅ Chatbot rediseñado + `_consultar_claude()` Claude Haiku API

### v1.5 — 2026-04-14
- ✅ Race condition stocks + `Agente_ClasificadorETF` Claude Haiku

### v1.4 — 2026-04-13
- ✅ TV toggle + BUY/SELL desde TV + Gmail Fase 0

### v1.3 — 2026-04-13
- ✅ `Class_ServiciosCrypto.py` Earn↔Spot + LTV gráfico

### v1.2 — 2026-04-13
- ✅ Parsers extractos BBVA + Santander (PDF)

### v1.1 — 2026-04-13
- ✅ `BinancePay` + KPIs Finance rediseñados

### v1.0 — 2026-04-05
- ✅ TradingView integrado + Módulo Finanzas Personales
