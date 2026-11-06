# Módulo Sentimiento — Diseño y estado
**AppOO · creado 2026-06 · actualizado 2026-11-06**

---

## Qué es

Scanner de **sentimiento general de noticias** para todos los símbolos en cartera.
No está orientado a temas tecnológicos — cubre toda la cartera sin distinción de sector.

Alimenta el voto **`Sent`** del Consenso y las features de sentimiento del modelo **Sell**.
**El modelo Buy ya no las consume** desde el 2026-11-06 — ver
[[ref-modelos-ia]] § "Features del modelo Buy": el sentimiento es uno de los 8 votos de la Señal,
así que mandar los dos al modelo lo entraba dos veces.

---

## Flujo

```
yf.Search(sym, news_count=5).news  →  titulares del día por símbolo
  → Claude Haiku: clasifica sentimiento (+1 / 0 / −1)
      → BD: market_sentiment  (lectura puntual, una fila por símbolo y corrida)

BD: market_sentiment (historial 7 días, mínimo 2 lecturas)
  → Claude Haiku: detecta patrón de comportamiento
      → BD: market_sentiment_analysis  (patrón diario)
```

**La fuente era `yf.Ticker(sym).news` hasta el 2026-11-06.** Yahoo dejó de servir ese endpoint
y yfinance 1.7.0 lo expone como `YFDataException: *** YAHOO! FINANCE IS CURRENTLY DOWN! ***`.
Medido sobre ABEV en una corrida: `history()` devuelve 5 filas, `news` falla 3 de 3 intentos,
`yf.Search("ABEV", news_count=5)` devuelve 5 noticias. `yf.Search().news` trae el payload plano
antiguo (`link`, `providerPublishTime`, `publisher`, `title`, `uuid`), **sin clave `content`**;
el parser ya hacía `item.get("content") or item`, así que el cambio fue solo de fuente.

---

## Módulos

| Archivo | Responsabilidad |
|---|---|
| `ConvergIA/Scanner_Sentimiento.py` | Descarga noticias + llama Haiku en lotes de 10 |
| `ConvergIA/Interprete_Sentimiento.py` | Lee historial BD + llama Haiku → patrón |
| `ConvergIA/ThemeMapper.py` | `voto_sentimiento()` — combina sentimiento + patrón |

Constantes compartidas: `_MODEL = "claude-haiku-4-5-20251001"`, `_DIAS_HISTORIAL = 7`,
`_BATCH_SIZE = 10`, `_MAX_HEADLINES = 5`.

**Las dos etapas leen la clave de `ClaudeAPIP` pero facturan el uso a `ClaudeAPIS`**
(`track_claude_usage("ClaudeAPIS", ...)`). Sin corregir — el consumo del sentimiento
aparece sumado al de otro vehículo en `tmp/claude_usage_by_key.json`.

---

## Agentes

| Agente | Intervalo | Función |
|---|---|---|
| `Agente_Sentimiento` | 8 horas (3×/día) | Llama `scan_sentimiento()` |
| `Agente_InterpreteSentimiento` | 24 horas (1×/día) | Llama `interpretar_sentimiento()` |

Conectados en `Class_DashBot.py:3383-3384`; entradas estáticas en `AGENTES_SCHEDULE`
(`Modulos_Utilitarios.py:101` y `:107`).

API key: `sesion.vehiculo = 'ClaudeAPIP'`

---

## Lógica del voto `Sent`

| Patrón | Sentimiento | Voto |
|---|---|---|
| `acumulacion` | cualquiera | `+1` |
| `distribucion` | cualquiera | `−1` |
| `neutro` | cualquiera | `0` |
| `inflexion` | `>= 0` | `+1` |
| `inflexion` | `< 0` | `0` (abstención — no penaliza) |
| sin datos | — | `None` (excluido del denominador) |

**Criterio de inflexión:** un activo en transición con noticias negativas no se penaliza
porque puede estar en el inicio de una recuperación. Solo vota positivo si el sentimiento
acompaña el cambio de dirección.

---

## Consumidores — y cómo trata cada uno la ausencia de dato

Son tres caminos y **se degradan distinto**. Eso es lo que decide si una caída de la fuente
se ve o no se ve.

