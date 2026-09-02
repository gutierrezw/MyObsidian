# BotCrypto — Especificación Técnica
**AppOO · Binance SPOT · Versión 2026-W12**

---

## 1. Objetivo y Alcance

Bot de trading **Spot** para Binance que evalúa activos individuales y toma decisiones de compra/venta automática basadas en indicadores técnicos clásicos.

- **No predice precios** — identifica oportunidades razonables de entrada/salida
- **No agrega activos** al universo — el universo `otros_activos` se gestiona manualmente
- **Mercado:** Binance Spot · Pares crypto (BTCUSDT, ETHUSDT, etc.)
- **Operativa:** Market / Limit · Solo LONG

### Fuera de alcance — GainsCapture y Preservation (decisión 2026-08-23)

**BotCrypto es un bot de trading puro y no participa de los agentes de cartera.** No entra en
[[design-gains-capture]] ni en [[design-preservation]], y no es un pendiente: es la frontera del
diseño. El bot abre y cierra sus propias posiciones con su propia lógica de entrada/salida
(§ 4.3) y su propia gestión de riesgo (§ 6). Superponerle un agente que decida ventas por ROI o
que mueva stops sería **dos dueños sobre la misma posición**.

Consecuencias verificadas en BD, que confirman que hoy ya es así:

| Evidencia | Estado |
|---|---|
| Filas en `inversion` con `tipoinv = 'BotCrypto'` | **0** — GainsCapture y Preservation iteran `select_inversion()`, así que nunca lo ven |
| Filas en `oportunidadesbuysell` con `vehiculo = 'BotCrypto'` | **0** — no pasa por el pipeline de oportunidades |
| Bloques `gains_capture` / `gains_oportunidades` en `sesion.parameters` | **ausentes** — y así deben quedar |

Por eso **no hay que agregarle esos bloques de parámetros**: nadie los leería. Si en algún
momento aparece un `gains_config("BotCrypto", …)` en el código, es un error de diseño, no una
funcionalidad faltante.

`B0000002` sí participa de todo lo que es contabilidad y auditoría — `booktrading`, comisiones,
`diaria_performance`. La exclusión es solo de los agentes que **deciden operaciones**.

---

## 2. Principios de Diseño

- Separación estrategia / ejecución
- Parametrización total
- Estado persistente y verificable — el bot no asume, siempre verifica
- Persistencia histórica (operaciones, señales, resultados)
- Tolerancia a fallos (API, red, datos)

---

## 3. Arquitectura del Sistema

```
+---------------------------+
|      Binance Websocket    |
|   (klines / ticker)       |
+-------------+-------------+
              |
              v
+---------------------------+
|      Websocket Listener   |
|  (event dispatcher)       |
+-------------+-------------+
              |
              v
+---------------------------+
|        BotManager         |
|  - Control global         |
|  - Enrutamiento símbolos  |
+------+------+-------------+
       |      |
       v      v
+-----------+  +-----------+
| Bot BTC   |  | Bot ETH   |   ... (N símbolos)
| Strategy  |  | Strategy  |
| Risk      |  | Risk      |
| State     |  | State     |
+-----+-----+  +-----+-----+
      |              |
      +--------------+
             v
     +-------------------+
     |  Binance Spot API |
     | (órdenes, estado) |
     +-------------------+
             |
             v
     +-------------------+
     |   MySQL / Storage |
     | estado / trades   |
     +-------------------+
```

**Principios arquitecturales:**
- Un bot lógico por símbolo
- Websocket como fuente primaria de eventos
- API REST solo para confirmación y órdenes
- Solo evaluar velas cerradas — nunca ejecutar órdenes dentro del callback

---

## 4. Estrategia de Trading

### 4.1 Filtro Estructural — Régimen del Mercado (EMA100/200)

**Primera barrera antes de cualquier evaluación.** Evita entradas contra tendencia media.

```python
def _check_market_regime(self, df):
    ema100 = df["ema100"].iloc[-1]
    ema200 = df["ema200"].iloc[-1]
    close  = df["close"].iloc[-1]
    if close > ema100 and ema100 > ema200:
        return "BULL"
    else:
        return "NO_LONG"
```

