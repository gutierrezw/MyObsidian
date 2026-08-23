# Agente_GainsCapture — Diseño

**Estado:** Implementado — este documento fue reescrito 2026-08-21 para reflejar el código
real (`Class_DashBot.py:_gains_capture_run`), que divergió del diseño original en el
mecanismo de venta y en el schema de decisión de Claude. Ver [[resultado-revision-opus-preservation-gainscapture]]
H1/H9/H10 para el detalle de la revisión que detectó las divergencias.
**Fecha diseño original:** 2026-06-03 — **Fecha ajuste a código real:** 2026-08-22
(orden ROI DESC de los lotes, umbrales lote-vs-clase, tope contra posición, semáforo único)
**Prioridad:** Alta (ítem 53 backlog) — bloqueado en AUTONOMO hasta resolver H5 (gate cruzado con Preservation)

**Ver también:**
- [[design-preservation.md]] — agente complementario (defensivo, protege ganancias)
- [[analysis-agent-history-table.md]] — tabla unificada para aprender de ambos agentes

---

## Motivación y espíritu

Stocks volátiles como SKLZ, PLUG, CRNT, CHPT pueden hacer movimientos de +50% a +150%
en semanas para luego corregir fuertemente. Un trailing stop solo no alcanza para capturar
esas ganancias — hay que vender parcialmente en el camino antes de la corrección.

**SKLZ:** llegó a $20 (+79% ROI) y volvió a $9. Con el agente: hubiera vendido 25% a $16.77
(nivel 50%) y 50% más a $22.36 (nivel 100%). Ganancia asegurada en vez de ver la caída completa.

**Espíritu especulativo** — a diferencia de `Agente_ManagerPreservation` (defensivo, stocks
estables de dividendo), este agente opera sobre activos de crecimiento/volátiles aprovechando
su movimiento, no protegiéndose de él.

---

## Distinción de responsabilidades

| | `Agente_ManagerPreservation` | `Agente_GainsCapture` |
|---|---|---|
| Espíritu | **Defensivo** — proteger lo ganado | **Especulativo** — capturar upside |
| Activos | `I` (Infravalorado: dividendo, fundamentals sólidos) | `N` (Sin dividendos: growth/volátil) |
| Acción | Coloca STOP trailing | Vende parcialmente por escenario (25%/33%/100% de los LOTES en ganancia) |
| Trigger | ROI >= 10% | `min_roi` filtra **lotes** (calidad individual), `min_ganancia` filtra la **clase** (fricción de la orden) — ver "Umbrales" abajo |
| Claude decide | Dónde poner el stop | Si hay más recorrido o vender ahora |
| Nivel jerárquico | N3 — Decisiones | N3 — Decisiones |

**Nota sobre `categoriaActivo` en `market.table`:**
- `I` = **Infravalorado** — precio bajo vs dividendo/earnings → Preservation protege ganancias
- `S` = **Sobrevalorado** — precio alto vs fundamentals → Preservation también protege (ya está en cartera)
- `N` = **Sin dividendos** — growth/especulativo → GainsCapture vende parcialmente en escalones

**Aclaración:** `S` es "evitar entrada" (no comprar nuevas posiciones), pero si ya está en cartera en ganancia, Preservation aplica igual. La valuación no cambia que necesites proteger lo ganado.

---

## Config en `sesion.parameters` — CÓDIGO REAL

Sección propia `"gains_capture"`, independiente de `"preservation"`, dentro de los
`parameters` de la sesión Stock en MySQL (no en un JSON de `tmp/`):

```json
{
  "gains_capture": {
    "min_roi": 0.20,
    "min_ganancia": 200.0
  }
}
```

