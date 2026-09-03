---
name: ref-oportunidades
description: Sistema BUY/SELL — reglas, señales, umbrales, score híbrido TOP10 y ciclo IA. Referencia principal.
metadata:
  tipo: ref
  modulo: oportunidades
  version: 2.3
  fecha: 2026-09-03
  status: activo
---

# Sistema de Oportunidades BUY / SELL

**Ver también:** [[design-autonomo]] — con qué mandato vende este módulo dentro del modo autónomo

> Documento unificado. Reemplaza: `ref-regla-buy`, `ref-modelo-buy`, `ref-modelo-sell`.
> Referencia técnica de clases: ver [[ref-modelos-ia]]
> Gate Telegram (qué tags permiten o bloquean el envío): ver [[ref-consenso]]

---

## 1. Flujo completo

```
CSV datosIA_buy          CSV datosIA_sell
       │                        │
  readCSV_buy()           readCSV_sell()
       │                        │
evaluar_oportunidades_buy  evaluar_oportunidades_sell
       │                        │
  ¿modelo cargado?         ¿modelo cargado?
   NO → origen="system"     NO → origen="system"
   SÍ → predice confianza   SÍ → predice confianza
         filtra por umbral         filtra por umbral
         origen="ia"               origen="ia"
       │                        │
oportunity_handler_buy   oportunity_handler_sell
       │                        │
  buy_enviados{}          sell_enviados{}
       │                        │
  get_top_buy(5)          get_top_sell(5)
       │                        │
       └────── TOP 10 Telegram ─┘
               5 BUY + 5 SELL
```

### Umbral SELL — se mide sobre la CLASE, no sobre el símbolo (2026-08-22)

Una fila SELL no es "el símbolo": `csv_OptionSales_write` escribe **una fila por clase**
(25% / 33% / 100% de los lotes en ganancia, ver `maximiza_sell_lotes()` en
[[design-gains-capture]]). El piso de ganancia se validaba sobre la suma de todos los lotes
del símbolo y después las clases salían sin volver a validarse: un símbolo con $200 generaba
clases de $27 y $46 que igual llegaban al Telegram.

| Umbral | Se aplica a | Dónde |
|---|---|---|
| `min_roi` | cada **lote** (calidad intrínseca) | `lotesGain()` / filtro previo |
| `min_ganancia` | cada **clase** (es la orden que paga comisión) | `csv_OptionSales_write`, `readCSV_sell`, `get_top_sell` |

- Fuente única: `DataHub.gains_config(vehiculo, "gains_oportunidades")` — umbrales **por
  vehículo** desde `sesion.parameters`, con fallback a los globales `MinProfit`/`MaxRoi`.
  Hoy: Stock y Crypto en `{"min_roi": 0.09, "min_ganancia": 90}`; BotCrypto sin bloque (usa
  los globales).
- Sobre el símbolo queda solo un pre-filtro barato: si el símbolo entero no llega al piso,
  ninguna de sus clases puede llegar.
- `readCSV_sell` dejó de hacer **fail-open** — devolvía el DataFrame completo sin filtrar
  cuando ninguna fila calificaba. Ahora devuelve vacío.
- ⚠️ `Agente_ManagerSell` llama `readCSV_sell(filtrar=False)` — bypass explícito, hoy inocuo
  porque el CSV ya viene filtrado en origen.
- ⚠️ El `factor` de divisa no se aplica al comparar contra el piso: en BBVA.ARS / SANT.ARS el
  profit está en pesos y pasa cualquier umbral en dólares. Pendiente.

### Gate de calendario — no se notifica un vehículo que hoy no opera (2026-08-22)

`schedule_oportunidades` corre **cada 15s por vehículo, sin gate de sesión**: regenera los CSV
con precios de BD/yfinance aunque IB esté caído (el fallback `schedule_ib_offline_sync` sigue
refrescando `last`). En **modo etiquetado** eso llega directo a Telegram: se saltan el control
de frecuencia (`Agente_message_Manager_Buy`) y el filtro del menú. Resultado: oportunidades
Stock un sábado, sobre precios congelados del viernes.

