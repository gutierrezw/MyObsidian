# Plan — Oportunidades Crypto ↔ GainsCapture

**Estado:** análisis cerrado, sin implementar. Pausado 2026-08-22 a la espera de revisar una venta real.
**Alcance:** parámetros y lógica de oportunidades para Crypto + enlace con GainsCapture.
**Fuera de alcance (diferido por el usuario):** Preservation para Crypto — se revisa después.

Relacionado: [[design-gains-capture]] · [[ref-oportunidades]] · [[design-preservation]] ·
[[resultado-revision-opus-preservation-gainscapture]]

---

## 1. Lo que YA funciona para Crypto (verificado en código)

- Pipeline de oportunidades: `oportunidades_sell` → `obtiene_lotes` → caché
  `DataHub.info[symbol]["sell"]` corre para Crypto (12 posiciones vivas, `tipoinv='Crypto'`,
  cuenta `B0000001`).
- `gains_config(vehiculo, bloque)` (`Class_customer.py:551`) ya resuelve umbrales por vehículo.
- `csv_OptionSales_write` (`:572`) escribe `vehiculo` por fila; el gate de calendario trata
  Crypto como 24x7.
- Riel de órdenes completo: `DataHub.QremoteOrder["Crypto"]` (`:231`) + rama Crypto en
  `schedule_order_remote` (`:3004-3080`) → `put_completa_orden` (Binance).
- `DataHub.get_info_symbols_gain()` (`:722`) **ya devuelve los símbolos Crypto mezclados**, cada
  fila con su `vehiculo`. El loop de GainsCapture los recorre hoy — y los descarta en el gate de
  categoría.
- `ClassAgenteIA` ya tiene `self.account_crypto` (`Class_DashBot.py:101`).

## 2. Los 5 bloqueadores

| # | Bloqueador | Evidencia |
|---|---|---|
| 1 | **`market` no tiene símbolos Crypto** → `categoriaActivo` inexistente | `market`: 1360 filas, **1 sola cuenta** (`U4214563`), 0 filas con `%USDT%`, 0 nulos. `inversion` **no tiene columna** `categoriaActivo`. → `categories.get("BTCUSDT")` = `None` → `!= "N"` → SKIP (`Class_DashBot.py:914`) |
| 2 | **Umbrales copiados de Stock** | Crypto `gains_capture` = `min_roi 0.2` / `min_ganancia 200`; `gains_oportunidades` = `0.09` / `90` — idénticos byte a byte a Stock, sobre un libro de **≈$4.26k**. La mejor ganancia no realizada del libro es **+$99** (BTCUSDT). `min_ganancia=200` es inalcanzable; `90` filtra casi todas las clases reales (BTC 25% = +$40, SOL 100% = +$29). **BotCrypto no tiene ninguno de los dos bloques** (cae a globales de Stock) |
| 3 | **Truncamiento numérico** | `vender_qty = int(sell_data["cantidad sell"])` → 0.0121 BTC = **0**; `_pos = int(position)` → el guard `0 < _pos < vender_qty` **falla abierto**; `lmt_price = round(last*0.995, 2)` → ADA $0.2235→0.22, VTHO $0.0004→**0.00**. El mismo `round(…, 2)` está en la rama Crypto de `preservation_build_trama` (`Class_customer.py:1111`) |
| 4 | **Cableado de vehículo** | `Agente_GainsCapture` gatea en `manager_sesion.get("Stock")` (`Class_DashBot.py:476-483`); `ClassAgenteIA.vehiculo = "Stock"` fijo (`:93`); `_gains_capture_run` hardcodea `"Stock"` en `_load_params`, `gains_config` y `select_inversion(tipoin="Stock")` (`:884+`) |
| 5 | **Trama faltante** | `gains_capture_build_trama_sell` devuelve `None` para todo `vehiculo != "Stock"` (`Class_customer.py:1204`). Falta la rama + redondeo por `tickSize`/`stepSize` (ya leídos en `Class_ApiBinnace.py:185`) |

### Libro Crypto al 2026-08-22 (≈$4.26k / 12 posiciones)