- Si la clave `gains_capture` no está presente en `parameters` → agente deshabilitado sin error (SKIP, `_gains_capture_run` corta antes de evaluar símbolos).
- `min_roi`/`min_ganancia` se leen vía `DataHub.gains_config("Stock", "gains_capture")`, con fallback a los globales `MaxRoi`/`MinProfit` de `sesion.userapi` si el bloque existe pero no trae la clave. El bloque en sí sigue siendo obligatorio para que el agente corra.
- **Ya no hay `modo` propio** (2026-08-22): GainsCapture usa `DataHub.modo_operacion`, el semáforo único del sistema — ver "Modos de operación" abajo. Un `"modo"` dejado en el JSON es ignorado.
- No hay `niveles` array ni `revisiones_dia`: el intervalo del agente es fijo, `@wait_rate(1800)` (cada 30 min), no configurable desde BD.
- **Cambio en BD toma efecto sin reiniciar** (2026-08-22): `load_vehiculo_params()` cachea `sesion.parameters` por vehículo con TTL de 60s. Antes quedaba cacheado hasta reiniciar el proceso.

### Umbrales — qué mide cada uno

`min_roi` y `min_ganancia` **no se aplican al mismo objeto** (corregido 2026-08-22, antes
ambos se evaluaban sobre el mejor lote suelto):

| Umbral | Se aplica a | Por qué |
|---|---|---|
| `min_roi` | **cada LOTE** | El ROI es una propiedad intrínseca de madurez del lote. Un lote con 30% de ROI es buen candidato tenga 5 o 500 acciones |
| `min_ganancia` | **la CLASE** (25%/33%/100%) | La clase es la orden que se ejecuta y la que paga comisión/impuesto/spread. Una ganancia de $27 no cubre la fricción aunque el símbolo entero acumule $200 |

**Bloque hermano `gains_oportunidades`** — mismo formato, distinto propósito: barrido
rutinario sobre toda la cartera (`csv_OptionSales_write`, `readCSV_sell`, `get_top_sell`),
con umbrales más bajos (`{"min_roi": 0.09, "min_ganancia": 90}` en Stock y Crypto).
`gains_capture` es el evento explosivo y puntual; `gains_oportunidades` es la rutina. Ambos
se leen con el mismo `DataHub.gains_config(vehiculo, bloque)`.

**Pendiente de calibrar:** con `min_ganancia = 200` medido sobre la CLASE, GainsCapture en
la práctica solo puede proponer ventas "100%" — el techo histórico de una clase 25% en
Stock es $181 y el de una 33% es $147 (`oportunidadesbuysell`, 187 filas SELL). El valor 200
se calibró cuando el piso medía el símbolo entero.

**Diseño original (obsoleto, no implementado así):** un array `niveles` de escalones ROI
(`{"roi": 0.50, "vender_pct": 0.25}`, ...) donde cada nivel se ejecutaba una sola vez y
`vender_pct` se aplicaba en cascada sobre la posición restante de IB. Este mecanismo
**no existe en el código** — fue reemplazado por el reparto por lotes descrito abajo.

---

## Mecanismo real de venta — reparto por LOTES, no por % de posición

`DataHub.maximiza_sell_lotes(list_gain, position, costobase)` reparte por **conteo de
lotes en ganancia**, no por cantidad de acciones ni por % de la posición corriente. Para
cada uno de tres escalones (25%, 33%≈1/3, 100%), un lote se incluye en el total del
escenario solo si `(lotes_incluidos + 1) / total_lotes_en_ganancia <= umbral`:

- **"25%"** solo es matemáticamente alcanzable con **≥4 lotes** en ganancia (1/4 ≤ 0.25).
- **"33%"** solo es alcanzable con **≥3 lotes** (1/3 ≤ 0.336).
- **"100%"** vende todos los lotes en ganancia — el símbolo puede tener 1 solo lote y ya
  cumple "100%".

**Confirmado con el usuario (2026-08-21) que esto es diseño intencional**, no un bug: el
espíritu siempre fue "vender 25%/33%/100% de los LOTES que tienen ganancia" para poder
operar también con lotes completos, no fraccionar acciones dentro de un lote. La
consecuencia — que "25%"/"33%" den 0 acciones con pocos lotes — es matemática, no un
error de cálculo (era el hallazgo H1 de la revisión Opus, reencuadrado tras esta
aclaración).

**Fix aplicado 2026-08-21** (`escenarios_disponibles` en `_gains_capture_run`): antes de
pedirle a Claude que elija escenario, el código calcula cuáles son alcanzables según
`len(list_gain)` del símbolo y solo esos se ofrecen — Claude ya no puede elegir "25%" en
un símbolo con 2 lotes y terminar en una venta de 0 acciones cancelada.