| Régimen | Condición | Acción |
|---------|-----------|--------|
| **BULL** | close > EMA100 AND EMA100 > EMA200 | Permitir evaluación |
| **RANGE** | EMA100 ≈ EMA200 | Reducir size / no operar |
| **BEAR** | close < EMA100 OR EMA100 < EMA200 | NO_SIGNAL — no operar |

> Causa raíz de rachas perdedoras: intentar rebotes en régimen BEAR/RANGE.
> El filtro reduce frecuencia pero aumenta probabilidad por trade.

---

### 4.2 Condiciones de Entrada LONG

Se requiere **todas** las condiciones en vela cerrada:

1. **RSI** — RSI < 35 con pendiente positiva (RSI actual > RSI previo)
2. **MACD** — cruza por encima de Signal; histograma > 0 o creciendo
3. **EMA** — EMA rápida > EMA lenta; distancia creciente (momentum)
4. **Volumen** (opcional) — volumen vela >= promedio (evitar falsas rupturas)

---

### 4.3 Condiciones de Salida

**A. Invalidación técnica** (cualquiera):
- RSI > 65 girando a la baja
- MACD cruza por debajo de Signal
- EMA rápida cruza por debajo de EMA lenta

**B. Toma de ganancia:**
- TP1: +X% → vender 25–33% de posición
- TP2: +Y% → vender otro 25–33%
- Resto: trailing stop sobre EMA lenta

---

### 4.4 Zonas de No-Operación

- RSI entre 45–55 (zona neutra)
- MACD plano sin divergencia
- EMAs entrelazadas (mercado lateral)
- Volumen anormal extremo (eventos)

---

## 5. Sistema de Scoring y Rotación

Dentro del universo `otros_activos`, no todos los activos operan simultáneamente. El scoring dinámico decide prioridad operativa en cada ciclo.

### 5.1 Score Técnico Base

```
Score_Base = Score_RSI + Score_MACD + Score_EMA + Score_Volatilidad
```

| Indicador | Condición | Puntos |
|-----------|-----------|--------|
| **RSI** | RSI > 50 | +1 |
| | RSI cruzando 30 desde abajo | +2 |
| | RSI > 70 | -1 |
| **MACD** | MACD > Signal | +1 |
| | Cruce alcista reciente | +2 |
| | Cruce bajista | -2 |
| **EMA** | Precio > EMA200 | +2 |
| | EMA50 > EMA200 | +1 |
| | Precio < EMA200 | -2 |
| **Volatilidad** | ATR/Precio saludable | +1 |
| | Volatilidad extrema | -1 |
| | Volatilidad muy baja | 0 |

---

### 5.2 Contexto Superior (penalización estructural)

El contexto 4H no es una señal más — es un **modificador estructural** del score.

```
score_total = score_base + score_contexto
```

**Evaluación del contexto 4H** (`evaluar_contexto_superior()`):

| Condición | Descripción |
|-----------|-------------|
| EMA20 > EMA50 | Tendencia alcista media |
| Precio > EMA20 | Precio sobre estructura |
| MACD > 0 | Impulso positivo |
| RSI > 50 | Momentum favorable |

- 3 o 4 condiciones → `context_ok = True` → `score_contexto = 0`
- Menos de 3 → `context_ok = False` → `score_contexto = CONTEXTO_PENALIZACION (-5)`

**Protección anti-confusión:**
```python
if not contexto["context_ok"]:
    score_total = min(score_total, 0)  # nunca lidera el ranking fuera de contexto
```

---

### 5.3 Selector de Timeframe Dinámico

El timeframe no es fijo — se decide **antes** de abrir cada operación según el estado real del mercado.

**Clasificación contexto 4H:**

| Score 4H | Fase | Timeframe | Ajuste riesgo |
|----------|------|-----------|---------------|
| 4/4 condiciones | Fuerte | 30m si ATR%>2.5 y Vol>1.3, sino 1H | Normal |
| 3/4 | Moderado | Solo 1H | -20% a -30% |
| 2/4 | Débil | 1H solo si score alto | Reducido |
| 0–1 | Bajista | No operar | — |

