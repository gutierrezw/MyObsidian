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

**Confirmado el 2026-11-06 por el lado de los datos — el `fh_count` vacío era un hueco de carga,
no ausencia de cobertura.** Ese día se cerró la cadena de cusips del 13F (`market` 1.036 → 1.165
cusips) y DLR apareció con **5 fondos**; el barrido de los XML en disco muestra **5.959
menciones** de su cusip. O sea el voto no estaba leyendo poca cobertura institucional: estaba
leyendo una tabla incompleta y traduciéndolo a **−1**. Eso invierte el argumento: `None` no es
una concesión al caso raro, es la única traducción honesta de *"todavía no lo sé"* — y los
símbolos nuevos **siempre** entran por ese estado.

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

> **Reversión 2026-11-08 — se midió y no discrimina.** El párrafo de arriba se conserva porque era
> el criterio con el que se abrió la etapa 2. Medido sobre los 12 pares del año en curso, el ratio
> da Spearman **0,0104** contra el retorno del día siguiente: es ruido. Lo que sí hace es describir
> **el día en curso** — Spearman **0,3029** contra el retorno del mismo día, que es el control que
> prueba que el dato se lee bien. Es un descriptor coincidente, no un predictor. Detalle en § 4.4.
>
> Los 4 campos siguen valiendo para persistir (etapa 5): como dato de actividad, no como voto.

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
>
> **Resuelto 2026-11-08 (etapa 3).** `Modulos_Utilitarios.py:468` apunta a `POL28321-USD`.
> Verificado contra Yahoo: `POL-USD` → NONE, `MATIC-USD` → NONE, `POL28321-USD` →
> CRYPTOCURRENCY, precio 0,08987, `circulatingSupply` 10.646.122.496.

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
>
> **Resuelto 2026-11-08 (etapa 3).** `Class_Screener.py:2858` es
> `yf.Ticker(convierte_ticket_crypto(sym)).info`. Los demás usos de `sym` en el método quedaron
> **sin tocar a propósito**: van contra la BD (`market.delete`, `mark_booktrading_delisted`,
> `market.update`, `update_market_cusip`) o contra EDGAR (`_resolve_cusip_from_edgar`), y ahí
> el símbolo tiene que ser el guardado. Validado en seco: de los **38** símbolos en cartera en
> `market`, la función no modifica **ninguno**.

### 4.4 Cripto no lleva `inst_score` ni `fh_buy_ratio` — las cinco fuentes, medidas

La etapa 2 se abrió para elegir el reemplazo. **No existe hoy.** No es una decisión de criterio: es
el resultado de medir todo lo que tiene historia, sobre los 12 pares y la ventana del año en curso.

| Candidato | Fuente | Spearman vs retorno futuro | Veredicto |
|---|---|---|---|
| **taker buy ratio** | klines spot (ya descargadas) | **0,0104** a +1d — y **0,3029** al mismo día | descriptor coincidente, no predictor |
| **momentum de precio** 24/48/72 h y 7 d | klines spot | todos entre **−0,052 y +0,013** a +1/+3/+7 d | los signos **se contradicen entre pares** (ADA −0,2324 contra SOL +0,0747 a 7 d): un voto diría cosas opuestas según el par |
| **funding rate** del perpetuo | `fapi/v1/fundingRate`, 500 registros = 166 días | **−0,044 / −0,043 / −0,040** diario | la única estructura es una monotonía contrarian débil en el retorno a 7 d (terciles +0,793 % / −0,047 % / −0,160 %), a Spearman −0,04 |
| **long/short de cuentas + open interest** | `futures/data/*` | **no medible** | Binance sirve **31 días** (2026-10-08 → 11-07). 372 observaciones en un solo régimen |
| **sentiment de la comunidad** | CoinGecko `/coins/{id}` | **no medible** | es un **valor puntual sin serie**. ADA marcaba 83,33 % positivo mientras caía 23,8 % en 30 días |

Dos límites de cobertura encontrados al medir el funding, que valen para cualquier uso futuro del
dato: **4 de los 12 pares** (CGPT, POL, VTHO, ZIL) tienen perpetuo solo desde el **2026-08-17** (83
días contra 166), y esos mismos 4 cobran funding **cada 4 h (6 cortes diarios)** contra 8 h de los
otros 8 — **el número crudo no es comparable entre pares** sin normalizar por intervalo.

**Por qué el tercil de momentum da todo negativo:** 43–48 % de días positivos en los tres terciles.
Eso es el **régimen** de 2026 en cripto, no una señal.