BTCUSDT $931 (+99 pnl) · SOLUSDT $796 · ADAUSDT $541 (−470) · ICPUSDT $466 (−537) ·
VETUSDT $282 · BNBUSDT $269 · DOTUSDT $248 · FILUSDT $247 · VTHOUSDT $176 · POLUSDT $166 ·
ZILUSDT $100 · CGPTUSDT $40

## 3. Plan — 5 etapas

### Etapa 0 — decidir el origen de `categoriaActivo` para Crypto
Bloquea todo lo demás, y también Preservation. Tres caminos:

- **(a)** Insertar los 12 símbolos Crypto en `market` con `account='B0000001'` y
  `categoriaActivo='N'`. Consistente con el modelo actual, pero mete crypto en una tabla pensada
  para el pipeline NASDAQ/13F (consenso, `inst_score`, etc. quedarían nulos).
- **(b)** Columna `categoriaActivo` en `inversion` (o tabla propia) como fuente para no-Stock.
- **(c)** Regla implícita: `vehiculo == "Crypto"` ⇒ categoría `'N'` (todo Crypto es
  especulativo), sin fila en BD.

> **Corrección al párrafo original.** Decía que (a) era el único camino que servía también a
> Preservation, por su filtro `categoriaActivo IN ('I','S')`. **Ese filtro no existe en el código**:
> `_preservation_run_vehiculo` itera `select_inversion()` y nunca consulta `categoriaActivo`. El
> filtro está solo en `design-preservation.md`. La decisión no dependía de Preservation.

**CERRADA 2026-08-22 — opción (b), con la coherencia garantizada por diseño.** Commit `2868fe8`.

La columna `categoriaActivo` **ya existía** en `inversion`; lo que no existía era código que la
escribiera — los 25 valores `'N'` de Crypto venían de una carga manual y un símbolo nuevo nacía
en NULL, invisible para el gate de GainsCapture.

La objeción del usuario al diseño inicial fue correcta: *"si cargás categoriaActivo en inversion
debés asegurar que tiene la misma información que market"*. El fallback que se había propuesto
(`market` gana si el símbolo está, `inversion` si no) permitía que las dos tablas contaran
historias distintas sin que nadie se enterara. Se reemplazó por **un solo escritor por símbolo**:

| Vehículo | Fuente de `categoriaActivo` | Cómo se escribe |
|---|---|---|
| Stock | `market` | `sync_market_to_inversion()` copia `market → inversion`, nunca al revés |
| Crypto | `inversion` | `update_inversion()` la fija en `'N'`; Crypto no está en `market` (0 de 25) → no hay dos valores que comparar |
| BBVA.ARS | — | **queda NULL** |

Puntos de sincronización — los dos lugares donde se decide la categoría en `market`:
`dividends_en_market_stock()` (Stock en cartera) y `sync_dividend_status_screener()` (Stock
ex-cartera, vía `Agente_DividendStatusScreener`).

**Casos que quedan en NULL, por decisión explícita:**
- **BBVA.ARS (15 filas, 7 vivas)** — `categoriaActivo` codifica una lectura de valuación por
  dividendos (`I` infravalorada / `S` sobrevalorada). Para un FCI en pesos no hay argumentos para
  valorar si está infra o sobrevalorada, así que **forzar un valor sería inventar señal**. NULL es
  la respuesta honesta.
- **20 Stock ex-cartera sin fila en `market`** — no hay origen del cual copiar. Son posiciones
  cerradas (`position` 0/NULL); la única con posición viva es TLT.

Cobertura tras el cambio: Stock en cartera 40 de 41 con categoría (falta TLT, que no está en
`market`); Stock ex-cartera 14 de 34; Crypto 25 de 25.

### Etapa 1 — recalibrar umbrales (sin tocar código, solo `sesion.parameters`)
Independiente del resto y verificable en el panel/Telegram el mismo día. Propuesta a discutir:

| Vehículo | Bloque | min_roi | min_ganancia |
|---|---|---|---|
| Crypto | `gains_oportunidades` | 0.09 | **50 — APLICADO 2026-08-22** (antes 90) |
| Crypto | `gains_capture` | **0.10 — APLICADO 2026-08-23** (antes 0.20) | **40 — APLICADO 2026-08-23** (antes 200) |
| BotCrypto | ambos | — | ~~agregar los bloques~~ → **DESCARTADO 2026-08-23**, ver abajo |

**BotCrypto sale del plan (2026-08-23).** Es un bot de trading puro: no entra en GainsCapture ni
en Preservation, por diseño y no por pendiente — el bot ya gestiona sus propias salidas y riesgo,
y superponerle un agente sería dos dueños sobre la misma posición. Registrado en
[[spec-botcrypto]] § 1. La afirmación original de esta tabla ("caen a `MinProfit` global") era
cierta en teoría pero irrelevante: `oportunidadesbuysell` no tiene **ninguna** fila de BotCrypto
y `inversion` no tiene **ninguna** con `tipoinv='BotCrypto'`, así que esos umbrales nunca se
consultan. Agregarle los bloques habría sido configuración muerta.

**Aplicado 2026-08-22:** `Crypto.gains_oportunidades.min_ganancia` 90 → **50** en `sesion.parameters`.
No requiere reinicio: `load_vehiculo_params` cachea con `PARAMS_TTL = 60` segundos. Con ese piso,
la clase 33% de BTCUSDT ($54) pasa a entrar; la de 25% ($41) sigue afuera.

**Aplicado 2026-08-23 — `Crypto.gains_capture` a `min_roi 0.10` / `min_ganancia 40`.** Valores de
prueba elegidos por el usuario para **ejercitar el flujo**, no como calibración definitiva: se
reajustan una vez que se vea comportamiento real. Se movieron los dos a la vez porque el gate
exige ambas condiciones (ver corrección al pie). Con ese piso entran las clases de BTCUSDT de
100% ($100 / 15,2%) y 33% ($54), que antes quedaban fuera por ROI.

`UPDATE sesion SET parameters = JSON_SET(...)` sobre el sub-objeto `gains_capture` únicamente —
`parameters` es un blob con credenciales, no se reescribe entero. Las 5 claves quedaron intactas.

> **Todavía inerte.** `_gains_capture_run` pide el bloque como `gains_config("Stock",
> "gains_capture")` (`Class_DashBot.py:894`), así que estos valores **no se leen hasta la Etapa 3**.
> Quedan cargados esperando el cableado de vehículo.

**Umbral del modelo — sin cambios por decisión del usuario (2026-08-22).** `modelo_sellv01`
queda en `umbral_sell = 0.65`. Detectado ese día que ese umbral, y no `min_ganancia`, era lo que
dejaba a BTCUSDT fuera de Telegram: la clase 100% llegó al CSV con $100,22 / ROI 15,2% pero el
modelo no le dio confianza suficiente (el 2026-08-21 la misma clase había sacado 0.6960, apenas
por encima del corte). **El descarte es mudo** — no hay log ni fila, solo se nota por ausencia.
Quedan sin hacer, para cuando moleste: (A) loguear los descartes con su confianza, (B) bajar
`umbral_sell`, (C) reactivar la rama de observación 0.35–0.65 comentada en
`Class_DashBot.py:2980`.

### Etapa 2 — sanear la aritmética (`float`, no `int`)
Quitar los `int()` de `vender_qty` y `_pos`; redondeo de precio dependiente del
vehículo/`tickSize` en vez de `round(…, 2)`. Toca también la rama Crypto ya existente de
`preservation_build_trama` → **beneficia a Preservation antes de revisarlo**. Verificable en
DRY-RUN sin abrir el gate de vehículo.

### Etapa 3 — abrir el cableado de vehículo
`_gains_capture_run(vehiculo)` parametrizado; `Agente_GainsCapture` itera `("Stock", "Crypto")`
como hace `Agente_ManagerPreservation` con su tupla.