**Ejemplo real (cartera 2026-08-21, vía `AppTest/replay_maximiza_sell_lotes.py`):**
símbolos con 1-2 lotes en ganancia (BABA, UL, ...) solo reciben `["100%"]`; símbolos con
3 lotes (ABEV, RELX, PHYS) reciben `["33%", "100%"]`; símbolos con ≥4 lotes (DLR, PBR, ...)
reciben los tres escenarios.

**Orden de los lotes — ROI DESC (fix 2026-08-22).** `DataHub.lotesGain()` entregaba la
lista en orden cronológico: el contador `Nro.Lote` se reseteaba en cada vuelta, valía
siempre 1, y el `sorted()` posterior era un no-op. Consecuencia: las clases 25%/33% tomaban
los lotes **más antiguos**, no los de mejor rendimiento — lo contrario al objetivo de la
plataforma y a cómo está configurada la venta de lotes en IB. Ahora ordena por `roi` DESC y
`Nro.Lote` es un ranking 1..n. Mismo orden en `lotesGainLost()`, así la ventana fiscal
(Análisis) acumula en la misma secuencia con que se arman las clases; su columna `Clase`
marca hasta qué lote llega cada una, reusando `maximiza_sell_lotes()`.

**Tope contra la posición real (fix 2026-08-22).** Los lotes salen de `booktrading` y pueden
descuadrar contra el broker (splits, dividendos en acciones, ventas parciales). Dos guardas:
`lotesGain()` ahora descuenta lo ya vendido (`cantidad - sell`, antes usaba `cantidad` cruda
y lotes 100% vendidos se contaban enteros — CRNT, MPT, CNH, INTC), y `_gains_capture_run`
recorta `vender_qty` a `position` con la ganancia prorrateada; si tras el recorte cae bajo
`min_ganancia`, cancela y deja `CANCELLED` en `symbol_decision_history`. Esto **no** sustituye
el techo cruzado con Preservation (H5) — acota contra la posición, no contra lo ya
comprometido por el otro agente.

**Parametrizado por vehículo (2026-08-23).** `_gains_capture_run(vehiculo="Stock")` y
`Agente_GainsCapture` iterando `("Stock", "Crypto")` con chequeo de `DataHub.manager_sesion` por
vehículo. La `categoriaActivo` que abre el gate se resuelve una vez antes del loop en
`_gains_capture_categorias()`: Stock desde `market` (`load_symbols`), el resto desde
`inversion.categoriaActivo`, que ya viene en `positions` sin query extra. El loop además filtra
`get_info_symbols_gain()` por `vehiculo`, así los símbolos de otros vehículos no entran.

Un vehículo sin rama en `gains_capture_build_trama_sell()` —ver
`DataHub.gains_capture_vehiculos_trama`, hoy `("Stock", "Crypto")`— deja el candidato en el log como
`candidato en observacion` y corta **antes** de la evaluación Claude y antes de la propuesta por
Telegram. La trama se arma antes del gate de modo justamente para eso: proponer un `/ok_SYMBOL` que
después no puede ejecutarse sería peor que no proponer.

**Trama Crypto (2026-08-23).** LIMIT SELL GTC contra Binance, con `price` y `quantity` pasados por
`quantiza_precio`/`quantiza_qty`; sin `conid` ni `intent`, igual que la rama Crypto de
`preservation_build_trama()`. Los textos de precio usan `DataHub.format_precio()`, que saca los
decimales del mismo `tickSize` — antes iban con `:.2f` y un par barato se leía `$0.00` en la
propuesta de Telegram.

**Saldo en spot y rescate de Earn (2026-08-23).** La cantidad a vender sale de los lotes de
`booktrading` y el tope duro sale de `position`: los dos miden posición contable, ninguno mide el
saldo *free* de spot. Si la cripto está en Earn Flexible (`LDADA`, `LDBTC`…) la orden se rechaza por
saldo aunque la posición exista.