El gate quedó en `opportunity_handler_message_buy` / `opportunity_handler_message_sell`:

```python
if not DataHub.mercado_abierto(row.get("vehiculo", "Stock")):
    return
```

- **Por fila, no por agente** — los CSV mezclan vehículos (columna `vehiculo`). Un gate en
  `Agente_ManagerBuy` habría silenciado también Crypto, que sí opera sábado.
- **La oportunidad se inserta igual en BD**; lo único que se corta es la notificación
  (Telegram + chat interno). No se pierde dato para entrenamiento.
- Alcanza también al TOP10: al no marcarse en `buy_enviados`/`sell_enviados`, la fila no entra
  a `get_top_buy`/`get_top_sell`.
- Helper: `DataHub.mercado_abierto(vehiculo)` → ver [[ref-datahub]].
- ⚠️ El `hash_id` incluye `Fecha = datetime.now().date()`, así que cada día nuevo reenvía la
  misma lista. Con el gate el fin de semana queda callado, pero el reenvío diario en días
  hábiles sigue vigente. Pendiente.

---

## 2. Regla BUY vs DIVIDENDS

Un activo BUY se clasifica en una de dos categorías según dividendos:

| Condición | Categoría | Clave en `self.info` |
|-----------|-----------|----------------------|
| `dividendo == 0` | **buy** | `"buy"` |
| `dividendo > 0` | **dividends** | `"dividends"` |

Una sola categoría por activo, nunca ambas.

---

## 3. Señales por lado

| Señal | BUY | SELL |
|-------|-----|------|
| RSI | Bajo (<40) = sobreventa | Alto (>70) = sobrecompra |
| Tendencia EMA | 20 > 50 > 100 (alcista) | 20 < 50 < 100 (bajista) |
| Confianza IA | >= 0.65 → COMPRAR | >= 0.65 → VENDER |
| Features escalares | ganancia_precio, dividend_yield, score | roi, profit |
| Features técnicas | RSI, MACD, EMAs, Fibonacci, max/min 13/26/52 sem (timeframe _d) |

---

## 4. Umbrales

| Zona | Condición | Acción |
|------|-----------|--------|
| **COMPRAR / VENDER** | confianza >= 0.65 | Envía mensaje Telegram canal Buy/Sell |
| **Observar** | 0.35 <= confianza < 0.65 | Aparece en panel Monitor IA, no envía |
| **Ignorar** | confianza < 0.35 | Descartado |

Umbrales configurables en tabla BD `modelos_ia`.

---

## 5. Score Híbrido — Ranking TOP 10

Reemplaza el filtro binario para el ranking TOP10. Combina IA + señales técnicas.

### BUY
```
score = (confianza × 0.50) + (rsi_score × 0.30) + (tendencia_ema × 0.20)

rsi_score    = max(0, (60 - RSI) / 60)      # RSI 37 → 0.38 | RSI 60+ → 0
tendencia_ema = 1.0 si EMA20>EMA50>EMA100
              = 0.5 si EMA20>EMA50 (parcial)
              = 0.0 sin alineación
```

### SELL (simétrico)
```
score = (confianza × 0.50) + (rsi_score_sell × 0.30) + (tendencia_ema_sell × 0.20)

rsi_score_sell = max(0, (RSI - 60) / 40)    # RSI 75 → 0.375 | RSI <60 → 0
tendencia_ema_sell = 1.0 si EMA20<EMA50<EMA100
                   = 0.5 si EMA20<EMA50 (parcial)
                   = 0.0 sin alineación
```

**Umbral de entrada TOP10:** `score >= 0.25`
**Candidatos sin confianza** (origen system): solo entran si `score_rebalanceo >= MinScoreBuy AND ganancia_precio >= MinGananciaPrecio`.

---

## 6. Ciclo de entrenamiento (aplica a BUY y SELL)