**Lo que no existe como dato público, no como hueco de un proveedor.** Yahoo devuelve ~80 campos por
par cripto y **ninguna clave de voto**: sin `recommendationKey`, sin `numberOfAnalystOpinions`, sin
`heldPercentInstitutions`, sin `targetMeanPrice`. La misma llamada sobre DLR trae las cuatro
(`strong_buy`, 32 analistas, 1,0296, 224,125). En cripto no hay analistas ni 13F porque no hay
emisor que reporte.

**Decisión: las tres features de Señal entran en `NULL` para cripto y el modelo trabaja sin ellas.**
Es la alternativa que la etapa 2 ya tenía anotada. Con la etapa 1 cerrada el hueco es explícito
—`consenso_suma` NULL, tag `— S/D`— y no un 0 que el modelo leería como una Señal neutral medida.

> **Punto de atención — un campo de Yahoo sí sirve, para la etapa 5.** `volume24HrMarketCapPercent`
> (volumen 24 h sobre capitalización) es exactamente la **Rotación** que el Screener calcula para
> acciones con `floatShares`, la columna que hoy queda vacía en cripto. No es un voto de dirección
> —es actividad, como el taker ratio— pero llena un hueco real sin cálculo propio ni descarga nueva.

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

### 5.1 Revisados el 2026-11-08 — ninguno cambia la decisión

| Proveedor | Qué vende | Veredicto |
|---|---|---|
| **TAAPI.IO** | indicadores técnicos (RSI, MACD, Bollinger) | **descartado sin medir**: son funciones del precio y se calculan de las klines que ya se descargan. Pagar por un cálculo propio |
| **CoinGlass** | long/short de cuentas y open interest **con historia completa** | **el único con el dato correcto.** Sin tier gratuito: Hobbyist 29 USD/mes (intervalo 4 h o mayor — el diario entra) y marcado *personal use only*; comercial arranca en 299 USD/mes |
| **Free Crypto API** | Supertrend + fear & greed | Supertrend es precio otra vez; fear & greed es de mercado, no por símbolo |
| **CoinAPI** | OHLCV multi-exchange, índices, volatilidad | más precio y más volumen, de más fuentes. No es posicionamiento por símbolo |

**Lo que decide el caso de CoinGlass es una medición propia, no el precio:** el **funding rate es el
precio del desequilibrio long/short** — es lo que paga el lado sobrecargado para que el perpetuo
siga pegado al spot. Ya está medido con 166 días gratis de Binance y dio Spearman **−0,044** (§ 4.4).
Si el posicionamiento tuviera señal direccional, su contrapartida económica la mostraría. No es
prueba definitiva —el arbitraje puede comprimir el funding— pero invierte la carga de la prueba: se
pagarían **348 USD/año para mirar el mismo fenómeno desde otro ángulo**, y la medición recién se
podría hacer *después* de pagar.

Y hay vía gratis al mismo dato: **acumular los 30 días rodantes de Binance**. CoinGlass no entrega
nada que Binance no dé hacia adelante — solo adelanta la medición ~6 meses.

**El espacio no es la restricción.** Medido contra `diaria_performance`, la serie diaria más
parecida de `bdinv`: **274 bytes por fila**. Los 12 pares con una fila diaria son **1,2 MB/año**;
50 pares, 5,0 MB/año; 50 pares cada 4 h, 30 MB/año. `bdinv` pesa hoy 571,5 MB y `fund_holdings` sola
son 494 MB de esos. La restricción es **el tiempo**: 6 meses hasta tener muestra.

---

## 6. Plan de ejecución

Cada etapa se valida antes de pasar a la siguiente. **Nada de código hasta que el usuario autorice
etapa por etapa.**

