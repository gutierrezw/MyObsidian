---
tipo: resultado-revision-externa
estado: validado-verificado-en-codigo
fecha: 2026-08-20
---

# Resultado — revisión crítica Opus: Preservation + GainsCapture

Prompt origen: [[prompt-revision-opus-preservation-gainscapture]].
Veredicto de Opus: **no activar PROD**. Los 9 hallazgos bloqueantes/advertencia fueron
**verificados línea por línea contra el código actual** en esta sesión (no se aceptaron
por confianza en el análisis externo — mismo criterio que [[feedback_validar_integracion_por_capa]]).

Todos los hallazgos H1-H4 y H6-H9 coincidieron exactamente con el código leído
(`Class_customer.py:917-981` `maximiza_sell_lotes()`, `Class_AgentManager.py:780-1259`
`Agente_ManagerPreservation`/`_preservation_run_vehiculo`, `Class_DashBot.py:1005-1040`
`_gains_capture_run`). H5 y H10 son consecuencia lógica de H1/H9 y de la comparación
diseño-vs-código ya documentada — no requerían grep adicional.

## Hallazgos validados y cómo se abordan

| # | Hallazgo | Severidad | Verificado en | Abordaje |
|---|----------|-----------|----------------|----------|
| H1 | `maximiza_sell_lotes()` reparte por **conteo de lotes**, no por cantidad — "25%/33%/100%" son etiquetas falsas. Con <4 lotes, "25%" vende 0; "100%" vende TODOS los lotes en ganancia aunque Claude solo vio el ROI de uno | 🔴 Bloqueante | `Class_customer.py:953-978` — confirmado exacto: `cant_lote/c_sell <= threshold`, `cant_acum` suma `stock` (qty íntegra del lote) | Reescribir para operar sobre cantidad real de acciones, no sobre conteo de lotes. Decidir explícitamente si el escenario aplica al lote evaluado o a la posición completa — hoy el código y el prompt de Claude (`Class_DashBot.py:1174`, "salida total del **lote**") se contradicen. Bloquea ítem #53 |
| H2 | `if vender_qty <= 0: continue` sin log — cada vez que H1 da 0, la decisión de Claude se evapora sin rastro. Las "semanas sin incidentes" no prueban prudencia | 🔴 Bloqueante | `Class_DashBot.py:1016-1018` — confirmado exacto, sin log en esa rama | ✅ **Hecho 2026-08-21**: query de calibración corrida contra `symbol_decision_history` — tabla con 1 sola fila total (Preservation/EXIT), cero eventos CLAUDE/ENVIADA de GainsCapture → no hay evidencia de prudencia, coincide con el bug. `Class_DashBot.py` ahora loguea WARNING + inserta fila `CANCELLED` en `symbol_decision_history` cuando `vender_qty<=0` |
| H3 | Ratchet del stop se rompe por el round-trip del estado: `stop_final` se cachea contra `stop_max` (basado en `last-ATR` del ciclo), pero se persiste igual aunque no se haya tocado la orden — el JSON puede quedar por debajo del stop real en IB | 🔴 Bloqueante | `Class_AgentManager.py:1086-1088` (cap) + `:1247-1249` (persistencia incondicional del valor capado) — confirmado exacto | ✅ **Hecho 2026-08-21**: se separó `stop_persistido` de `stop_final`. Solo se persiste el valor capado cuando la orden se modificó en IB con `order_id` confirmado; en DRY-RUN se persiste `stop_final` (simulación); si no hay cambio o el cancel+send no confirma `order_id`, se persiste `stop_anterior`. Pendiente (no hecho): reconciliar `preservation_state.json` contra órdenes vivas de IB al **inicio** de cada ciclo — cambio más grande, a evaluar aparte |
| H4 | Cancel+send sin transacción ni verificación del retorno del cancel — gap sin protección si el send falla tras un cancel exitoso, y una excepción en un símbolo aborta el resto del vehículo (try/except está a nivel de vehículo, no de símbolo) | 🔴 Bloqueante | `Class_AgentManager.py:1116-1119` (cancel+send sin try propio) + `:787-794` (try/except envuelve el `for vehiculo`, no hay uno por símbolo dentro de `_preservation_run_vehiculo`) — confirmado exacto | ✅ **Hecho 2026-08-21**: todo el cuerpo por símbolo dentro de `_preservation_run_vehiculo` quedó envuelto en su propio try/except (una excepción ya no aborta el resto de posiciones del vehículo, solo salta ese símbolo con log). `preservation_cancel_order` ya propaga excepción si falla (patrón existente `doesn't exist`/error genérico) — al estar ahora dentro del try por símbolo, un cancel fallido corta ese símbolo antes de llegar al `send`, sin dejar el estado corrupto (ver H3: `order_id`/`stop_persistido` no se tocan si no hubo confirmación) |
| H5 | STOP (Preservation) + LMT SELL (GainsCapture) pueden coexistir sobre el mismo símbolo sin vínculo OCA ni consulta cruzada de órdenes vivas — con escenario "100%" + Preservation 33%, hasta 133% de las acciones en ganancia comprometidas simultáneamente | 🔴 Bloqueante | Consecuencia directa de H1 (qty no acotada) + estados en dos JSON independientes (`preservation_state.json`, `gains_capture_state.json`) sin lock ni reconciliación — no requiere grep adicional, ya señalado como pregunta abierta en `design-gains-capture.md` | ⛔ **No aplicable todavía (2026-08-21)**: confirmado que depende de H1 — sin qty acotada por lote, el gate cruzado no tiene contra qué comparar `qty_propuesta + qty_comprometida <= position`. Bloqueado hasta reescribir `maximiza_sell_lotes()`, que queda pendiente por falta de datos operativos |
| H6 | Preservation sobre Crypto nunca envía órdenes reales — calcula todo, arma la trama, loguea `[DRY-RUN]` y nunca manda nada | 🟠 Advertencia | `Class_AgentManager.py:1102` — `is_live = not self._preservation_dry_run and vehiculo == "Stock"` confirmado exacto | ✅ **Hecho 2026-08-21**: decisión explícita — sacar `"Crypto"` del loop (`Agente_ManagerPreservation` ahora itera solo `("Stock",)`, con docstring explicando el motivo). Razón del usuario: Crypto es más volátil y Stock todavía no está maduro — no tiene sentido simular protección en Crypto en silencio. Retomar cuando Stock esté sólido |
| H7 | `PRECIO_MINIMO = 50.0` hardcodeado excluye justo el perfil `categoriaActivo='N'` (volátil) que GainsCapture ataca — las posiciones más riesgosas quedan con venta agresiva pero sin protección | 🟠 Advertencia | `Class_AgentManager.py:1030-1035` — confirmado exacto, sin diferenciación por vehículo/símbolo | Ligar el piso a `categoriaActivo` o bajar el umbral para Crypto (el BACKLOG ítem #76 ya documenta `Crypto=$0.001` como validación de precio mínimo distinta — revisar por qué `_preservation_run_vehiculo` no la usa) |
| H8 | Fallback silencioso `sma_base = last` cuando falla SMA20 — en tendencia alcista da un stop más cercano al precio (fail-open), y el ratchet lo deja fijado permanentemente aunque la SMA vuelva | 🟠 Advertencia | `Class_AgentManager.py:1042-1047` — confirmado exacto | Cambiar a fail-closed: si no hay SMA y ya existe stop, no tocar nada y alertar; si no hay SMA y no hay stop, no operar ese ciclo |
| H9 | GainsCapture no tiene interruptor propio pese a aparentarlo — `gains_capture_modo="automatico"` está declarado pero nadie lo lee; el código usa `DataHub.modo_operacion`, compartido con `Agente_ClaudeIA` (Capa 4) | 🟠 Advertencia | `Class_customer.py:1153` (única aparición del atributo en todo el repo, confirmado por grep) vs `Class_DashBot.py:1032` (`DataHub.modo_operacion`) | ✅ **Hecho 2026-08-21**: `DataHub.gains_capture_modo` (global, mismo patrón que `modo_operacion`), leído en `_gains_capture_run` desde `parameters["gains_capture"]["modo"]` (default `SUPERVISADO`), botón propio `📈 GC:{modo}` en `DashMain.py` con `_toggle_gc_modo()` — AUTONOMO deshabilitado en UI para GC hasta resolver H1/H5. `agente_ia.modo` (Capa 4) queda intacto y ya no arrastra a GainsCapture |
| H10 | Divergencias diseño/código: timeout 2h en código vs 30min documentado; `gains_capture_modo`/`gains_capture_config.json` documentados pero ausentes/sin uso; rama Crypto de Preservation documentada y muerta (H6) | 🟡 Cosmético (síntoma) | `Class_DashBot.py:959` (`elapsed > 7200`) vs `design-gains-capture.md` — confirmado en sesión previa | Actualizar `design-preservation.md` y `design-gains-capture.md` para que reflejen el código real, no al revés — la decisión de activar PROD se está leyendo desde el diseño, y el diseño miente en al menos 3 puntos |

## Orden de trabajo (según Opus, adoptado)

1. Query `symbol_decision_history` (`agente='GainsCapture'`, `tag='CLAUDE'`, acción vender) vs órdenes enviadas — antes de tocar código, para calibrar cuánto de las "semanas sin incidentes" es H2.
2. Reescribir `maximiza_sell_lotes()` sobre cantidad, no conteo de lotes (H1).
3. Techo de reglas fijas sobre `vender_qty` en Python, post-Claude (equivalente al piso que ya tiene Preservation).
4. Arreglar ratchet del stop — no persistir capado si no se modificó la orden; reconciliar contra IB (H3).
5. try/except por símbolo + cancel/send con verificación (H4).
6. Gate cruzado STOP vs LMT SELL por símbolo (H5) — depende de 2.
7. Interruptor propio para GainsCapture (H9) + decisión explícita sobre Crypto en Preservation (H6).

**Estado 2026-08-21:** puntos 1, 4, 5, 7 (H2/H3/H4/H6/H9) resueltos. Punto 2 (H1) queda
pendiente explícitamente — el usuario decidió esperar a tener datos operativos reales antes de
reescribir `maximiza_sell_lotes()`. Punto 3 (techo sobre `vender_qty`) y punto 6 (H5, gate
cruzado) dependen de H1 y siguen bloqueados por transitividad.

**No activar `AUTONOMO`/producción para GainsCapture hasta resolver H1 (y por lo tanto H5).**
Con H2 corregido, ahora sí se puede correr una semana en OBSERVACION/SUPERVISADO con el log
registrando los no-ops, para calibrar antes de tocar `maximiza_sell_lotes()` con datos reales
en vez de a ciegas. Preservation (Stock) puede evaluarse en producción de forma independiente —
ya no comparte switch con GainsCapture (H9) y Crypto quedó explícitamente fuera del loop (H6).

## Ítems de BACKLOG afectados

- [[BACKLOG]] #53 (`Agente_GainsCapture`) — bloqueado por H1/H2/H5/H9.
- [[BACKLOG]] #76 (`Agente_ManagerPreservation` Fase 1) — bloqueado por H3/H4/H6/H7/H8.
- Cruza con [[BACKLOG]] #81 (Capa 4 AUTONOMO) por H9 — mismo switch compartido, no subir uno sin el otro.