**Falta también el lector de la categoría (detectado 2026-08-23).** La Etapa 0 resolvió *dónde
vive* la categoría de Crypto — `inversion.categoriaActivo`, en `'N'` para los 25 símbolos. Pero el
gate de `_gains_capture_run` la busca con `self.Market.load_symbols(self.account)`
(`Class_DashBot.py:906`), que consulta `market`. Los símbolos Crypto entran al loop
—`get_info_symbols_gain()` los devuelve— y mueren ahí con `categ = None`.

Para que Crypto pase el gate, el origen de la categoría tiene que resolverse **una sola vez antes
del loop**, según el vehículo, y que el loop consuma un dict ya resuelto sin saber de dónde salió.
Crypto lo toma de `inversion`. El camino que ya existe no se toca.

> **Cuidado — lección H6 (2026-08-21):** en Preservation, Crypto quedó **perpetuamente en
> DRY-RUN** porque `is_live` dependía de `vehiculo == "Stock"`, y por eso se lo removió de la
> tupla (`Class_AgentManager.py:781-799`). Revisar ese mismo patrón acá **antes** de habilitar.

### Etapa 4 — rama Crypto en `gains_capture_build_trama_sell`
Trama Binance con `stepSize`/`tickSize`. El resto del riel ya existe. Recién acá se emiten
órdenes reales.

### Orden sugerido
`0 → 1 → 2` (las tres sin riesgo de orden real) → validar unos días en DRY-RUN → `3` → `4`.

## 3.bis Bug de comisiones Binance — corregido 2026-08-22

Detectado al revisar una venta real de BNBUSDT (0.105 @ 693.65) que debía dar ~+$13 y quedó
registrada como **−$36,87** en `booktrading`.

**Causa:** el código asumía que Binance siempre cobra la comisión en el **activo base** y hacía
`commission * price`. Eso es correcto en las compras (comisión en POL/VET/ICP), pero en las
**ventas** Binance cobra en el activo *quote* (USDT, que ya es USD) → la comisión quedaba
inflada exactamente `price` veces. Con BNB ($693) 0.0728 USDT se guardó como **50.5208**.

El error es proporcional al precio de la moneda, por eso pasó desapercibido años: en VTHO
($0.009) la comisión quedaba 1.000× *menor*; en BTC ($110k) una venta habría registrado una
comisión ficticia de más de un millón de dólares.

**Fix:** helper único `comision_a_usd(symbol, commission, commission_asset, price, price_bnb)`
en `Class_ApiBinnace.py`, que resuelve por `commissionAsset`: quote USD → tal cual; base →
`× price`; BNB (descuento de fees) → `× precio_BNB`. Reemplaza la lógica que estaba duplicada
en 4 lugares (`Class_ApiBinnace.sync_trades`, `Class_BotCryptoUI._get_insert_fallidos`,
`Class_BotCryptoUI._almacenar_trades_booktrading`, `AppTest/run_binance_import.py`).
Las compras no cambian de valor.

**Corregido en BD:** solo la fila de la venta del 2026-08-22 (`booktrading.id=5113`):
`tarifacomision` 50.5208 → 0.0728332, `gprealizadas` −36.8738 → **+13.5742**,
`mtmgp` −351.179 → **+129.278**. El `hash_id` no incluye la comisión, así que un `sync_trades`
posterior no la duplica.

**Histórico saneado — 2026-08-22.** `AppTest/run_fix_comisiones_crypto.py` releyó `myTrades`
de Binance (`BinanceClient(vehiculo=...)`, paginado por `fromId`), unió por `idtrans` y recalculó
con el mismo `comision_a_usd()`. `importe` no se recalcula: sale por álgebra de la fila existente
(`gpreal_new = gpreal_old + comision_old − comision_new`), así que no hizo falta volver a correr el
emparejador de lotes. Cobertura **100%**: ninguna venta quedó sin su trade en Binance.

| | B0000001 / Crypto | B0000002 / BotCrypto |
|---|---|---|
| Ventas | 54 | 439 |
| Corregidas | 46 (8 ya estaban bien) | 439 |
| Cambian de signo | 0 | 3 (ICPUSDT) |
| Delta neto `gprealizadas` | +$0,24 | +$1,26 |

