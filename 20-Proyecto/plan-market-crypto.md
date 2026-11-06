# Plan — Cripto en la tabla `market`

**Estado:** análisis cerrado, **nada implementado**. Validado contra código y BD el 2026-11-06.
**Qué se decide:** si `market` deja de ser una tabla de equities y pasa a cubrir también cripto
(las ~50 primeras del ranking + las de cartera), y qué hay que arreglar *antes* para que eso sume.
**Fuera de alcance:** FCI. Y BotCrypto (`B0000002`), por la misma razón del
[[plan-crypto-gainscapture]]: superponerle lógica sería dos dueños sobre la misma posición.

Relacionado: [[spec-consenso]] · [[ref-modelos-ia]] · [[design-sentimiento]] ·
[[plan-crypto-gainscapture]] · [[ref-oportunidades]]

---

## 0. Corrección — GainsCapture no es el motivo (y es una restricción)

El argumento que circulaba era *"`market` sin símbolos Crypto deja `categoriaActivo` NULL y eso
bloquea GainsCapture"*. **Ya no vale:** [[plan-crypto-gainscapture]] tiene las 5 etapas **cerradas
el 2026-08-23**, y la Etapa 0 se cerró el 22-08 con la **opción (b)** (`2868fe8`) —
`categoriaActivo` de Crypto vive en **`inversion`**, la escribe `update_inversion()` en `'N'`, y
`_gains_capture_categorias(vehiculo, positions)` la resuelve desde `positions` para todo lo que no
es Stock. Cobertura 25/25. Crypto emite órdenes LIMIT reales en Binance desde esa fecha.

**Se invierte el signo:** hoy hay **un solo escritor** de la categoría por vehículo. Filas de cripto
en `market` crean **un segundo valor posible** para el mismo símbolo, que es exactamente la objeción
con la que se descartó la opción (a) en agosto.

> **Punto de atención — categoría duplicada.** Si se cargan filas de cripto en `market`, hay que
> decidir explícitamente quién manda. La opción barata es dejar `market.categoriaActivo` **NULL
> para cripto** y que `_gains_capture_categorias` siga leyendo `inversion`: `market` aporta Señal y
> datos de mercado, no categoría. Sin esa decisión escrita, el riesgo es silencioso.

El índice de `MEMORY.md` todavía dice *"PAUSADO 2026-08-22, 5 bloqueadores"* — **desactualizado**.

---

## 1. El motivo real — la Señal como feature del modelo Buy

Es lo que el usuario planteó y es lo único que no tiene otro camino.

```
market.consenso_suma / inst_score / fh_buy_ratio
  → MarketScreen.load_consenso_features(account)      Modulos_Mysql.py:3212
      → ModeloOportunidadesBuy.COLS_CONSENSO          Class_IA_modelos.py:552
          → scalar_features del modelo                Class_IA_modelos.py:710
```

`load_consenso_features` filtra `WHERE account = %s AND consenso_suma IS NOT NULL` —
**sin filtro de `encartera`**, a propósito: las oportunidades de compra incluyen símbolos fuera de
cartera.

Y el CSV de entrada **es multi-vehículo**: `csv_OptionBuy_write` (`Class_customer.py:697`) itera
`DataHub.rebalanceo.items()` y para todo lo que no es Stock toma `datos["ranking"]`, escribiendo
`vehiculo` y `account` por fila. `evaluar_oportunidades_buy_con_IA` lo lee completo
(`Class_DashBot.py:3228`) y llama `predecir_oportunidades(df_buy, self.account, …)` (`:3979`).

**Consecuencia medible:** `market` tiene **1361 filas, todas de `U4214563`**. Cero de cuentas
cripto. Para cada fila cripto que llegue al modelo Buy, las tres features de consenso entran en
**0.0** vía el fallback del propio método.

**Y no es hipotético — ya está medido.** [[ref-modelos-ia]] § "Features del modelo Buy" lo tiene
contado sobre la muestra real:

| Dato | Cobertura |
|---|---|
| `consenso_suma` | 154 / 170 (**91%**), rango real −1 a 6 |
| `inst_score` | 150 / 170 |
| **Sin fila en `market`** | **16** — 5 pares cripto (`ADAUSDT`, `ZILUSDT`, `CGPTUSDT`, `ICPUSDT`, `FILUSDT`) y 3 ETF (`TLT`, `VGSH`, `BIL`) |

Esos 5 pares son la prueba de que el CSV ya trae cripto y de que sus features valen 0.0. **La etapa
0 del plan queda validada sin encender la app.**

Es **el mismo defecto que se corrigió esta mañana** en `efe52a3`, donde las 4 features de
sentimiento del modelo Buy valían 0. Cargar filas sin arreglar lo que sigue lo repite.

