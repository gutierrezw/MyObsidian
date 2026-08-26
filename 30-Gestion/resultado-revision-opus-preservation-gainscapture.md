---
tipo: resultado-revision-externa
estado: en-ejecucion
fecha: 2026-08-20
actualizado: 2026-08-26
---

# Resultado — revisión crítica Opus: Preservation + GainsCapture

Prompt origen: [[prompt-revision-opus-preservation-gainscapture]].
Veredicto original de Opus: **no activar PROD**. Los 10 hallazgos fueron verificados línea por
línea contra el código, no aceptados por confianza en el análisis externo — mismo criterio que
[[feedback_validar_integracion_por_capa]].

**Este documento quedó reordenado el 2026-08-26 para que lo pendiente sea lo primero que se lee.**
Motivo: la sección de estado seguía diciendo *"H1 sigue pendiente y sigue siendo el bloqueante de
PROD"* varios días después de que H1 se reencuadrara como no-bug, y esa es justamente la línea
desde la que se decide subir a producción. Lo cerrado no se borró — está abajo, comprimido.

---

## Pendiente

| # | Hallazgo | Severidad | Dónde | Qué falta |
|---|----------|-----------|-------|-----------|
| H5 | STOP (Preservation) + LMT SELL (GainsCapture) pueden coexistir sobre el mismo símbolo sin vínculo OCA ni consulta cruzada de órdenes vivas — hasta 133% de las acciones en ganancia comprometidas a la vez | 🔴 **Bloqueante de #53** | Estados en dos JSON independientes (`preservation_state.json`, `gains_capture_state.json`), sin lock ni reconciliación | Gate cruzado por símbolo: antes de proponer, sumar `qty_propuesta + qty_comprometida` y exigir `<= position`. **Ya no depende de H1** — el tope duro `vender_qty <= position` con ganancia prorrateada existe desde el 2026-08-22, así que hay contra qué comparar |
| H7 | `PRECIO_MINIMO = 50.0` hardcodeado excluye justo el perfil volátil que GainsCapture ataca — las posiciones más riesgosas quedan con venta agresiva pero sin protección | 🟠 Advertencia (#76) | `Class_AgentManager.py:1061` | Decidir el valor real y si se mueve a `gains_config` por vehículo, en vez de un piso único para todos. El `$0.001` de Crypto que documenta el punto (4) de #76 nunca se usó |
| H8 | Fallback silencioso `sma_base = last` cuando falla SMA20 — en tendencia alcista da un stop más cercano al precio (fail-open), y el ratchet lo deja fijado permanentemente aunque la SMA vuelva | 🟠 Advertencia (#76) | `Class_AgentManager.py:1042-1047` | Pasar a fail-closed: si no hay SMA y ya existe stop, no tocar nada y alertar; si no hay SMA y no hay stop, no operar ese ciclo |

### Condición para PROD

- **GainsCapture (#53)** — bloqueado solo por **H5**. El resto de sus hallazgos está cerrado.
- **Preservation (#76), Stock únicamente** — sin bloqueantes. H7 y H8 son advertencias: se puede
  evaluar en producción con ellas abiertas, pero conviene cerrarlas antes de subir el volumen.
  Crypto quedó fuera del loop por H6 y no vuelve hasta que Stock esté sólido.

---

## Cerrados

Se conservan porque explican por qué el código es como es hoy. No reabrir sin motivo nuevo.

| # | Hallazgo | Cierre |
|---|----------|--------|
| H1 | `maximiza_sell_lotes()` reparte por conteo de lotes, no por cantidad de acciones | 🟢 **Reencuadrado 2026-08-21 — no era un bug.** Confirmado con el usuario, dueño del diseño original: "vender 25%/33%/100% de los lotes en ganancia" siempre fue por CONTEO de lotes. Que 25%/33% sean inalcanzables con pocos lotes es matemática, no un error. Fix aplicado en `Class_DashBot.py`: `escenarios_disponibles` según `len(list_gain)` (25% pide ≥4 lotes, 33% pide ≥3) — a Claude solo se le ofrecen los escenarios alcanzables. Validado contra cartera real: 14/26 símbolos con 1-2 lotes reciben solo `["100%"]`. **Ratificado 2026-08-26** |
| H2 | `if vender_qty <= 0: continue` sin log — la decisión de Claude se evaporaba sin rastro, así que "semanas sin incidentes" no probaban nada | ✅ 2026-08-21 — WARNING + fila `CANCELLED` en `symbol_decision_history`. La query de calibración previa confirmó el diagnóstico: la tabla tenía 1 sola fila en total |
| H3 | Ratchet del stop roto por round-trip del estado: se persistía el valor capado aunque no se tocara la orden, y el JSON podía quedar por debajo del stop real en IB | ✅ 2026-08-21 — `stop_persistido` separado de `stop_final`. Solo se persiste el capado si la orden se modificó con `order_id` confirmado; sin confirmación se persiste `stop_anterior`. **Queda fuera de alcance** (no es este hallazgo): reconciliar `preservation_state.json` contra órdenes vivas de IB al inicio de cada ciclo |
| H4 | Cancel+send sin verificación, y una excepción en un símbolo abortaba el resto del vehículo por 12h | ✅ 2026-08-21 — try/except por símbolo dentro de `_preservation_run_vehiculo`; un cancel fallido corta ese símbolo antes del `send` sin dejar estado corrupto |
| H6 | Preservation sobre Crypto nunca enviaba órdenes reales — calculaba todo y quedaba en `[DRY-RUN]` permanente | ✅ 2026-08-21 — decisión explícita: Crypto sale del loop (`for vehiculo in ("Stock",)`). Crypto es más volátil y Stock todavía no está maduro; no tiene sentido simular protección en silencio |
| H9 | GainsCapture no tenía interruptor propio pese a aparentarlo — `gains_capture_modo` declarado y nunca leído | ✅ Diagnóstico correcto. Se implementó un switch propio y **se revirtió el 2026-08-22 por decisión del usuario**: GainsCapture siempre notifica por Telegram y nunca ejecuta en silencio, así que un switch propio no agregaba control — solo permitía silenciarlo y perder oportunidades. Se unificó en `DataHub.modo_operacion` (semáforo único) |
| H10 | Divergencias diseño/código: timeout 2h en código vs 30min documentado, config documentada e inexistente, rama Crypto documentada y muerta | ✅ 2026-08-26 — los tres puntos cayeron: el timeout es `GC_PENDIENTE_TTL = 1800` y ahora se aplica de verdad (barrido previo al loop), la config se unificó vía `DataHub.gains_config()`, y la rama Crypto se eliminó con H6. Los docs se vienen actualizando en cada commit |

### Hallazgos propios, fuera de la revisión Opus

- **`lotesGain()` entregaba los lotes en orden cronológico** — el `sorted` era un no-op porque
  `Nro.Lote` valía siempre 1, así que las clases 25%/33% tomaban los lotes más antiguos en vez de
  los de mejor ROI. Corregido a ROI DESC el 2026-08-22.
- **`lotesGain()` contaba como disponible lo ya vendido** — corregido a `cantidad - sell`.
- **Tope duro `vender_qty <= position`** con ganancia prorrateada, contra la posición real del
  broker (2026-08-22). Es la pieza que deja a H5 implementable.
- **Las propuestas vencidas no se expiraban** si el símbolo dejaba de ser candidato: el chequeo
  vivía dentro del loop, detrás de cinco `continue`. Cinco propuestas quedaron con botones vivos en
  Telegram. Corregido el 2026-08-26 con `_gains_capture_expirar_pendientes()` previo al loop —
  ver [[design-gains-capture]].

---

## Ítems de BACKLOG afectados

- [[BACKLOG]] #53 (`Agente_GainsCapture`) — bloqueado **solo por H5**.
- [[BACKLOG]] #76 (`Agente_ManagerPreservation` Fase 1, Stock) — sin bloqueantes; H7 y H8 abiertos
  como advertencias.
- [[BACKLOG]] #81 (Capa 4 AUTONOMO) — ya no cruza por H9: el switch compartido fue la decisión
  deliberada, no un defecto.