```
BD oportunidades (tipo='buy' / tipo='sell')
          │
          ▼
  obtener_por_tipo() → convertir_dataset_entrenamiento()
          │
          ▼
  aplanar_datos_tecnicos()   ← extrae RSI/MACD/EMAs del JSON
          │
          ▼
  entrenar_modelo()
    ├── StratifiedKFold 5 folds
    ├── RandomForestClassifier (n=100, depth=10)
    └── métricas: precision / recall / F1 / accuracy / ROC-AUC
          │
          ▼
  save_modelo() → modelo_buyv01.pkl / modelo_sellv01.pkl
          │
          ▼
  Próximo ciclo de evaluación usa el modelo guardado
```

**Disparador:** Botón "Entrenar" en tab Monitor IA (Class_SystemStatus).
**Etiquetado:** usuario aprueba/rechaza en Telegram → `Recomendado = 1/-1` → tabla `oportunidades`.

---

## 7. Archivos clave

| Archivo | Rol |
|---------|-----|
| `Class_DashBot.py` | Agentes ManagerBuy/Sell, handlers, TOP10, read CSV |
| `Class_IA_modelos.py` | Clases ModeloOportunidadesBuy/Sell, entrenamiento, predicción |
| `Class_customer.py` | `csv_OptionBuy_write()` — genera csv_datosIA_buy.csv |
| `csv_datosIA_buy.csv` | Input modelo BUY (datos de rebalanceo + técnicos) |
| `csv_datosIA_sell.csv` | Input modelo SELL |
| `modelo_buyv01.pkl` | Modelo entrenado BUY |
| `modelo_sellv01.pkl` | Modelo entrenado SELL |
| BD `oportunidades` | Historial etiquetado para entrenamiento |
| BD `modelos_ia` | Parámetros y umbrales configurables |

---

## 8. Camino único de predicción

El panel Sell IA y el agente daban **números distintos para la misma fila**. El agente llamaba a
`enriquecer_con_sentimiento`; el panel no, y `predecir_modelo` rellenaba en silencio las cuatro
columnas de sentimiento ausentes con `0.0`. Medido sobre el CSV vivo: PBR 33% daba **0.9449** en el
panel y **0.5705** en el agente. En BUY el desvío va al revés — el panel subreporta (PFLT 0.470 vs
0.694 real), así que con umbral 0.65 mostraba 3 COMPRAR donde el camino real da 5.

Hoy los dos consumen `ModeloOportunidades{Sell,Buy}.predecir_oportunidades(df, account, columnas_map)`,
que hace la secuencia completa: `hash_id → rename → aplanar → sentimiento → modelo → merge por hash_id`.
Cualquier atajo vuelve a separar los dos números.

El relleno silencioso de features ya no es silencioso: `predecir_modelo` loguea las columnas que
rellenó en el logger `ClassMoodeloIA` (WARNING).

**Efecto secundario aprovechado:** `predecir_oportunidades` devuelve **todas** las filas con su
confianza, no solo las aprobadas — de ahí sale la traza de la capa 3.

**Punto abierto:** `load_sentiment_features(account)` carga una sola cuenta. El CSV de venta trae tres
(`U4214563`, `BBVA0001`, `SANT0001`), así que las filas de las otras dos siguen prediciendo con
sentimiento en `0.0`. Preexistente, no lo introdujo este cambio.

---

## 9. Trazabilidad — `json_detalle.capas`

Una oportunidad de venta atraviesa 7 filtros. Cuando no llega a Telegram, la pregunta siempre era
*cuál* la frenó, y no había forma de saberlo sin leer logs de tres módulos.