> **Punto de atención — `0.0` es un valor legítimo.** Hay símbolos con `consenso_suma = 0`, así que
> el modelo **no puede distinguir *"Señal neutra"* de *"sin Señal"***. Es el mismo borde que
> `sentiment_patron` en [[design-sentimiento]]. Cargar las filas lo cierra para cripto; los 3 ETF
> quedan abiertos (`audit_portfolio` **borra los ETF de `market`** por diseño).

---

## 2. El prerequisito — el Consenso está sesgado hacia abajo, y no solo para cripto

### 2.1 La spec dice una cosa y el código hace otra

| Voto | [[spec-consenso]] dice | `AppValuations/consenso_score.py` hace | Efecto sin 13F |
|---|---|---|---|
| **Net** (`:225`) | `None si sin datos` | `:95` → `if not fh_count: return 0` | entra al denominador como **0** |
| **Flujo** (`:236`) | `None si sin fh_count` | `:182` → `if not fh_count: return 0` | entra al denominador como **0** |
| **Cobertura** (`:252`) | no declara abstención | `:148` → `< 5` → **−1**, sin rama `None` | entra como **−1** |
| Opt | `None si sin opciones` | `:108` → `return None` | abstiene ✓ |
| Ana | `None si sin datos` | `:122` → `return None` | abstiene ✓ |
| Val | `None si X/T` | `:135` → `N` → **0** | entra como **0** |

`Class_Screener.py:2019-2030` arma los 8 votos y filtra dejando fuera del denominador
**solo los `None`**. `senal_consenso` (`:159`) **no exige un mínimo de votos**.

### 2.2 Ya está afectando a Stock en cartera, hoy

Medido el 2026-11-06 sobre `market` / `U4214563`: **295 de 1361 símbolos sin `fh_count`**, 12 más
con 1 a 4 fondos. De los que están en cartera:

| Símbolo | `fh_count` | `consenso_suma` | `consenso_tag` | `categoriaActivo` |
|---|---|---|---|---|
| DLR | NULL | −1 | ↘ ALERTA | S |
| SMR | NULL | −1 | ↘ ALERTA | N |
| PHYS | NULL | −1 | ↘ ALERTA | N |
| MPT | NULL | 0 | → NEUTRO | I |
| LUNR | NULL | 0 | → NEUTRO | N |

DLR es Digital Realty Trust, un REIT grande con cobertura real de analistas. Su ALERTA sale de
`Ana +1, Val −1, Cob −1, Net 0, Flujo 0` = **−1 sobre n=5** → pct −0,20. Con la spec aplicada
(Net y Flujo abstienen, Cob abstiene sin datos) queda `+1 −1` = **0 sobre n=2** → **NEUTRO**.
**El tag cambia por el sesgo, no por la tesis.**

De los 295 sin 13F, 289 tienen `consenso_tag` NULL (nunca calculado) y 6 tienen tag: 3 ALERTA,
3 NEUTRO.

### 2.3 Por qué cripto lo amplifica hasta volverlo inútil

Una cripto sin ETF al contado no tiene 13F, ni opciones listadas, ni `recommendationKey`:

| Voto | Valor para cripto sin ETF |
|---|---|
| Net | **0** (en el denominador) |
| Flujo | **0** (en el denominador) |
| Cobertura | **−1** (en el denominador) |
| Valuación | **0** si la categoría es `N` |
| Opt / Ana | `None` — abstienen |
| Sent | ±1 según noticias |

Con sentimiento positivo: suma **0** sobre n=5 → **ALERTA**. Con negativo: **−2/5** → **SALIDA**.
**Una cripto sin ETF no puede leer bien, haga lo que haga el precio.**

Cobertura de ETF al contado: ~4 de los 12 pares (BTC; SOL vía BSOL/VSOL/GSOL; DOT vía 21Shares
TDOT; BNB vía VanEck VBNB). ADA sin resolver — solo productos sobre futuros. FIL nada.
**Los tickers hay que verificarlos contra los filings del emisor**, las fuentes secundarias se
contradicen.

> **Punto de atención.** Mientras `consenso_suma` entre así, el modelo Buy recibiría ruido con
> signo, no una señal. **Esto se arregla antes de cargar una sola fila.**

---

## 3. Lo que ya se descarga y se tira

Nada de lo que sigue cuesta una llamada nueva.

