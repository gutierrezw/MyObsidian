---
tipo: prompt-revision-externa
estado: pendiente-de-uso
fecha: 2026-08-20
---

# Prompt para sesión Opus independiente — Preservation + GainsCapture (go/no-go PROD)

**Instrucciones de uso:** pegar todo el contenido de abajo en una sesión NUEVA (idealmente Opus),
que no haya visto el resto de esta conversación. El objetivo es una opinión sin sesgo del autor
del código.

---

## Rol

Sos un revisor de riesgo técnico-financiero, externo e independiente, evaluando si dos agentes
de trading automatizado están listos para operar con dinero real en producción, sin supervisión
humana en el loop de decisión (o con supervisión mínima). **No es un ejercicio de validación de
cortesía.** Asumí que el sistema tiene al menos un problema de fondo que el autor no vio — porque
lo escribió, probó y lo dio por bueno más de una vez, y sigue sin animarse a activarlo en PROD. Tu
trabajo es encontrar ESE problema (o esos problemas), no confirmar que "en general está bien
diseñado". Si terminás tu respuesta sin al menos un hallazgo concreto que el autor no haya
anticipado, no lo uses como salida fácil — releé el código una vez más buscando specifically
condiciones de carrera, fallos silenciosos, y escenarios donde la ausencia de una excepción no
significa ausencia de daño.

## Contexto del sistema

AppOO es una plataforma de trading personal (no institucional, 1 usuario) que opera sobre
Interactive Brokers (Stock) y Binance (Crypto). Dos agentes corren en background, cada uno cada
X horas, evaluando posiciones abiertas con ganancia y decidiendo si protegerlas o capturarlas
parcialmente. Ambos usan Claude Haiku como "afinador" de una decisión que las reglas fijas ya
tomaron — Claude nunca decide solo, según diseño.

**Estado real hoy:** ambos llevan semanas "en observación"/"en prueba" sin que el usuario se anime
a dejarlos en modo autónomo sin vigilancia. El usuario pide una segunda opinión — no sabe si el
motivo es (a) el sistema realmente no está listo y hay un problema real sin resolver, (b) es solo
aversión al riesgo razonable pero el sistema ya es sólido, o (c) falta instrumentación/evidencia
para poder confiar, no falta lógica.

---

## Agente 1 — `Agente_ManagerPreservation` (defensivo)

**Propósito:** proteger ganancias ya generadas colocando STOP dinámicos. Corre sobre Stock Y
Crypto, cada 12h por defecto (`@wait_rate(43200)`, aunque el intervalo real de revisión lo decide
`revisiones_dia` en `sesion.parameters.preservation`, ver más abajo).

**Archivo:** `Class_AgentManager.py`, método `Agente_ManagerPreservation` (línea 780) →
`_preservation_run_vehiculo(vehiculo)` (línea 924).

**Lógica (resumen fiel al código):**
```python
for vehiculo in ("Stock", "Crypto"):
    if sesion activa: _preservation_run_vehiculo(vehiculo)

# dentro de _preservation_run_vehiculo:
for positio in positions:  # self.PlanInversion.select_inversion(tipoin=vehiculo, ticket="all")
    roi = unrealizedpnl / costobase
    if roi < roi_minimo or unrealizedpnl < gain_inv_usd:
        # cancela STOP existente si lo había, continue
        continue

    last = DataHub.preservation_get_price(symbol, positio)
    if not last or last <= 0: continue           # sin precio → SKIP

    PRECIO_MINIMO = 50.0
    if last < PRECIO_MINIMO: continue             # floor absoluto, mismo valor para TODOS los símbolos Stock

    atr, atr_error = DataHub.preservation_get_atr(symbol, vehiculo)
    if atr is None: continue

    sma_base, sma_error = DataHub.preservation_get_sma(symbol, vehiculo)
    if sma_base is None: sma_base = last           # fallback silencioso: usa precio actual como base

    max_price = max(state.get("max_price", last), last)
    stop_distance = max(correccion_pct * sma_base, atr_mult * atr)
    stop_calculado = sma_base - stop_distance
    stop_final = max(stop_anterior, stop_calculado)   # nunca puede bajar

    if _claude_key:
        ctx = self._build_preservation_context(...)
        claude_result = self._claude_preservation_eval(ctx, _claude_key)  # timeout 15s, max_tokens=256
        if claude_result:
            stop_final = max(stop_final, claude_result.get("stop_sugerido", 0))  # Claude solo puede subir

    stop_max = round(last - atr, 2)
    if stop_final > stop_max: stop_final = stop_max   # cap: no pegar el stop al precio actual

    qty = DataHub.preservation_calc_qty(self.account, vehiculo, symbol, last, base_limit, proteccion_qty_pct)
    if qty <= 0: continue

    trama = DataHub.preservation_build_trama(vehiculo, account, symbol, conid, stop_final, max_price, qty)
    # → envía orden STP LMT a IB/Binance
```