| # | Capa (clave en el JSON) | Filtro |
|---|---|---|
| 1 | `1_csv_OptionSales_write` | piso por vehículo — Stock: ROI ≥9%, ganancia ≥$90 |
| 2 | `2_Agente_ManagerSell` | lee el CSV sin filtrar (cada 15s) |
| 3 | `3_evaluar_oportunidades_sell_con_IA` | técnicos + sentimiento → modelo → confianza ≥ 0.65 |
| 4 | `4_oportunity_handler_sell` | inserta en `oportunidadesbuysell` — acá sí o sí queda registro |
| 5 | `5_gate_menu` | `MostrarOpcionMenu_enTelegram == "Sell"` |
| 6 | `6_Agente_message_Manager_sell` | mejora de ROI, tiempo mínimo, gate Consenso |
| 7 | `7_opportunity_handler_message_sell` | calendario del vehículo → Telegram |

Cada casillero es `{"ok": bool, "dato": "<motivo o valor medido>"}`. Cada capa se anota **donde
realmente decide** — no hay un lugar central que reconstruya el resultado.

```json
"capas": {
  "1_csv_OptionSales_write":             {"ok": true,  "dato": "roi 26.10% | ganancia 94.93"},
  "2_Agente_ManagerSell":                {"ok": true},
  "3_evaluar_oportunidades_sell_con_IA": {"ok": false, "dato": "confianza 0.57 vs umbral 0.65"}
}
```

**No toca `timestamp`.** `actualizar_capas()` usa `JSON_SET` sobre `$.capas` solamente, mientras que
`actualizar_oportunidad()` pisa `timestamp = NOW()` en sus dos ramas. Esa distinción es lo que permite
seguir usando el `timestamp` como señal de diagnóstico: fue una fila de venta congelada a las 18:14
mientras las compras marcaban 20:12 lo que probó que el pipeline de venta había dejado de producir.
Anotar una capa no es una corrida.

**Escribe solo cuando cambia el veredicto.** `Agente_ManagerSell` no tiene `@wait_rate`: corre dentro
del loop de `agentesIA()` con `time.sleep(15)`, todo el día → **5.760 ciclos/día**. Con 23 filas en el
CSV de venta, escribir en cada pasada serían **~132.000 UPDATE/día**.

La firma en memoria (`_capas_ultimo`) toma **solo los `ok`, nunca el `dato`**. Es la parte que importa:
el `dato` de la capa 1 es `roi 26.10% | ganancia 94.93` — valores vivos que se mueven con cada tick, así
que una firma sobre el JSON completo cambia casi siempre y el candado no frena nada. Con la firma sobre
el veredicto, el volumen queda acotado por los cambios de estado, no por los ciclos.

**Consecuencia a tener presente:** el `dato` guardado es el del momento en que el veredicto cambió por
última vez, no el actual. Es una foto del porqué, no un valor en vivo — para el valor vigente están
`roi` y `profit` del mismo `json_detalle`, que sí se refrescan en cada corrida.

**Límites conocidos:**

- **El orden real no es 6→7.** En el código el gate de calendario (capa 7) corre *antes* que
  `Agente_message_Manager_sell` (capa 6). La numeración se conserva porque es la que se usa para leer
  el flujo; en el JSON eso se ve como un `7:false` sin `6` cuando el mercado está cerrado.
- **Una fila descartada en capa 3 solo deja rastro si ya tiene oportunidad en BD.** La fila nace en la
  capa 4; un símbolo que nunca la alcanzó no tiene dónde escribir. Ese caso es de
  `symbol_decision_history` y quedó fuera.
- **Solo SELL.** BUY tiene otros nombres de capa.

---

## Historial
| Versión | Fecha | Cambio |
|---------|-------|--------|
| 2.3 | 2026-09-03 | Camino único de predicción (panel = agente) + traza de las 7 capas en `json_detalle.capas` |
| 2.2 | 2026-08-22 | Gate de calendario en los message handlers — no se notifica vehículo cerrado |
| 2.1 | 2026-08-22 | Umbral SELL por CLASE y por vehículo (`gains_oportunidades`), fail-open cerrado en `readCSV_sell` |
| 2.0 | 2026-06-29 | Unificación buy+sell, score híbrido TOP10, elimina duplicados |
| 1.0 | 2026-04 | Documentos separados ref-regla-buy, ref-modelo-buy, ref-modelo-sell |