| Dato | Dónde está hoy | Qué pasa con él |
|---|---|---|
| `marketCap`, `volume24Hr`, `totalSupply`, `maxSupply`, `circulatingSupply`, `fiftyTwoWeekHigh/Low` | `get_yfinance(ticket, vehiculo="Crypto")` — **la llamada que el panel ya hace** | llega para **11 de 12** pares; no se persiste |
| `taker_buy_base_volume`, `taker_buy_quote_volume`, `number_of_trades`, `quote_asset_volume` | klines de Binance: manda **12 campos por vela** | el parser (`Class_DataFrame.py:575-625`) conserva **6** y descarta esos 4 |
| Posiciones 13F de los ETF de cripto | `AppValuations/edgar_13f.py:268-270` | `symbol = cusip_map.get(pos["cusip"])` → `if not symbol: continue`. `cusip_map` se arma desde `market`, así que **sin fila no hay cruce** |

`taker_buy_base_volume / volume` es un **ratio compra/venta diario y real** — mejor insumo que
`fh_buy_ratio`, que es trimestral y se publica con 45 días de retraso. Es el candidato natural a
reemplazar `inst_score` / `fh_buy_ratio` en el vector de cripto.

Medido: 24 tickers de ETF/proxy de cripto → **0 en `market`, 0 en `fund_holdings`**
(1.804.601 filas, 1.037 símbolos distintos, último reporte 2026-11-04, 0 sin resolver).

---

## 4. Decisiones ya cerradas

### 4.1 El símbolo canónico es `ADAUSDT`

`convierte_ticket_crypto()` (`Modulos_Utilitarios.py:467`) ya traduce `USDT` → `-USD` y
`get_yfinance` la usa en la rama Crypto (`Class_DataFrame.py:395-397`). 14 consumidores.

Entonces `market.symbol` guarda la forma del par —que cruza con `booktrading`, `inversion` y
`otros_activos` sin traducir— y Yahoo se llama **siempre** a través de la función. No hace falta
columna nueva ni duplicar el símbolo.

> **Punto de atención — el diccionario apunta a un ticker muerto.** La excepción es
> `{"VRLA": "VRLAF", "POL-USD": "MATIC-USD"}`, y `MATIC-USD` también devuelve `None` en Yahoo
> (`quoteType` NONE). El vivo es **`POL28321-USD`**, que no se deriva del par. Es la razón por la
> que POL es el único de los 12 sin datos de supply. **Arreglo: una línea en ese diccionario.**

### 4.2 Las columnas de supply se reusan, no se crean

`market` tiene **86 columnas** y ~28 no significan nada para cripto (PE, EPS, márgenes, ROE,
cashflow, deuda, todo el bloque de dividendos, `lastFiscalYearEnd`, `financialCurrency`).

Reusables tal cual: `shortName`, `lastPrice`, `marketCap`, `volume`, `averageVolume`,
`fiftyTwoWeekHigh/Low`, `beta` (ya existe `Agente_CryptoBeta`), `sector`, `industry` y todo el
bloque `fh_*`.

Propuesta: `sharesOutstanding` ≈ `totalSupply` y `floatShares` ≈ `circulatingSupply` —
**cero columnas nuevas**. Si se decide crearlas (`circulating_supply`, `max_supply`), van en
`float` y con `COMMENT`, según la convención del `CLAUDE.md` § "Tipos de columna".

Análogos que ya existen en otro lado: el dividendo ↔ **APR de Simple Earn**
(`earn_spot_balances()` → `latestAnnualPercentageRate`); `sector`/`industry` ↔ **narrativa**
(L1 / DeFi / storage / AI), donde `ThemeMapper` ya tiene estructura.

### 4.3 Salir del top 50 no borra la fila

Se marca con `favorito` / `encartera`. Una fila que desaparece se lleva la historia de Señal y el
cruce de CUSIP con ella.

> **Punto de atención — `audit_portfolio` borra filas de `market`.**
> `Class_Screener.py:2824`, lo llama `Agente_AuditPortfolio` (`Class_AgentManager.py:248`,
> `@wait_rate(2592000, persist=True, ventana=(0, 6))`) con la cuenta de Stock. Pide
> `yf.Ticker(sym).info` **en crudo** (`:2847`, sin pasar por `convierte_ticket_crypto`) y borra en
> dos casos: ETF/trust, y `if not qt or qt == "NONE"` → deslistado. Con `ADAUSDT` guardado y una
> cuenta cripto, Yahoo devuelve NONE y **borra las filas como si estuvieran deslistadas.**
> **Arreglo: una línea** — enrutar el ticker por la función.

---

## 5. Fuentes externas — qué aportan y qué no