**Parámetros configurables** (`sesion.parameters.preservation`, por vehículo):
- `roi_minimo` (default 0.10) — ROI mínimo para activar
- `proteccion_base` (default 0.50) — % de la ganancia no realizada usado como `base_limit`
- `correccion_pct` (default 0.08) — corrección permitida desde SMA20
- `atr_mult` (default 2.0) — multiplicador ATR
- `proteccion_qty_pct` (default 0.33) — % de la posición que se protege
- `revisiones_dia` (default 2) — controla el intervalo real vía `preservation_state.json`
- `gainInversion` en `sesion` (no en `preservation`): piso absoluto en USD — Stock=$100, Crypto=$20
  (si `unrealizedpnl < gainInversion`, no protege aunque el ROI% sea alto)

**Gate de precio mínimo:** `PRECIO_MINIMO = 50.0` está **hardcodeado en el código**, aplica a
Stock (no hay diferenciación por símbolo, y es el mismo valor documentado también para Crypto en
otra parte del código — el diseño dice Crypto=$0.001, verificar si ese segundo valor se aplica acá
o es un piso distinto no visto en este fragmento).

**Claude solo afina, nunca bloquea:** si Claude falla (timeout 15s) o no responde JSON válido,
`claude_result` es `None` y el `stop_final` de reglas fijas se usa igual — invariante documentado:
`stop_final >= stop_calculado` siempre.

**Fallback silencioso #1:** si `DataHub.preservation_get_sma()` falla, usa `last` (precio actual)
como base — esto REDUCE la distancia de stop calculada en ese ciclo (SMA20 suele ser más bajo que
el precio en tendencia alcista), sin loguear como error, solo warning.

**Historial reciente (memoria del proyecto, no verificado en este prompt):** hubo un bug de logger
huérfano (`Agente.Preservation` nunca registrado con FileHandler, 13 puntos de log se perdían
silenciosamente) corregido el 2026-08-17, y un doble gate de intervalo (`@wait_rate` del decorador
+ `revisiones_dia` contra `preservation_state.json`) que generó falsas alarmas de "hang". Ambos
ya corregidos, mencionados acá solo como antecedente de que el sistema tuvo fallos de observabilidad
recientes, no de lógica de trading.

---

## Agente 2 — `Agente_GainsCapture` (especulativo)

**Propósito:** capturar ganancias parciales por escalones de ROI en activos volátiles/growth
(`categoriaActivo == 'N'`), vendiendo % de la posición en cada nivel. Corre cada 30 min
(`@wait_rate(1800)`), solo Stock.

**Archivo:** `Class_DashBot.py`, método `_gains_capture_run` (línea 885).

**Lógica (resumen fiel al código):**
```python
for sym_data in symbols_gain:  # DataHub.get_info_symbols_gain()
    if categoriaActivo(symbol) != "N": continue

    lotes_validos = [lote for lote in list_gain
                      if roi_lote >= min_roi (default 0.20) and ganancia_lote >= min_ganancia (default $100)]
    if not lotes_validos: continue

    mejor_lote = max(lotes_validos, key=roi)   # el de mayor ROI, no el más antiguo ni el de mayor $

    estado = self.gains_capture_state.get(symbol, {}).get("estado", "normal")
    if estado == "escalon_pendiente": continue          # espera fill de la venta anterior
    if estado == "pendiente_autorizacion":
        if elapsed > 7200 (2h): estado → normal, propuesta cancelada
        else: continue

    claude_result = self._gains_capture_claude_eval(symbol, roi_ref, ganancia_ref, last, datos_tecnicos, _claude_key)
    if not claude_result or claude_result.get("accion") != "vender":
        continue   # Claude dice "esperar" → NO vende, reintenta próximo ciclo (30min)

    escenario_key = claude_result.get("escenario", "25%")   # Claude también elige CUÁNTO vender, no solo cuándo
    ventas = DataHub.maximiza_sell_lotes(list_gain, position, costobase)
    sell_data = ventas.get(escenario_key, ventas.get(" 25%", {}))
    vender_qty = int(sell_data.get("cantidad sell", 0))

    lmt_price = round(last * 0.995, 2)

    if DataHub.modo_operacion in ("OBSERVACION", "SUPERVISADO"):
        # Telegram: propuesta + /ok_SYMBOL /no_SYMBOL, expira en 2h
        continue
    # modo AUTONOMO:
    trama = DataHub.gains_capture_build_trama_sell("Stock", account, symbol, conid, lmt_price, vender_qty)
    response = DataHub.preservation_send_order("Stock", trama)
    # → orden LMT SELL enviada directo a IB
```

**Parámetros configurables** (`sesion.parameters.gains_capture`):
- `min_roi` (default 0.20) — ROI mínimo del lote para considerar venta
- `min_ganancia` (default $100) — ganancia mínima absoluta del lote

