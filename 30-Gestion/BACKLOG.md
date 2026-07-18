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
| 67 | **Stock/UI** | Panel trades IB en la app — visualizar los trades ejecutados en IB (últimos 7 días al arranque, solo día actual en adelante). Útil para auditar operaciones que no se registraron en booktrading sin tener que salir de la app. Vista de solo lectura, sin inserción directa para no descontrolar inventario. | Media |
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
| 🧪 53 | **Stock/GainsCapture** | `Agente_GainsCapture` — espíritu especulativo (distinto a Preservation que es defensivo). Opera sobre `categoriaActivo='N'` (volátiles): venta parcial por niveles de ROI (50%/100%/150%), validación Claude de técnicos (RSI_d/RSI_w/EMA) antes de cada nivel. Dos modos: `automatico` (LMT directo a IB + notif Telegram) y `autorizado` (propuesta Telegram /ok /no, timeout 30min → cancela sin ejecutar). **En observación.** | Alta |
| 55 | **IA/Consenso** | Tech Alignment — 7º voto Consenso: RSS TechCrunch+MIT Review → Claude Haiku clasifica temas activos (ai_semiconductors, clean_energy, biotech, etc.) → `voto_tech_alignment(symbol)`. Plan completo en `ConvergIA/`: `Scanner_Tecnologias.py` + `ThemeMapper.py`. Señal para **nuevas oportunidades** según tesis de inversión, no para gestión de cartera activa. | Baja |
| 56 | **Stock/Oportunidades** | Re-entry Scanner — detectar ex-activos con condiciones mejores que la entrada original: ROI histórico > 0 + consenso favorable + precio actual < avgcost histórico. `get_reentry_candidates()` en `PlanInversion` + `Agente_ReentryScanner` @24h → `DataHub.system_alerts`. Diseño en `Doc/reentry_scanner_design.md`. | Media |
| 57 | **IA/Plan** | **"Riesgos del Plan"** — panel UI editable para los parámetros de la misión del agente: meta capital (hoy $1.2M/2030), objetivo ingresos pasivos (hoy 3%), escalado de objetivo (ej: revisar a 4% cuando portafolio ≥ $800K), leverage máximo tolerado, pérdida máxima aceptable. Hoy estos valores están hardcodeados en el prompt. Pasar a `llave_privada.agente_ia.plan` → editable desde UI sin reiniciar app. Segunda sección "Riesgos del Plan": concentración máxima por sector/región, criterios de salida de emergencia. | Media |
| 58 | **Infraestructura/IB** | Migrar `Class_Ibrks.py` legacy a librería oficial de IB — intento anterior revertido por complejidad. Arrancar en rama separada, validar con `AppTest/run_ib_websocket.py` antes de tocar main. La reconexión automática (`_tickle_loop`) debe quedar igual que hoy. | Baja |
| 71 | **FCI/BrowserFCI** | **Primera automatización ejecución FCI** — Santander: traspaso directo entre fondos vía Playwright. BBVA: rescate + suscripción separados. Canal Telegram como aprobador (propuesta → /ok → ejecuta). Candidato destino por `select_fci_rf_candidato` (más deprimido). Todo en `Class_BrowserFCI.py`, reutilizando perfiles Chrome y credenciales existentes. **Pendiente:** guide & capture de ambos bancos + separar señal RV-captura vs RF-acumulación. Spec en `20-Proyecto/spec-fci-browser-ejecucion.md`. | Alta |
| 62 | **Infraestructura/Migración** | Mantenimiento versiones Node.js — definir proceso equivalente a Python: evaluar `nvm-windows` como gestor de versiones, documentar versión mínima requerida por `server-api`, validar PM2 + dependencias npm tras cada upgrade. Ref: [[ref-instalacion]] | Baja |

---

## Historial

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