| Consumidor | Ventana de fecha | Qué hace sin dato fresco |
|---|---|---|
| `Class_Screener.py:2029` — voto `Sent` del Consenso | **ninguna** (`MAX(fecha_hora)` sin filtro) | **Vota con el dato viejo** como si fuera de hoy |
| `Class_IA_modelos.py` — 4 features del modelo **Sell** | 7 días | Devuelve vacío y loguea `features ausentes rellenadas con 0.0` |
| `Modulos_Mysql.py:3629 select_preservation_context` | **ninguna** (`ORDER BY fecha DESC LIMIT 1`) | Entrega a Claude el patrón viejo sin fecha |

`load_latest_sentiment()` (`Modulos_Mysql.py:3054`) hace `MAX(fecha_hora) GROUP BY symbol`
**sin filtro de fecha**, mientras `load_sentiment_analysis()` (`:3143`) sí filtra por
`WHERE sa.fecha = CURDATE()`. La combinación es la peor de las dos: con el patrón ausente,
`voto_sentimiento()` cae a la rama `val = sentiment.get(symbol)` y **devuelve el voto viejo con
plena confianza** en vez de abstenerse. No se excluye del denominador: vota.

**Punto de atención — abierto.** Entre el 2026-09-28 y el 2026-11-06 el voto `Sent` del Consenso
se calculó con el sentimiento del **28 de septiembre**, 39 días. Los dos consumidores sin ventana
(`load_latest_sentiment` y `select_preservation_context`) comparten el mismo defecto y **ninguno
está corregido**. El de 7 días es el único que se comporta bien.

**Y la ventana de 7 días tiene su propio borde:** `load_sentiment_features` une el patrón con
`LEFT JOIN market_sentiment_analysis msa ON msa.symbol = ms.symbol AND msa.fecha = CURDATE()`.
Si el agente de 24h no corrió hoy, `sentiment_patron` entra como **0.0 (neutro)** — el modelo no
puede distinguir *"sin señal"* de *"no se calculó"*.

---

## Cobertura

Medido 2026-11-06:

| Métrica | Valor |
|---|---|
| Símbolos en cartera (`encartera='Y'`, `U4214563`) | 41 |
| Símbolos distintos en `market_sentiment` | 42 |
| Símbolos con lectura en los últimos 7 días | **0** |
| Última `fecha_hora` de `market_sentiment` | 2026-09-28 13:47 |
| Última `fecha` de `market_sentiment_analysis` | 2026-10-04 |

Tras el cambio de fuente, `_fetch_headlines` sobre los 8 símbolos de prueba devuelve
**8/8 con 5 titulares cada uno** (antes 0/8).

---

## Stock — la caída de 39 días (2026-09-28 → 2026-11-06)

**No era saldo de API.** `_fetch_headlines` levanta por símbolo, `headlines_map` queda vacío y el
lazo de clasificación es `range(0, 0, 10)` → **cero llamadas a Claude**. Un problema de clave
habría aparecido un paso después, como ERROR de `_classify_batch`.

**Quedó invisible porque cuatro capas se comían el error:**

1. `_fetch_headlines` captura por símbolo en WARNING y no propaga.
2. `scan_sentimiento` escribe solo `if sentiment:` — sin datos no falla, no escribe.
3. El agente devolvía `symbols: 41, with_news: 0, classified: 0` y el decorador lo marcaba
   cumplido → **panel de agentes en verde**.
4. Los consumidores no se quejan: el modelo rellena 0.0 y los dos sin ventana sirven el dato viejo.

**Corregido 2026-11-06** (`37bf825`): fuente a `yf.Search`; el agente levanta incidencia cuando hay
símbolos y ninguno trae noticias (`dedup_key="sentimiento_sin_fuente"`, `renotify=86400`); y
`scan_sentimiento` loguea ese caso a **ERROR** en vez de WARNING.

**Para que surta efecto hay que reiniciar la app.** El agente repuebla en su siguiente turno de 8h
y el patrón vuelve solo en la **segunda** corrida: `interpretar_sentimiento` exige ≥2 lecturas en
7 días (`if len(historial) < 2: continue`).

---

## Crypto — medido, no implementado

Viable, al contrario que FCI, **pero no por ticker**.

| Vía de búsqueda en Yahoo | Cobertura sobre las 12 posiciones activas |
|---|---|
| Ticker (`ADA-USD`, `SOL-USD`…) | **1 de 12** — solo BTC |
| Nombre de la moneda (`Cardano`, `Solana`…) | **9 de 12** |