**Hallazgo — a diferencia de lo que dice el diseño (`design-gains-capture.md`), que describe un
modo propio `gains_capture_modo` (`automatico`/`autorizado`, persistido en
`gains_capture_config.json`), el código real usa `DataHub.modo_operacion`, la MISMA variable
global que gobierna el modo de `Agente_ClaudeIA` (Capa 4, OBSERVACION/SUPERVISADO/AUTONOMO),
seteada desde `agente_ia.modo` (`Class_DashBot.py:154`, `DashMain.py:2262`). Es decir:
GainsCapture **no tiene un interruptor independiente** — su nivel de autonomía queda atado al
modo que el usuario configuró para el agente de decisión de Capa 4, aunque son dos sistemas
distintos con distinto perfil de riesgo (uno decide BUY/SELL/HOLD sobre el portfolio completo,
el otro vende parcialmente posiciones puntuales).

**Selección de escenario de venta:** Claude elige el `escenario` (`25%`/`33%`/`100%` de la
posición) dentro del JSON de respuesta — no solo el "sí/no vender", sino cuánto vender. No hay
un tope de reglas fijas sobre el % máximo que Claude puede autorizar vender en un solo ciclo
(a diferencia de Preservation, donde el stop de Claude tiene un piso — acá no se ve un techo
explícito al escenario elegido por Claude en el fragmento revisado).

**Timeout expiración propuesta:** 2 horas (código real) vs. 30 minutos (documentado en
`design-gains-capture.md`) — divergencia entre diseño y código no explicada en el fragmento
disponible.

---

## Riesgo compartido entre ambos agentes

Según el diseño (`design-preservation.md`), Preservation y GainsCapture pueden operar
**simultáneamente sobre el mismo símbolo**: Preservation coloca un STOP trailing, GainsCapture
coloca ventas LMT parciales por escalón. El diseño deja explícitamente como pregunta abierta
(no resuelta en el documento): *"Si ambos agentes operan sobre SKLZ simultáneamente (Preservation
pone STOP, GainsCapture pone LMT SELL): ¿hay riesgo de conflicto en IB? Verificar que IB permita
STOP + LMT abiertos al mismo tiempo sobre el mismo símbolo."* — no se pudo confirmar en este
prompt si esa verificación ya se hizo.

---

## Preguntas concretas para tu revisión

1. **¿Qué falta para activar en PROD con confianza real**, no solo "no tirar error en las últimas
   semanas"? Distinguí explícitamente entre "no hay evidencia de bug" (ausencia de prueba) y
   "hay evidencia de que funciona bien" (prueba positiva) — el sistema hoy tiene mayormente lo
   primero.
2. **Condiciones de carrera / conflicto entre agentes:** ¿el escenario STOP (Preservation) + LMT
   SELL parcial (GainsCapture) simultáneo sobre el mismo símbolo es seguro, o puede generar una
   venta mayor a la posición real, una orden rechazada por IB, o un estado inconsistente entre
   `preservation_state.json` y `gains_capture_state.json`?
3. **Fallbacks silenciosos:** el `sma_base = last` cuando falla `preservation_get_sma()` reduce
   la protección justo cuando falta un dato — ¿es aceptable ese comportamiento por defecto, o
   debería ser fail-closed (no operar) en vez de fail-open (operar con un dato degradado)?
4. **GainsCapture sin interruptor propio:** ¿es un problema real que dependa de
   `DataHub.modo_operacion` compartido con Agente_ClaudeIA, o es aceptable por ahora?
5. **Techo de la decisión de Claude en GainsCapture:** Claude elige tanto el "sí/no" como el
   "cuánto" (`escenario`) sin un techo de reglas fijas visible — ¿es un riesgo real o está
   suficientemente acotado por `maximiza_sell_lotes()` (no visto en este prompt, asumí que lo
   necesitás para responder con seguridad y decilo explícitamente si es así)?
6. **Criterio de "listo para PROD":** dado que llevan semanas en observación sin incidentes,
   ¿cuánto tiempo/cuántos ciclos son estadísticamente suficientes para un sistema que opera pocas
   veces por semana (GainsCapture: 1 revisión/30min pero pocos símbolos califican; Preservation:
   2 revisiones/día)? ¿O el problema no es de tiempo sino de falta de métricas explícitas
   (tasa de acierto, distribución de urgencia de Claude, cuántas veces Claude corrigió vs.
   confirmó la regla fija) que hoy no se están midiendo?
7. Cualquier otro hallazgo que consideres relevante y que no esté cubierto arriba — priorizá
   lo que represente riesgo real de pérdida de capital o de comportamiento inesperado en PROD
   sobre observaciones cosméticas.

## Formato de respuesta pedido

- Lista de hallazgos, cada uno con: qué es el problema, por qué importa (escenario concreto de
  falla), y severidad (bloqueante para PROD / advertencia / cosmético).
- Un veredicto final explícito: ¿activar en PROD ahora, activar con condiciones (cuáles), o no
  activar todavía (por qué, y qué falta)?
- Si identificás que el problema real es falta de instrumentación/evidencia y no de lógica,
  decilo así de claro — es una respuesta tan válida como encontrar un bug de lógica.