**Métricas:**
- `ATR% = ATR(14) / Precio` — >2.5% expansión, 1.5–2.5% moderado, <1.5% compresión
- `Vol_ratio = Volumen / Promedio_20` — >1.5 fuerte, 1.2–1.5 normal, <1.2 bajo

> Regla: no cambiar temporalidad con posición ya abierta.

---

### 5.4 Clasificación por Prioridad y Rotación

```
Score >= 5  → PRIORIDAD ALTA
Score 3–4   → PRIORIDAD MEDIA
Score 1–2   → OBSERVACIÓN
Score <= 0  → BLOQUEADO
```

**Reglas de rotación:**
- Ordenar activos por `Score_Total` descendente
- Solo los Top N (`max_positions`) pueden abrir nuevas posiciones
- Si un activo sale del Top N: no se cierran posiciones activas, solo se bloquean nuevas entradas
- Control de exposición global: `exposicion_actual + nueva_posicion <= max_exposure_global`

---

## 6. Gestión de Riesgo

> "El bot puede equivocarse en la entrada; nunca debe equivocarse en cuánto puede perder."

- **Riesgo por trade:** 1–2% del capital asignado al bot
- **Tamaño de posición:** calculado antes de enviar orden (capital × %riesgo / distancia_SL)
- Respetar `minQty` y `stepSize` de Binance — redondear siempre hacia abajo
- **Stop Loss técnico** (no fijo): último mínimo relevante / EMA lenta / % fijo de emergencia
- SL se define antes de entrar; si queda demasiado cerca → no operar
- SL solo se ajusta a favor (trailing)

**Take Profit parcial:**

| Nivel | % desde entrada | Acción |
|-------|----------------|--------|
| TP1 | +X% | Vender 25–33% → SL a break-even |
| TP2 | +Y% | Vender otro 25–33% → SL dinámico sobre EMA lenta |
| Resto | — | Trailing stop; dejar correr |

**Reglas de supervivencia:**
- Máximo N trades negativos consecutivos → pausa automática
- Stop diario de pérdidas
- Pausa tras drawdown

---

## 7. Gestión de Estado

> "Si el estado no es confiable, el bot debe dejar de operar inmediatamente."

**Variables persistidas por símbolo:**

| Variable | Tipo | Descripción |
|----------|------|-------------|
| `position_status` | NONE / LONG | Estado actual |
| `entry_price` | float | Precio de entrada |
| `position_qty` | float | Cantidad total |
| `remaining_qty` | float | Cantidad disponible |
| `tp1_executed` | bool | TP1 ejecutado |
| `tp2_executed` | bool | TP2 ejecutado |
| `stop_loss_price` | float | SL actual |
| `last_decision` | BUY/SELL/HOLD | Última señal |
| `last_update_timestamp` | datetime | Timestamp |

**Máquina de estados:**
```
FLAT → ENTERING → IN_POSITION → PARTIAL_EXIT → EXITING → FLAT
                                                        ↘ ERROR
```

**Al reiniciar:**
1. Cargar estado persistido
2. Consultar posiciones y órdenes reales en Binance
3. Reconstruir estado local
4. Validar cantidades — en caso de inconsistencia: priorizar estado Binance
5. Si no se puede resolver → estado ERROR → no operar

**Reglas de integridad:**
- Nunca abrir posición si `position_status != NONE`
- Nunca vender más de `remaining_qty`
- Nunca ejecutar TP dos veces
- Nunca modificar SL en contra de la posición

---

## 8. Flujo Websocket (Event-Driven)

```
[Websocket Binance]
        |
        v
[Parseo y validación vela cerrada]
        |
        v
[BotManager.on_event()]
        |
        v
[BotSymbol.on_market_data()]
        |
        v
[1. _check_market_regime()]     ← BEAR/RANGE → NO_SIGNAL
        |
        v
[2. evaluar_contexto_superior()] ← penaliza score si 4H débil
        |
        v
[3. calcular_score_total()]      ← base + contexto
        |
        v
[4. seleccionar_timeframe()]     ← 30m vs 1H
        |
        v
[5. evaluate() → BUY/SELL/HOLD]
        |
        +--> HOLD
        |
        +--> BUY/SELL
                |
                v
         [Gestión de Riesgo]
                |
                v
         [Enviar orden Binance Spot]
                |
                v
         [Actualizar estado + persistir MySQL]
```