La UI ya resolvía esto en `valida_wallet_spot()` (`pre_orden`, antes de ceder el control), pero es
una función anidada en la ventana Tk y atada a `messagebox` — inalcanzable desde un agente. El mismo
circuito ahora vive en el riel de órdenes, en la rama SELL de `place_OrderCrypto()`, simétrica a la
rama BUY que ya chequeaba USDT. Lo heredan GainsCapture, Preservation Crypto y Telegram de una vez:

1. `crypto_spot_asegura(symbol, qty)` — mide spot, y si falta redime de Earn Flexible el faltante
   (`productId` = `{ASSET}001`, p.ej. `ADA001`) y vuelve a medir.
2. Si aun así falta, la orden **se recorta** a lo disponible (cuantizado por `stepSize`) con aviso
   por Telegram. Si no queda nada, no se manda. La UI, en cambio, avisa con un popup y manda la
   orden igual — está bien ahí porque hay alguien leyendo el cartel.

`crypto_earn_rescate()` pasó a devolver `True`/`False` y a loguear en vez de abrir un `messagebox`:
ahora corre en hilo de agente, donde una ventana modal no corresponde. La UI no pierde el aviso,
porque `valida_wallet_spot()` muestra el suyo al volver a medir.

**Rechazo del broker sin excepción (2026-08-23).** `place_OrderCrypto` devuelve `{}, {}, {}` ante un
rechazo, que vuelve como `{"values": {}, "status": "No Submit"}` — sin excepción. El código marcaba
igual `estado="escalon_pendiente"`, que **no expira** (a diferencia de `pendiente_autorizacion`, que
se vence a los 30 min), así que el símbolo quedaba parkeado para siempre esperando un fill
inexistente. Ahora, si no hay `order_id`, se loguea, se avisa por Telegram y se deja el estado
intacto para reintentar. En el handler `/ok_SYMBOL` la propuesta sigue pendiente y se puede
reintentar. Aplica a los dos vehículos.

**Granularidad de qty y precio por vehículo (2026-08-23).** `vender_qty`, `_pos` y `lmt_price`
dejaron de usar `int()` y `round(…, 2)` en duro: pasan por `DataHub.quantiza_qty()` y
`DataHub.quantiza_precio()`, que truncan a `stepSize`/`tickSize` en Crypto y reproducen exactamente
la cuenta anterior en el resto. La granularidad sale de un único local `gc_vehiculo`, hoy fijo en
`"Stock"` — la Etapa 3 del [[plan-crypto-gainscapture]] lo convierte en parámetro del método. Sin
ese cambio la aritmética de este agente es la misma de siempre.

**Visible en el panel de TradingView (2026-08-22).** `_abrir_tradingview()` arma el bloque
"Venta por clase" llamando a `maximiza_sell_lotes()` con lo que ya está cacheado en
`self.info[symbol]["sell"]` (`list_gain`, `position`, `costobase`) — sin query extra a
`booktrading`, y con el `list_gain` ya capeado por `disponible`. Muestra por clase: lotes,
cantidad, profit, ROI y la posición resultante (`pos avgCost` / `pos position`), que hasta
ahora se calculaba y no se veía en ninguna pantalla. Es una **foto** del momento en que se
abrió la ventana: el panel refresca `last` cada poll pero las clases no se recalculan.

`Agente_ManagerPreservation` sigue corriendo en paralelo sobre el mismo símbolo si tiene
ganancia >= 10%, pero como `categoriaActivo='N'`, su STOP es más amplio (ATR × 2.5). Los
dos agentes conviven — **sin gate cruzado** (ver H5 en [[resultado-revision-opus-preservation-gainscapture]],
sigue bloqueado).

---

## Validación técnica por Claude antes de cada nivel

Antes de colocar la orden LMT, Claude evalúa si hay momentum restante o si el activo
está sobrecomprado. Puede **postergar** la ejecución al próximo ciclo si las condiciones
son favorables para continuar subiendo.

**Señales evaluadas (DataHub tiempo real):**

| Señal | "Esperar más" | "Ejecutar ahora" |
|---|---|---|
| RSI diario | < 65 (momentum intacto) | > 75 (sobrecomprado) |
| RSI semanal | < 65 | > 72 |
| Precio vs EMA50 diaria | Acelerando por encima | Tocando o debajo |
| Precio vs EMA200 diaria | Lejos por encima | Cerca o debajo |