Backups con los `UPDATE` de reversa en `AppTest/backups/backup_comisiones_<cuenta>_<stamp>.sql`.

> **Corrección de una estimación previa de esta misma sesión.** Antes de correr el script estimé
> el impacto con SQL (`tarifacomision − tarifacomision/preciotrans`, es decir asumiendo que toda
> fila guardaba `commission × price`) y proyecté **−$64 y 26 cambios de signo**. Los `myTrades`
> reales muestran que varias filas (VET, DOGE) no seguían ese patrón, así que la aproximación
> estaba mal en buena parte del histórico. **El cuadro de arriba es el bueno.**

Lo que sí se confirmó es la mecánica: todos los fees son `USDT` (quote); en monedas > $1 la
comisión quedó inflada (ICP 0,2666 → 0,0779 = precio de ICP) y en las < $1 subestimada
(VET 0,00057 → 0,0793). Por eso el neto casi se cancela, pero **fila por fila el error es real** —
las 3 ventas de ICP que pasan de pérdida a ganancia son exactamente las que contaminaban las
etiquetas del modelo de venta (`Batch_Oportunidades_sell-ejecutadas.py:189` etiqueta con
`gprealizadas > 0`, y la línea 43 deriva el ROI de `mtmgp`).

> **Impacto en la Etapa 1:** con un delta neto de $1,50 sobre 485 ventas, `min_ganancia` **no
> necesita recalibrarse** por este motivo. Queda descartado como bloqueador.

**Hallazgo colateral — no es de comisiones.** Al medir el arrastre hacia `diaria_performance`
apareció que esa tabla no refleja las ganancias realizadas de Crypto/BotCrypto: de 181
combinaciones fecha+símbolo con ventas, 82 no tienen fila y 78 tienen `gyp_dia` distinto
(+$186,95 Crypto / −$78,25 BotCrypto sin reflejar). Es **preexistente** y de otro orden de
magnitud. Registrado como **BACKLOG #82**, con alcance limitado a diagnóstico: rehacer esa
historia arrastra extractos, diaria y performance — tres capas encadenadas, no una tabla.

## 4. Próximo paso al retomar

El usuario quiere **revisar primero una venta real que ejecutó**, y con esos datos volver acá.
Al retomar, la pregunta abierta es: ¿arrancamos por la decisión de la Etapa 0, o calibramos la
Etapa 1 con las clases reales de las 12 posiciones antes de decidir?

**Actualización 2026-08-22:** la Etapa 0 quedó **cerrada** (ver arriba).

**Actualización 2026-08-23 — la Etapa 1 queda casi vacía.** Al medir qué efecto real tenían sus
dos ítems restantes:

1. **BotCrypto — descartado**, no pertenece a este plan (ver arriba).
2. **`Crypto.gains_capture` — es preparatorio, no operativo.** `_gains_capture_run` pide el bloque
   hardcodeado como `gains_config("Stock", "gains_capture")` (`Class_DashBot.py:894`), así que el
   bloque de Crypto **no se lee hoy**; recién tendrá efecto tras la Etapa 3. Solo
   `gains_oportunidades` resuelve por vehículo dinámico, y ese ya se bajó a 50.

**Además, el plan tenía mal el diagnóstico del umbral.** Proponía bajar `min_ganancia` 200 → ~60
dejando `min_roi` en 0.20, pero el gate exige **ambas** condiciones: la mejor clase de BTCUSDT dio
ROI **15,2%**, así que queda afuera por ROI, no por monto. Bajar solo `min_ganancia` no habría
cambiado una sola decisión. Si se aplica, van los dos: `min_roi` 0.20 → 0.12 y `min_ganancia`
200 → 60. **Pendiente de autorización del usuario.**

El siguiente paso con efecto real es la **Etapa 2** (aritmética `float`), que además arregla el
`round(…, 2)` que hoy rompe la rama Crypto de Preservation. Sigue en pie instrumentar los tres
`continue` silenciosos de `_gains_capture_run`, que son la razón por la que GainsCapture no deja
rastro en `symbol_decision_history`.