| # | Etapa | Qué se hace | Criterio de validación |
|---|---|---|---|
| **0** | ¿El modelo Buy ve cripto? | **YA VALIDADO** — [[ref-modelos-ia]] lo tiene medido: 16 de 170 filas sin fila en `market`, **5 son pares cripto** (`ADAUSDT`, `ZILUSDT`, `CGPTUSDT`, `ICPUSDT`, `FILUSDT`) | cerrado sin encender la app. Las 3 features de Señal entran en `0.0` para esas filas |
| **1** | Abstención en el Consenso | **CERRADA 2026-11-07** — `Net`/`Flujo` devuelven `None` sin `fh_count`, `Cob` tiene rama `None` (conserva −1 con 1 a 4 fondos), piso `MIN_VOTOS_SENAL = 3`, y un tag `— S/D` persiste `consenso_suma` **NULL** en vez de 0 | **0 diferencias** sobre las 38 filas en cartera. El criterio escrito (DLR de ALERTA a NEUTRO) **quedó obsoleto**: DLR ya tiene 1.490 fondos y es NEUTRO por dato propio — motivo en [[spec-consenso]]. Un par cripto pasa de **ALERTA en 3 de 4 escenarios** a `— S/D` |
| **2** | Qué reemplaza a `inst_score` y `fh_buy_ratio` | **CERRADA 2026-11-08 — nada lo reemplaza.** Las tres features de Señal quedan **NULL** para cripto y el modelo trabaja sin ellas (§ 4.4) | los 3 candidatos con historia medidos sobre los 12 pares y el año en curso: taker ratio **0,0104**, momentum **−0,052 a +0,013**, funding **−0,044**. Los 2 que faltan **no son medibles**: Binance sirve 31 días de long/short y el sentiment de CoinGecko es un valor puntual. Yahoo **no tiene ninguna clave de voto** para cripto |
| **3** | Dos arreglos de una línea | **CERRADA 2026-11-08.** `POL-USD` → `POL28321-USD` en `convierte_ticket_crypto` (`Modulos_Utilitarios.py:468`); `audit_portfolio` enruta el ticker por esa función (`Class_Screener.py:2858`) | ambos verificados: `POL28321-USD` devuelve `circulatingSupply` 10.646.122.496, y en seco la función no modifica ninguno de los 38 símbolos en cartera en `market` (§ 4.1, § 4.3) |
| **4** | Las filas | 12 de cartera primero, el top 50 después. `categoriaActivo` **NULL** para cripto (§0) | **el criterio escrito es inalcanzable** y se reemplaza (2026-11-08): con la etapa 1 cerrada un par cripto persiste `consenso_suma` **NULL**, así que `load_consenso_features` solo lo devuelve por el `OR` de `inst_score`/`fh_buy_ratio`, que la etapa 2 dejó NULL a propósito. Lo verificable es que la fila exista en `market` con `categoriaActivo` NULL y que GainsCapture siga leyendo `inversion`. Y hay un segundo borde: el método toma **una** `account` y el panel le pasa la de Stock, así que cripto (`B0000001`) no entra al dict aunque tenga fila |
| **5** | Lo que ya se tira | persistir supply/marketCap del `info` de Yahoo y el taker-buy de las klines | 11 de 12 con supply, 12 de 12 con ratio, **sin descargas nuevas** |
| **6** | Sentimiento | mapa par→nombre (12 filas) + filtro de homónimos. Ver [[design-sentimiento]] pendiente #7 | el `EXISTS` contra `market` resuelve; "Polkadot" no trae noticias de wellness |
| **7** | Reentrenar Buy | con las features de cripto poblando | capturar métricas **antes**: `modelo_buyv01_metrics.pkl` se sobreescribe en cada entrenamiento |

Las etapas 1 y 2 son el prerequisito: **sin ellas, las etapas 4 en adelante meten ruido con signo
al modelo.** Las dos quedaron cerradas el 2026-11-07 y el 2026-11-08, así que **la etapa 3 ya no
está bloqueada**.

La etapa 1 **no deja a la cripto con Señal, la deja sin Señal**. La etapa 2 confirma que eso es el
estado final, no un tramo intermedio: el hueco **no se llena** porque el dato que lo llenaría no
existe con historia. Queda explícito (`consenso_suma` NULL, tag `— S/D`) en vez de un 0 que el
modelo leería como una Señal neutral medida.

---

## 7. Lo que NO se hace

- **No se toca `categoriaActivo` de cripto en `market`** — queda NULL; la fuente sigue siendo
  `inversion` ([[plan-crypto-gainscapture]] Etapa 0, opción b).
- **No entra BotCrypto** (`B0000002`) — dos dueños sobre la misma posición.
- **No se migran las columnas de equity** — las ~28 sin sentido quedan NULL, no se borran: las usa
  Stock.
- **No se borra una fila que sale del top 50** — se desmarca.
- **No se contrata plan pago** de CoinGecko/CMC: el free alcanza para 50 monedas por día.
- **No se contrata CoinGlass** (29 USD/mes) — es el único proveedor con la historia de long/short,
  pero el funding rate, que es el precio de ese mismo desequilibrio, ya se midió gratis y dio ruido
  (§ 5.1). No se paga por un insumo sin validar, y la vía gratis para validarlo existe.

---

**Ver también:**
- [[spec-consenso]] — los 7 votos y la regla del denominador; el hallazgo de §2.1 vive ahí
- [[ref-modelos-ia]] — las features del modelo Buy, y por qué dejó de consumir sentimiento el 06-11
- [[design-sentimiento]] — pendiente #7: el bloqueador del `EXISTS` contra `market`
- [[plan-crypto-gainscapture]] — por qué GainsCapture ya no es motivo y sí restricción
- [[ref-oportunidades]] — el CSV de entrada del modelo Buy