**Respuesta Claude (JSON) — CÓDIGO REAL** (`_gains_capture_claude_eval`):
```json
{"accion": "vender", "escenario": "33%", "razon": "RSI_d=78 sobrecomprado, señal de toma de ganancias"}
```

- `escenario` está restringido en el prompt/enum a los valores de `escenarios_disponibles`
  (solo los alcanzables por conteo de lotes, ver arriba) — no siempre las 3 opciones.
- `accion: "esperar"` → símbolo omitido en este ciclo, re-evalúa en el próximo (sin
  guardar estado especial).
- **Claude falla o no responde → el código NO ejecuta con fallback a reglas.** Es
  fail-closed: `if not claude_result or claude_result.get("accion") != "vender": continue`
  (SKIP, se loguea el motivo). Esto es lo opuesto al diseño original ("ejecuta igual
  usando solo la regla de nivel ROI") — más conservador, decisión de facto no revisada
  explícitamente con el usuario, dejar constancia por si se quiere cambiar.
- El campo `"urgencia"` del diseño original no se usa en el schema real (queda en el
  ejemplo del audit log más abajo por herencia del diseño, pero `_gains_capture_claude_eval`
  no lo pide ni lo recibe).

---

## Tipo de orden — LIMIT fijo

`precio_lmt = last × 0.995` (0.5% bajo precio actual).  
Ejecuta rápido en mercado activo sin regalar precio. No se usan órdenes MKT.

---

## Modos de operación — CÓDIGO REAL

**Vocabulario real** (distinto al del diseño original `automatico`/`autorizado`):
`AUTONOMO` / `SUPERVISADO` / `OBSERVACION`. La lógica es **fail-closed**: solo
`gc_modo == "AUTONOMO"` ejecuta en vivo; cualquier otro valor, incluido uno no
reconocido, pide confirmación por Telegram (fix 2026-08-21, antes era fail-open por un
`in ("OBSERVACION","SUPERVISADO")` que dejaba pasar valores no listados).

**Semáforo único (2026-08-22).** El interruptor propio `DataHub.gains_capture_modo` y su
botón `📈 GC:{modo}` — agregados el 2026-08-21 como respuesta a H9 — **fueron eliminados**.
GainsCapture vuelve a usar `DataHub.modo_operacion` (`Class_DashBot.py:897`), el mismo
semáforo que `Agente_ClaudeIA`, con un solo botón en `DashMain.py`
(`_toggle_modo_operacion`). El motivo del cambio no es el de H9: GainsCapture **siempre
notifica por Telegram**, nunca ejecuta solo sin dejar rastro, así que silenciarlo con un
switch propio solo servía para perder oportunidades. Un `"modo"` que haya quedado en el
bloque `gains_capture` de BD es ignorado.

**Implementación real:**
- `DataHub.modo_operacion` (class var), leída de `sesion.parameters["agente_ia"]["modo"]`,
  default `"OBSERVACION"` (`Class_DashBot.py:154`, `DashMain.py:2261`).
- El botón la actualiza en caliente. `min_roi`/`min_ganancia` también toman efecto en
  caliente desde 2026-08-22 (TTL 60s de `load_vehiculo_params`).
- `tmp/gains_capture_config.json` no existe ni se usa.

**Modo `AUTONOMO`** (sin presencia del usuario, actualmente deshabilitado en UI):
1. Claude valida técnicos → decide `accion: "vender"`
2. LMT enviada a IB directamente
3. Alert Telegram informativo posterior (`DataHub.add_alert(..., telegram=True)`)

**Modo `SUPERVISADO`/`OBSERVACION`/cualquier otro** (pide confirmación):
1. Claude valida técnicos → prepara la propuesta
2. Telegram envía propuesta:
   ```
   📈 GainsCapture — PBR
   ROI lote: 33.0% | Ganancia: $120
   Escenario: 33% | Vender 12 acc LMT $19.50
   RSI_d=78 sobrecomprado — señal de toma de ganancias
   [ ✅ Ejecutar ]  [ ⏸ Diferir ]
   ```
   Botones inline, homologados con la propuesta del Agente IA (2026-08-23). Antes eran los
   comandos de texto `/ok_PBR | /no_PBR`, que **siguen funcionando**: sirven para mensajes
   viejos y como salida si el callback falla.
   `callback_data` = `gc_ok|SYMBOL|pendiente_id` / `gc_no|SYMBOL|pendiente_id`.

   **Por qué el `pendiente_id`:** por símbolo hay una sola propuesta viva a la vez (mientras el
   estado sea `pendiente_autorizacion` el agente saltea ese símbolo), pero el chat de Telegram
   acumula las viejas. Un mensaje de las 10:24 con clase 25% y otro de las 11:24 con clase 100%
   muestran botones idénticos: tocar el viejo ejecutaba **la propuesta vigente**, o sea otra clase
   y otro precio que los que el usuario está leyendo. El id (epoch del momento de la propuesta) se
   guarda en el estado y viaja en el `callback_data`; si no coincide, se rechaza con
   *"esa propuesta ya venció"*. Los comandos de texto no lo mandan y siguen actuando sobre la
   vigente.

   **El envío va por `DataHub.add_alert(msg, telegram=True, markup=...)`, no por
   `exec_modulo_async(send_Telegram(...))`.** `_gains_capture_run()` es sincrónico pero corre
   *dentro* de la corrutina `Agente_GainsCapture`, así que `exec_modulo_async` toma la rama
   `loop.is_running()` → `create_task()`, que vuelve enseguida y queda pendiente; cuando el agente
   termina, `run_until_complete` corta el loop y **la tarea se descarta sin ejecutarse**. Las
   propuestas del 2026-08-23 10:54 quedaron logueadas como "enviada" sin que saliera ningún
   mensaje. `_propuesta_supervisado` (Agente IA) sí puede usar `exec_modulo_async` porque
   `Agente_ClaudeIA` se invoca de forma sincrónica, fuera del loop. `add_alert` además deja
   registro en la tabla de incidencias; el `markup` viaja solo en memoria, así que una alerta
   recuperada por `load_pending_alerts()` tras una caída se reenvía como texto plano.
3. `gains_capture_state[symbol]` → `estado: "pendiente_autorizacion"`, guarda la propuesta completa (`escenario`, `qty`, `lmt_price`, `conid`, `account`, `det`) en `pendiente`.
4. Botón **Ejecutar** o `/ok_<SYMBOL>` → `_gains_capture_aprobar(symbol)` → coloca la orden → flujo normal de fill
5. Botón **Diferir** o `/no_<SYMBOL>` → `_gains_capture_rechazar(symbol)` → `estado` vuelve a `"normal"`, `pendiente: None` — no hay omisión de 6h como decía el diseño original, se re-evalúa en el próximo ciclo (30 min) sin restricción especial
6. **Sin respuesta en 30 minutos → propuesta cancelada** (`elapsed > 1800`, coincide con el diseño)

---

## Flujo de estados por símbolo

Los 4 estados y el timeout de 30 min coinciden exactamente con el diseño original. La
condición de entrada al flujo es distinta (ver mecanismo real arriba):

```
[normal]
  ↓  algún lote con roi_lote >= min_roi   (min_roi filtra LOTES)
  ↓  AND escenario elegido por Claude (dentro de escenarios_disponibles) da vender_qty > 0
  ↓  AND vender_qty acotado a position, con ganancia prorrateada >= min_ganancia
  ↓      (min_ganancia filtra la CLASE — si no llega, CANCELLED en symbol_decision_history)
  ↓  AND Claude dice accion="vender"
Colocar LMT SELL parcial  →  [escalon_pendiente]
  (guarda escalon_order_id en gains_capture_state)

Agente_SyncOrders detecta LMT filled
  → actualizar gains_capture_state: niveles_ejecutados + [nivel]
  → [esperando_reset]

Próximo ciclo Agente_GainsCapture
  → leer qty actual desde IB (posición ya reducida)
  → estado → [normal]
  → evalúa si hay próximo nivel alcanzado
```

| Estado | Qué hace el agente |
|---|---|
| `normal` | Chequea niveles ROI no ejecutados |
| `escalon_pendiente` | No toca nada — espera que SyncOrders detecte el fill |
| `esperando_reset` | Lee posición fresca de IB, vuelve a normal |
| `pendiente_autorizacion` | Espera /ok o /no por Telegram (timeout 30min → cancela) |

---

## Persistencia

### Estado temporal: `gains_capture_state.json`
Archivo propio, independiente de `preservation_state.json`:

```json
{
  "SKLZ": {
    "escalon_order_id": null,
    "estado": "normal",
    "niveles_ejecutados": [0.50],
    "pendiente_autorizacion": null,
    "last_check": "2026-06-03T10:30:00"
  }
}
```

### Audit trail permanente: `order_trader.json_audit_log`

**Cada orden SELL de GainsCapture registra su histórico completo:**

```json
{
  "events": [
    {
      "ts": "2026-08-03T10:30:15.963",
      "tag": "CLAUDE",
      "msg": "RSI_d=78 sobrecomprado, nivel 50% alcanzado",
      "data": {
        "nivel_roi": 0.50,
        "rsi_d": 78.2,
        "rsi_w": 71.1,
        "ema200_rel": "sobre",
        "macd_estado": "alcista",
        "ejecutar": true,
        "urgencia": "alta"
      }
    },
    {
      "ts": "2026-08-03T10:30:16.400",
      "tag": "ENVIADA",
      "msg": "LMT SELL 34 acc @ $16.69 | nivel 50%",
      "data": {
        "qty": 34,
        "lmt_price": 16.69,
        "modo": "automatico",
        "order_id": 123456789
      }
    },
    {
      "ts": "2026-08-03T10:35:00.000",
      "tag": "FILLED",
      "msg": "Venta completada @ $16.67",
      "data": {
        "precio_fill": 16.67,
        "ganancia_realizada": 175.45
      }
    }
  ]
}
```

**Campos guardados en order_trader (GainsCapture):**
- `account`, `symbol`, `conid`
- `clientOrderId` = order_id de IB
- `orderType` = "LMT"
- `side` = "SELL"
- `quantity` = qty_vender
- `price` = lmt_price
- `tif` = "DAY"
- `stampPlace`, `stampSubmit` = datetime.now()
- `json_detalle` = contexto técnico (roi, rsi, ema, nivel_roi, etc)
- `json_audit_log` = eventos históricos [[analysis-agent-history-table.md]]

---

## `json_detalle` en `order_trader` — para aprendizaje futuro

Cada orden guarda el contexto técnico + decisión Claude:

```json
{
  "tipo": "gains_capture",
  "nivel_roi": 0.50,
  "modo": "autorizado",
  "tecnico": {
    "rsi_d": 78.2,
    "rsi_w": 71.1,
    "ema200_rel": "sobre",
    "macd_estado": "alcista"
  },
  "claude": {
    "ejecutar": true,
    "razon": "RSI diario sobrecomprado, señal de toma de ganancias oportuna",
    "urgencia": "alta"
  },
  "orden": {
    "qty": 34,
    "lmt_price": 19.50
  }
}
```

**Valor futuro del dataset:**
- Cruzar contexto técnico vs. precio posterior a la venta → ¿fue oportuno vender?
- Detectar qué condiciones RSI/EMA predicen mejor el peak local
- Calibrar `vender_pct` por nivel según efectividad histórica
- Mejorar el prompt de Claude con casos reales de acierto/error

---

## Plan de implementación

**Estado 2026-08-21: implementado, con el mecanismo de venta y el schema de Claude
distintos a lo planeado abajo — ver secciones "CÓDIGO REAL" arriba.** Se deja el plan
original como referencia histórica de los pasos ejecutados, no como pendiente.

### Paso 1 — BD
- `order_trader.json_detalle` ya existe (creado en Fase 1 de preservation)
- Sin cambios de schema adicionales

### Paso 2 — `Class_customer.py` — DataHub
- Agregar `DataHub.gains_capture_modo: str = "automatico"` (class var)
- Agregar `DataHub.gains_capture_build_trama_sell(vehiculo, account, symbol, conid, last, qty)` → trama LMT SELL
- En inicio leer `gains_capture_config.json` para restaurar el modo guardado

### Paso 3 — `Class_DashBot.py` — nuevo agente
- Agregar `_gains_capture_claude_eval(symbol, nivel_roi, ctx)` → `{"ejecutar": bool, "razon": str, "urgencia": str}`
  - Recibe RSI_d, RSI_w, EMA50/200 desde DataHub.info tiempo real
  - Claude decide si hay momentum restante o sobrecompra
- Agregar `Agente_GainsCapture(self)` con `@wait_rate(intervalo, persist=True)`:
  - Lee posiciones `categoriaActivo='N'` con ROI > primer nivel config
  - Por cada nivel no ejecutado con ROI alcanzado: llama `_gains_capture_claude_eval`
  - Si ejecutar + modo automático: coloca LMT, guarda escalon_order_id, estado → escalon_pendiente
  - Si ejecutar + modo autorizado: envía Telegram propuesta, estado → pendiente_autorizacion
  - Maneja estado `esperando_reset`: lee qty fresca de IB, vuelve a normal
- Registrar en `AgentManager.register_threads()`

### Paso 4 — `Agente_SyncOrders`
- Detectar fill de órdenes con `intent='gains_capture'`
- Al fill: actualizar `gains_capture_state` → `esperando_reset`

### Paso 5 — `Class_SystemStatus.py` — botón en panel Agentes
- En `btn_frame` de `agentes_system()`, agregar botón toggle
- Verde: `"⚡ GainsCapture: Auto"` | Naranja: `"🔐 GainsCapture: Autorizar"`
- Click → toggle `DataHub.gains_capture_modo` + `write_json_tmp("gains_capture_config.json", {"modo": ...})`

### Paso 6 — Telegram handler
- Botones inline **Ejecutar** / **Diferir** (`gc_ok|` / `gc_no|` en `handle_callback`)
- La lógica vive en `_gains_capture_aprobar()` / `_gains_capture_rechazar()`, no en el handler:
  entra por dos puertas (botón y comando) y no puede estar duplicada
- Comandos `/ok_<SYMBOL>` y `/no_<SYMBOL>` para modo autorizado
- `/ok_SKLZ` → coloca la orden pendiente → flujo normal de fill
- `/no_SKLZ` → nivel omitido 6h, re-evalúa en próximo ciclo
- Sin respuesta 30min → propuesta cancelada (no ejecuta)

### Paso 7 — `Class_debugging.py`
- Registrar logger `"GainsCapture"` con `setLevel(WARNING)`

---

## Pendientes / preguntas abiertas

- [x] ¿`Agente_GainsCapture` corre en el mismo hilo que `Agente_ManagerPreservation` o en uno separado? → Separado: `Agente_GainsCapture` vive en `Class_DashBot.py` (trading/mercado), `Agente_ManagerPreservation` en `Class_AgentManager.py` (infraestructura), cada uno con su propio `@wait_rate`.
- [ ] Si ambos agentes operan sobre el mismo símbolo simultáneamente (Preservation pone STOP, GainsCapture pone LMT SELL): ¿hay riesgo de conflicto en IB? **Sigue sin resolver — es H5 en [[resultado-revision-opus-preservation-gainscapture]], bloqueado por depender de que `maximiza_sell_lotes()` acote qty contra la posición comprometida, cosa que hoy no hace de forma verificable entre los dos agentes.**
- [x] Definir intervalo del agente → `@wait_rate(1800)` fijo (30 min), no `revisiones_dia` configurable como decía el diseño original.
- [~] Techo de reglas fijas sobre `vender_qty` en Python post-Claude (equivalente al piso que ya tiene Preservation) — **parcial 2026-08-22**: existe el tope `vender_qty <= position` con ganancia prorrateada y el piso `min_ganancia` sobre la clase. Falta el techo relativo (ej: "nunca más del X% de la posición en un ciclo") y el cruce con Preservation (H5).
- [ ] Calibrar `gains_capture.min_ganancia` — con 200 sobre la clase el agente solo puede proponer "100%" (ver "Umbrales" arriba).
- [ ] El fallback fail-closed cuando Claude falla (esta sesión lo documentó como divergencia del diseño original, que era fail-open) — decidir explícitamente con el usuario si se mantiene o se revierte al comportamiento documentado originalmente.
