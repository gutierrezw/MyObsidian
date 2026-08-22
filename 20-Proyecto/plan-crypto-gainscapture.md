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

**Recomendación Claude:** (c) para GainsCapture — no inventa datos, es una línea, y es
semánticamente cierto. Pero **(a) es el único que sirve también a Preservation**
(`categoriaActivo IN ('I','S')`). La decisión depende de si Crypto va a tener alguna vez activos
"defensivos" → **decisión del usuario, pendiente**.

### Etapa 1 — recalibrar umbrales (sin tocar código, solo `sesion.parameters`)
Independiente del resto y verificable en el panel/Telegram el mismo día. Propuesta a discutir:

| Vehículo | Bloque | min_roi | min_ganancia |
|---|---|---|---|
| Crypto | `gains_oportunidades` | 0.09 | **50 — APLICADO 2026-08-22** (antes 90) |
| Crypto | `gains_capture` | 0.20 | ~60 (hoy 200) |
| BotCrypto | ambos | — | **agregar los bloques** (hoy ausentes → caen a `MinProfit` global) |

**Aplicado 2026-08-22:** `Crypto.gains_oportunidades.min_ganancia` 90 → **50** en `sesion.parameters`.
No requiere reinicio: `load_vehiculo_params` cachea con `PARAMS_TTL = 60` segundos. Con ese piso,
la clase 33% de BTCUSDT ($54) pasa a entrar; la de 25% ($41) sigue afuera.

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

**Pendiente (no ejecutado):** el histórico sigue con el error — Crypto 54 ventas
($1.792,91 → ≈$1.833,01) y BotCrypto 439 ventas (−$91,15 → ≈−$145,23). No se puede corregir con
SQL a ciegas: `booktrading` no guarda `commissionAsset` y las comisiones pagadas en BNB se
corregirían mal. Requiere releer `myTrades` de Binance.

> **Impacto en la Etapa 1:** los umbrales `min_ganancia` se calibran contra ganancias que hasta
> hoy venían mal calculadas en las ventas. Recalibrar después de tener el histórico saneado.

## 4. Próximo paso al retomar

El usuario quiere **revisar primero una venta real que ejecutó**, y con esos datos volver acá.
Al retomar, la pregunta abierta es: ¿arrancamos por la decisión de la Etapa 0, o calibramos la
Etapa 1 con las clases reales de las 12 posiciones antes de decidir?
