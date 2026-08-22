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
| Crypto | `gains_oportunidades` | 0.09 | 40 (hoy 90) |
| Crypto | `gains_capture` | 0.20 | ~60 (hoy 200) |
| BotCrypto | ambos | — | **agregar los bloques** (hoy ausentes) |

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

## 4. Próximo paso al retomar

El usuario quiere **revisar primero una venta real que ejecutó**, y con esos datos volver acá.
Al retomar, la pregunta abierta es: ¿arrancamos por la decisión de la Etapa 0, o calibramos la
Etapa 1 con las clases reales de las 12 posiciones antes de decidir?