Sin noticias por ninguna vía: **CGPT, VET, VTHO**. Son demasiado chicas; ninguna fuente gratuita
las cubre.

**Punto de atención — homónimos.** Buscar "Polkadot" devolvió
*"**Polkadot FX** Expands Its Functional Wellness Platform"* — una empresa de wellness, no DOT.
Sin filtro de relevancia el sentimiento de DOT mide noticias de otra industria.

### Tres cosas faltan

1. **Mapa par→nombre**, 12 filas. No es derivable del ticker.
2. **Filtro de relevancia** en el prompt de Haiku, por los homónimos.
3. **El bloqueador estructural:** `load_sentiment_features` y `load_latest_sentiment` exigen
   `EXISTS (SELECT 1 FROM market m WHERE m.symbol = ms.symbol AND m.account = %s AND m.encartera = 'Y')`
   y **`market` tiene 0 filas de cuentas Crypto** (solo `U4214563`, 1361 filas). El sentimiento se
   escribiría en `market_sentiment` sin problema, pero ningún consumidor lo leería.

El punto 3 **no se resuelve insertando 12 filas**: `market` está armada para equities — 13F,
consenso, fundamentals, `categoriaActivo`. Abrir `market` a cripto arrastra esa decisión completa,
no solo el sentimiento, y tiene su propio plan: **[[plan-market-crypto]]**, donde el sentimiento es
la etapa 6 y depende de dos arreglos previos del Consenso.

**Corregido el 2026-11-06 — este párrafo citaba mal su antecedente.** Decía que era *"el mismo
bloqueador que dejó pausado [[plan-crypto-gainscapture]] el 2026-08-22"*. Ese plan **no está
pausado**: sus 5 etapas se cerraron el 23-08 y su Etapa 0 se resolvió **sin** `market` (la categoría
de Crypto vive en `inversion`). El bloqueador del sentimiento es real, pero es propio.

### Fuente: por qué Yahoo y no CoinMarketCap

| Fuente | Noticias por moneda | Costo |
|---|---|---|
| Yahoo (`yf.Search` por nombre) | 9 de 12, probado | gratis, ya integrado |
| CoinGecko | requiere plan **Analyst** | pago (confirmado en su propia guía) |
| CoinMarketCap | free Basic son 15.000 créditos/mes; **no está confirmado** que el endpoint de noticias esté en ese plan | a verificar |

Lo que CMC sí da gratis y sin clave es **Fear & Greed**, CMC100 y Altcoin Season — pero eso es
sentimiento **de mercado, no por símbolo**: no entra en `market_sentiment.symbol`, sería otra
feature, global. Candidato aparte, no reemplazo.

---

## Pendientes

| # | Qué | Estado |
|---|---|---|
| 1 | Ventana de fecha en `load_latest_sentiment` — el voto `Sent` vota viejo | abierto |
| 2 | Ventana de fecha en `select_preservation_context` — patrón viejo sin fecha a Claude | abierto |
| 3 | `msa.fecha = CURDATE()` colapsa "no calculado" a 0.0 en las features | abierto — **solo Sell**, Buy dejó de usarlas el 06-11 |
| 4 | Las dos etapas facturan a `ClaudeAPIS` leyendo la clave de `ClaudeAPIP` | abierto |
| 5 | Logger "Sentimiento" registrado dos veces (`Class_debugging.py:400` y `:440`) | abierto |
| 6 | `enriquecer_con_sentimiento` duplicado (`Class_IA_modelos.py:115` y `:610`) | **cerrado 06-11** — la copia de Buy pasó a `enriquecer_con_features(df, features, columnas)`, genérico; Sell conserva la suya |
| 7 | Crypto: mapa par→nombre + filtro de homónimos + decisión sobre `market` | no iniciado |

---

**Ver también:**
- [[ref-consenso]] — los votos del Consenso, donde entra `Sent`
- [[design-preservation]] — consume el patrón vía `select_preservation_context`
- [[spec-convergia]] — el paquete `ConvergIA/` completo
- [[ref-modelos-ia]] — por qué el modelo Buy dejó de consumir las 4 features de sentimiento
- [[plan-market-crypto]] — el plan completo para abrir `market` a cripto; el sentimiento es su etapa 6