---

## 9. Persistencia y Auditoría

Guardar por operación:
- Señal generada (BUY/SELL/HOLD + score_base + score_contexto + timeframe)
- Orden enviada y confirmación
- PnL realizado
- `contexto_ok` (permite analizar trades con/sin contexto alineado)

---

### 9.1 Diaria y performa — `categoria = 'BotCrypto'` (corregido 2026-09-02)

BotCrypto no tiene una cadena propia: comparte `diaria_performance` → `performa_inversion`
con el resto de los vehículos, alimentada por `schedule_diario` → `diaria_book_performance()`
→ `detalle_book()` (`Modulos_Comunes.py`).

`detalle_book()` elegía la fuente de precios por la columna `booktrading.categoria`, y solo
reconocía `Stock` / `BBVA.ARS` / `Crypto`. Las filas de B0000002 traen **`categoria = 'BotCrypto'`**,
así que caían al `else` y le pedían `ADAUSDT` a Yahoo — que no lo resuelve. Efecto en cadena:

1. `csv_app_BotCrypto.csv` salía **con solo el header** — ningún símbolo escribía fila.
2. `diaria_performance` de B0000002 quedó congelada el **2026-08-27**.
3. `performa_inversion` no pudo avanzar más allá de esa fecha.
4. `check_performance_vehiculo()` (`Class_gestion.py`) pide
   `fechaclose >= ultimo_dia_habil(mes)` → BotCrypto **bloqueó el cierre de agosto de los
   cinco vehículos**, porque `cierre_extractos()` es todo-o-nada.

Nada de esto era visible: `diaria_book_performance()` devolvía `update=True` aunque no hubiera
insertado una sola fila, así que `schedule_diario` estampaba `diaria_BotCrypto` en
`tmp/agents_schedule.json` y logueaba *"procesada OK"*. Y como `accion="diaria_app"` **no
reconstruye hacia atrás**, cada día estampado se perdía para siempre.

**Los dos arreglos:**

| Archivo | Cambio |
|---|---|
| `Modulos_Comunes.py` — `detalle_book()` | `elif categoria in ("Crypto", "BotCrypto"):` — mismo idiom que ya usaba `vehiculo_parm()` |
| `Modulos_Comunes.py` — `diaria_book_performance()` | `update = bool(diaria)` + WARNING: sin filas insertadas el día **no** está procesado y la key no se estampa |

**Efecto lateral aceptado del segundo:** en feriado de mercado sin datos nuevos el schedule
reintenta cada 3 h y loguea el WARNING en vez de estampar. Es ruido; perder el día en silencio
era el daño real.

**La serie histórica quedó mal desde el origen.** Con la categoría rota los valores salieron de
un ticker mal resuelto: el 27/08 DOGEUSDT figuraba con `value = 46756` sobre 402 unidades
(close implícito 116 USD) cuando el close real de Binance era **0,08137** → ~33 USD. Toda la
`diaria_performance` de B0000002 entre 2026-02-07 y 2026-08-27 está inflada en órdenes de
magnitud y se reconstruye con
`AppTest/run_rebuild_diaria_cartera.py --account B0000002 --vehiculo BotCrypto --desde 2026-02-07 --ejecutar`.

---

## 10. Tabla `otros_activos`

```sql
CREATE TABLE otros_activos (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    symbol      CHAR(25),
    cuenta      CHAR(10),
    idcrypto    BIGINT,
    descripcion CHAR(50),
    base_asset  CHAR(25),
    quote_asset CHAR(10),
    avgcost     FLOAT,
    objetivo    FLOAT,
    indicadores BLOB,
    fecupdate   DATETIME
);
```

---

## 11. Seguridad

- API Keys sin permisos de retiro
- Credenciales en BD (sesión Binance) — nunca en código
- Logs sin datos sensibles

---

## 12. Evolución Futura

- Backtesting con mismo flujo de eventos (event sourcing)
- Escalado multi-proceso
- Integración scoring IA (`inst_score` + señales de mercado Stock)