| Fuente | Qué da | Límite free |
|---|---|---|
| **Yahoo** (ya integrado) | precio, supply, marketCap, 52w, noticias por **nombre** (9 de 12) | gratis |
| **Binance** (ya integrado) | klines con taker-buy, 24h, APR de Earn | gratis |
| **CoinGecko** Demo | `/coins/markets` hasta 250 por página: ranking, supply, dominancia | 10.000 créditos/mes, 100 req/min, atribución obligatoria, **sin licencia comercial** |
| **CoinMarketCap** Basic | ranking, supply, y **Fear & Greed / CMC100 / Altcoin Season sin clave** | 15.000 créditos/mes, 50 req/min, uso comercial para **un** producto |

50 monedas = **1 llamada por día = 30 al mes**: entra muchas veces en cualquiera de los dos planes.
Los límites salen de blogs de los propios proveedores y **se contradicen entre sí** (una nota de
CoinGecko dice 30 req/min): **verificar en la página de precios antes de elegir.**

Lo único que no se puede sacar de Yahoo/Binance es **el ranking** — y es justamente el recorte que
define el universo. Fear & Greed es sentimiento **de mercado, no por símbolo**: sería otra feature,
global, no entra en `market_sentiment.symbol`.

---

## 6. Plan de ejecución

Cada etapa se valida antes de pasar a la siguiente. **Nada de código hasta que el usuario autorice
etapa por etapa.**

| # | Etapa | Qué se hace | Criterio de validación |
|---|---|---|---|
| **0** | ¿El modelo Buy ve cripto? | **YA VALIDADO** — [[ref-modelos-ia]] lo tiene medido: 16 de 170 filas sin fila en `market`, **5 son pares cripto** (`ADAUSDT`, `ZILUSDT`, `CGPTUSDT`, `ICPUSDT`, `FILUSDT`) | cerrado sin encender la app. Las 3 features de Señal entran en `0.0` para esas filas |
| **1** | Abstención en el Consenso | alinear `consenso_score.py` con [[spec-consenso]]: `Net`/`Flujo` → `None` sin `fh_count`; `Cob` → rama `None`; mínimo de votos en `senal_consenso` | recalcular y comparar tags **antes/después** sobre los 295 sin 13F. DLR debe pasar de ALERTA a NEUTRO |
| **2** | Qué reemplaza a `inst_score` y `fh_buy_ratio` | decidir: taker-buy ratio de las klines (candidato), o dejarlas en 0 y que el modelo aprenda el hueco | medir el taker-buy ratio de los 12 pares y ver si discrimina |
| **3** | Dos arreglos de una línea | `POL-USD` → `POL28321-USD` en `convierte_ticket_crypto`; `audit_portfolio` enruta por esa función | POL devuelve supply; `audit_portfolio` en seco no borra filas de cripto |
| **4** | Las filas | 12 de cartera primero, el top 50 después. `categoriaActivo` **NULL** para cripto (§0) | `load_consenso_features(B0000001)` devuelve las 12; GainsCapture sigue leyendo `inversion` |
| **5** | Lo que ya se tira | persistir supply/marketCap del `info` de Yahoo y el taker-buy de las klines | 11 de 12 con supply, 12 de 12 con ratio, **sin descargas nuevas** |
| **6** | Sentimiento | mapa par→nombre (12 filas) + filtro de homónimos. Ver [[design-sentimiento]] pendiente #7 | el `EXISTS` contra `market` resuelve; "Polkadot" no trae noticias de wellness |
| **7** | Reentrenar Buy | con las features de cripto poblando | capturar métricas **antes**: `modelo_buyv01_metrics.pkl` se sobreescribe en cada entrenamiento |

Las etapas 1 y 2 son el prerequisito: **sin ellas, las etapas 4 en adelante meten ruido con signo
al modelo.**

---

## 7. Lo que NO se hace

- **No se toca `categoriaActivo` de cripto en `market`** — queda NULL; la fuente sigue siendo
  `inversion` ([[plan-crypto-gainscapture]] Etapa 0, opción b).
- **No entra BotCrypto** (`B0000002`) — dos dueños sobre la misma posición.
- **No se migran las columnas de equity** — las ~28 sin sentido quedan NULL, no se borran: las usa
  Stock.
- **No se borra una fila que sale del top 50** — se desmarca.
- **No se contrata plan pago** de CoinGecko/CMC: el free alcanza para 50 monedas por día.

---

**Ver también:**
- [[spec-consenso]] — los 7 votos y la regla del denominador; el hallazgo de §2.1 vive ahí
- [[ref-modelos-ia]] — las features del modelo Buy, y por qué dejó de consumir sentimiento el 06-11
- [[design-sentimiento]] — pendiente #7: el bloqueador del `EXISTS` contra `market`
- [[plan-crypto-gainscapture]] — por qué GainsCapture ya no es motivo y sí restricción
- [[ref-oportunidades]] — el CSV de entrada del modelo Buy
