# Preservation Dinámico con Claude — Diseño

**Estado:** Diseño inicial — pendiente implementar  
**Fecha:** 2026-06-02  
**Prioridad:** Alta

**Ver también:**
- [[design-gains-capture.md]] — agente complementario (especulativo, captura upside)
- [[analysis-agent-history-table.md]] — tabla unificada para aprender de ambos agentes
- [[design-autonomo]] — con qué mandato vende este módulo dentro del modo autónomo

**Activos objetivo:** `categoriaActivo` IN ('I', 'S') en tabla `market`  
- `I` = **Infravalorado** — precio bajo vs dividendo/earnings (defensivo, compra activa)
- `S` = **Sobrevalorado** — precio alto vs fundamentals (ya en cartera, proteger ganancia)
- Complemento: `Agente_GainsCapture` opera sobre `N` (sin dividendos = volátiles especulativos)

---

## Problema actual

El agente `Agente_ManagerPreservation` usa reglas fijas:
- `roi_minimo = 10%` — umbral de activación
- `correccion_pct = 8%` — stop fijo bajo max_price
- `atr_mult = 2.0` — componente ATR

**Limitación:** No distingue contexto. CRNT con ROI 7%, señal VENDER y RSI bajando
es tratado igual que CRNT con ROI 7% en tendencia alcista → ambos son SKIP.
Un analista humano vería la primera combinación y activaría protección.

---

## Solución propuesta

Integrar Claude Haiku en `_preservation_run_vehiculo()` para razonar sobre cada
posición antes de decidir. Las reglas fijas siguen siendo el **piso y fallback**.

```
Reglas fijas (siempre corren)
  └── calculan stop_calculado base
Claude Haiku (corre sobre umbral ampliado 7%)
  └── decide activar/ajustar según contexto técnico/fundamental
  └── si falla → reglas fijas toman el control
```

---

## Datos a pasar a Claude por posición

### Del agente (ya disponibles)
| Campo | Fuente |
|---|---|
| symbol, qty, roi | `inversion` / IB |
| last, max_price, stop_actual | DataHub / preservation_state.json |
| atr | yfinance ATR(14) |
| base_limit | unrealizedpnl × proteccion_base |

### Desde `market` table (MarketScreen)
| Campo | Descripción |
|---|---|
| consenso_tag | UNANIME/CONSENSO/TENDENCIA/NEUTRO/ALERTA/SALIDA |
| consenso_suma | Suma de votos (+4 a -6) |
| inst_score | Score institucional v2 |
| inst_ownership_pct | % float en manos institucionales |
| fh_buy_ratio | % fondos 13F comprando |
| rotacion | Momentum institucional |
| analyst_rec | buy/hold/sell |
| analyst_mean | Score promedio analistas |

### Desde `market_sentiment` / `market_sentiment_analysis`
| Campo | Descripción |
|---|---|
| sentiment_score | +1/0/-1 puntual |
| patron | acumulacion/distribucion/neutro/inflexion |

### Desde `booktrading.indicadores` (ya grabado en cada transacción)
| Campo | Descripción |
|---|---|
| rsi_d | RSI diario |
| macd_d | MACD diario (estado) |
| EMA200 posición | sobre/bajo EMA200 |
| rango_52w_pct | posición dentro del rango anual |

---

## Rol de Claude — Opción B (acordada)

**Claude NO decide si activar** — eso lo hacen las reglas fijas (roi >= 10%, siempre).  
**Claude SOLO afina el stop** sobre posiciones que ya pasaron el umbral.

```
Reglas fijas (roi >= 10%)  →  activan preservation, calculan stop base
Claude Haiku               →  ajusta nivel del stop + agrega razon + urgencia
Si Claude falla (timeout)  →  stop base de reglas fijas se coloca igual
```

**Sin gray zone:** ninguna posición queda desprotegida por falla de Claude.

---

## Prompt a Claude (por posición)

```
Eres un agente de preservación de ganancias para un portfolio de inversión.
Las reglas fijas ya activaron la protección de esta posición (ROI >= 10%).
Tu tarea es ajustar el nivel del STOP para maximizar la protección según el contexto.

Posición: {symbol}
- ROI actual: {roi:.1%} | Precio: ${last:.2f} | Max histórico: ${max_price:.2f}
- Stop base (reglas): ${stop_calculado:.2f} | Stop anterior: ${stop_actual:.2f}
- ATR(14): ${atr:.2f}

Contexto fundamental:
- Consenso: {consenso_tag} ({consenso_suma:+d} votos)
- Inst Score: {inst_score} | 13F Buy ratio: {fh_buy_ratio:.1%}
- Analistas: {analyst_rec} (mean={analyst_mean:.1f})
- Sentimiento: {patron} (score={sentiment_score:+d})

Técnico diario:
- RSI: {rsi_d} | MACD: {macd_estado}
- EMA200: precio {ema200_rel}
- Rango 52W: {rango_52w_pct:.0%}

Podés subir el stop (más protección) o mantener el base.
NUNCA sugerir un stop inferior al base calculado por reglas.
Respondé SOLO con JSON válido:
{"stop_sugerido": float, "razon": "str max 120 chars", "urgencia": "alta"|"media"|"baja"}
```

---

## Lógica de activación — Opción B

```python
# Umbral único: reglas fijas siempre mandan
UMBRAL_REGLAS = 0.10   # 10% — sin cambio respecto al agente actual

if roi >= UMBRAL_REGLAS:
    # 1. Reglas fijas calculan el stop base (siempre)
    stop_calculado = max_price - max(correccion_pct * max_price, atr_mult * atr)
    
    # 2. Claude afina el nivel (opcional — si falla, stop base se usa igual)
    context = _build_preservation_context(symbol, ...)
    claude = _claude_preservation_eval(context)  # timeout 15s
    
    stop_claude = claude.get("stop_sugerido") if claude else None
    stop_final = max(stop_anterior, stop_calculado, stop_claude or 0)
    
    # 3. Colocar STOP siempre — Claude no puede bloquearlo
    # → colocar orden STOP con stop_final
```

**Invariante clave:** `stop_final >= stop_calculado` siempre.
Claude solo puede subir el stop, nunca bajarlo ni cancelarlo.

---

## Persistencia de decisiones — `order_trader`

**Preservation registra STOP orders en order_trader** con dos niveles de información:

### 1. `json_detalle` — Contexto técnico/decisión (ya existe)

```json
{
  "tipo": "preservation_stop",
  "decision": {
    "roi": 0.07,
    "max_price": 3.01,
    "atr": 0.15,
    "stop_calculado_reglas": 2.73,
    "consenso_tag": "NEUTRO",
    "consenso_suma": 0,
    "inst_score": 45,
    "fh_buy_ratio": 0.32,
    "sentiment_patron": "distribucion",
    "rsi_d": 42,
    "macd_estado": "bajista",
    "ema200_rel": "bajo",
    "base_limit": 89.26
  },
  "claude": {
    "activar": true,
    "stop_sugerido": 2.78,
    "razon": "RSI bajando desde zona sobrecomprada...",
    "urgencia": "media"
  },
  "resultado": {
    "stop_final": 2.78,
    "qty_protegida": 25,
    "ganancia_protegida_usd": 89.26
  }
}
```

### 2. `json_audit_log` — Histórico de eventos ([[analysis-agent-history-table.md]])

```json
{
  "events": [
    {
      "ts": "2026-08-03T09:27:25.963",
      "tag": "CLAUDE",
      "msg": "stop_sugerido=43.86 urgencia=media",
      "data": {
        "stop_sugerido": 43.86,
        "urgencia": "media",
        "roi": 0.122,
        "rsi_d": 67,
        "macd_estado": "alcista"
      }
    },
    {
      "ts": "2026-08-03T09:27:25.389",
      "tag": "ATR-CAP",
      "msg": "stop 44.42 recortado a 42.86",
      "data": {
        "stop_before": 44.42,
        "stop_after": 42.86,
        "atr": 0.98
      }
    },
    {
      "ts": "2026-08-03T09:27:25.400",
      "tag": "ENVIADA",
      "msg": "STP LMT 4 acc @ 42.86",
      "data": {
        "order_id": 886042637,
        "stop_final": 42.86,
        "qty": 4
      }
    },
    {
      "ts": "2026-08-03T21:27:22.930",
      "tag": "CLAUDE",
      "msg": "RSI 67 indica sobrecalentamiento. Subir stop",
      "data": {
        "stop_sugerido": 43.26,
        "urgencia": "media"
      }
    },
    {
      "ts": "2026-08-03T21:27:31.301",
      "tag": "MODIFICADA",
      "msg": "Orden cancelada + nueva STP LMT @ 43.26",
      "data": {
        "order_id_old": 886042637,
        "order_id_new": 886042638,
        "stop_before": 42.86,
        "stop_new": 43.26
      }
    },
    {
      "ts": "2026-08-04T10:15:00.000",
      "tag": "FILLED",
      "msg": "STOP completado @ 43.20",
      "data": {
        "precio_fill": 43.20,
        "ganancia_realizada_restante": 22.96
      }
    }
  ]
}
```

### Granularidad de la trama por vehículo (fix 2026-08-23)

`preservation_build_trama` armaba la rama Crypto con `float(round(stop_price, 2))` en `price` y
`stopPrice`. En cualquier par con `tickSize` < 0.01 (VET, DOGE, ADA) eso manda a Binance un precio
distorsionado o directamente inválido: 0.0234 → 0.02. Ahora usa `DataHub.quantiza_precio()` y
`DataHub.quantiza_qty()`, que truncan a `tickSize`/`stepSize` leyendo
`DataHub.info[symbol]["lotSize"]`. `preservation_calc_qty()` llama al mismo `quantiza_qty` en vez
de repetir la fórmula inline. La rama Stock de ese método no cambió.

### Campos guardados en order_trader (Preservation)

| Campo | Valor | Propósito |
|-------|-------|----------|
| `account` | "U4214563" | Cuenta destino |
| `vehiculo` | "Stock" o "Crypto" | Tipo vehículo |
| `symbol` | "BP" | Símbolo |
| `conid` | 5171 | Contract ID |
| `clientOrderId` | "886042637" | Order ID IB |
| `orderType` | "STP LMT" | Stop Limit |
| `side` | "SELL" | Siempre SELL (protección) |
| `price` | 42.83 | Límite ejecución (stop × 0.99) |
| `auxPrice` | 42.86 | Precio STOP actual |
| `quantity` | 4.0 | Cantidad protegida |
| `tif` | "GTC" | Good Till Cancelled |
| `intent` | "PRESERV" | Identificador agente |
| `status` | "ENVIADA" → "FILLED" | Ciclo vida |
| `stampPlace` | NOW() | Cuándo se colocó |
| `stampSubmit` | NOW() | Cuándo se confirmó |
| `json_detalle` | {decisión completa} | Contexto técnico + Claude |
| `json_audit_log` | {eventos} | Histórico CLAUDE → ENVIADA → MODIFICADA → FILLED |

### Ciclo de vida de la orden en `order_trader`

| Estado | Significado | Aprendizaje |
|---|---|---|
| NEW | STOP activo, posición protegida | — |
| FILLED | Stop tocado → ganancia limitada | Decisión correcta si alternativa era peor |
| CANCELLED (ajuste) | Precio subió → nueva orden con stop más alto | Stop conservador inicial |
| CANCELLED (cierre) | Posición cerrada por otra vía | Preservación exitosa o innecesaria |

### Valor futuro del dataset
- **Audit trail**: por qué se colocó cada stop
- **Training data**: cruzar decisión Claude + contexto vs outcome (FILLED/CANCELLED)
- **Análisis efectividad**: qué combinación roi+consenso+RSI tuvo mejor resultado
- **Calibración umbrales**: ajustar UMBRAL_CLAUDE según histórico de aciertos

---

## Configuración técnica

| Parámetro | Valor | Motivo |
|---|---|---|
| Modelo | `claude-haiku-4-5-20251001` | Costo mínimo ~$0.001/posición |
| Key BD | `ClaudeAPIP` (nueva) o `ClaudeAPIC` | Separar costos por módulo |
| Timeout | 15s | Si no responde → reglas fijas |
| Log | `preservation_diag.log` | Incluir razon Claude en cada línea |
| Frecuencia | Por posición, cada revisión (2x día) | Solo si roi >= UMBRAL_CLAUDE |

---

## Casos de uso identificados

| Caso real | Regla fija | Con Claude |
|---|---|---|
| CRNT ROI 7%, VENDER, RSI 42 bajando | SKIP | ✅ Activar stop ajustado |
| CVS ROI 22%, CONSENSO (+4) | ✅ Activa | Confirma, stop estándar |
| PLUG ROI 70%, NEUTRO | ✅ Activa | Mantiene o sube stop |
| Stock ROI 12%, distribución institucional | ✅ Activa stop std | Stop más ajustado, urgencia alta |
| Stock ROI 8%, acumulación, RSI subiendo | SKIP | ❌ No activar — tendencia alcista |

---

---

## Nota — Escalonamiento separado en agente propio

El escalonamiento de salida **no forma parte de este agente**.  
Su espíritu es especulativo (capturar máximo upside antes de caída), opuesto al espíritu
defensivo de Preservation. Se implementa como **`Agente_GainsCapture`** independiente.

Ver diseño completo en: `Doc/gains_capture_design.md`

---

## Distinción de responsabilidades

| | `Agente_ManagerPreservation` | `Agente_GainsCapture` |
|---|---|---|
| Espíritu | **Defensivo** — proteger lo ganado | **Especulativo** — capturar upside |
| Activos | `I` (dividendo, estables) | `N` (volátil, crecimiento) |
| Acción | Coloca STOP trailing | Vende parcialmente en niveles ROI |
| Trigger | ROI >= 10%, cualquier activo estable | ROI >= 50%, solo activos volátiles |
| Claude decide | Dónde poner el stop | Si hay más recorrido o vender ahora |
| Nivel jerárquico | N3 — Decisiones | N3 — Decisiones |

---

## Plan de implementación

### Fase 1 — Preservation Claude (trailing stop dinámico) — IMPLEMENTADO

#### Paso 1 — BD ✅
```sql
ALTER TABLE order_trader ADD COLUMN json_detalle TEXT NULL AFTER hash_id_oportunidad;
```

#### Paso 2 — `Modulos_Mysql.py` ✅
- `select_preservation_context(symbol, account)` — market + sentiment, sin oportunidadesbuysell

#### Paso 3 — `Class_AgentManager.py` ✅ *(reubicado desde `Class_DashBot.py`)*
- `_build_preservation_context()` — indicadores técnicos desde DataHub tiempo real
- `_claude_preservation_eval()` — llama Haiku, retorna stop_sugerido/razon/urgencia
- `_preservation_get_config()` — carga config una vez por vehículo + gate de intervalo (ver sección Debugging abajo)
- `_preservation_run_vehiculo()` — integra Claude con fallback a reglas

#### Paso 4 — `DashMain.py` ✅
- Columna 🤖 en Lista de Ordenes + popup doble-click con análisis Claude

#### Paso 5 — AppTest ✅
- `AppTest/run_preservation_eval.py` — evaluación standalone con _enrich_tecnicos

---

### Fase 2 — Agente_GainsCapture — PENDIENTE

Ver diseño completo en `Doc/gains_capture_design.md`.  
Este agente es independiente: espíritu especulativo, activos `N`, implementación separada.

---

## Debugging — dos gates de intervalo independientes (2026-08-17)

Investigando un supuesto "hang" de `Agente_ManagerPreservation` (BACKLOG #59) se confirmó
que el agente **nunca estuvo trabado** (thread vivo, verificado con `py-spy dump`). Había
dos problemas distintos, mezclados, que generaban la confusión:

### 1. Bug real — logger huérfano sin FileHandler (ya corregido)

`self._preservation_logger` apuntaba a `logging.getLogger("Agente.Preservation")`, un
nombre **nunca registrado** en `Class_debugging.py` (a diferencia de `Agente.Stock`,
`Agente.Crypto`, etc.), por lo tanto sin `RotatingFileHandler` — sus mensajes se perdían
silenciosamente. Afectaba 13 puntos del código: `"N posiciones cargadas"`, evaluación por
posición, colocación/ajuste de STOP y errores. Solo se veían `"config cargada"` y
`"REVISIÓN"` porque esas dos líneas usan `self._log_stock` (registrado, con handler).

**Fix:** [Class_AgentManager.py:54](../../../MyPython/AppOO/Class_AgentManager.py#L54) —
`self._preservation_logger = self._log_stock` (reutiliza el logger ya wireado en vez de
registrar uno nuevo).

#### Revertido el 2026-08-24 — el logger compartido volvió a silenciar todo

Reutilizar `Agente.Stock` resolvió el handler faltante pero dejó a Preservation atado al
nivel de un logger compartido con MarketScreener, DividendStatusScreener y PriceSync. Al
bajar el ruido de esos agentes poniendo `Agente.Stock` en **ERROR** desde el panel de
Debugging, Preservation quedó mudo por completo: todo lo que emite es `warning()`/`info()`.
Diagnóstico del 2026-08-24: el agente había corrido a las 13:49 (confirmado por
`agents_schedule.json` y `preservation_state.json`) sin dejar una sola línea en el log.

El detalle que lo volvió difícil de ver: existe un logger `Preservation` **sí registrado**
en [Class_debugging.py:423](../../../MyPython/AppOO/Class_debugging.py#L423) y visible en el
panel — pero ningún código de Preservation escribía por él, así que subirlo a DEBUG no hacía
nada.

**Fix definitivo (c2bf559):** `self._preservation_logger = logging.getLogger("Preservation")`
y migración de los call-sites que seguían en `_log_stock` (incluidos "config cargada" y
"REVISIÓN", que estaban fuera de `_preservation_run_vehiculo()` y la migración anterior no
había tocado). Los handlers cuelgan del root y propagan, así que el logger escribe a archivo
sin `addHandler` propio. Preservation pasa a tener su propio interruptor en el panel —
patrón `feedback_loggers_por_modulo`.

### 2. Dos intervalos independientes que pueden confundir al debuggear

El decorador `@wait_rate(intervalo, ...)` sobre `Agente_ManagerPreservation` controla
**cada cuánto se llama** a la función — es lo que el botón "Forzar" bypassea. Pero
`_preservation_get_config()` (línea 900-901) calcula un **segundo intervalo,
independiente**, contra `preservation_state.json`:

```python
revisiones_dia = pconfig.get("revisiones_dia", 2)          # ← vive en BD, no en archivo
intervalo_min = ((16 - 9) * 3600) / revisiones_dia          # default: 3.5h dentro de la ventana
```

Si `elapsed < intervalo_min` (medido contra `_last_run_{vehiculo}` en
`preservation_state.json`), la función retorna `time_revision=False` **sin loguear nada**
— ni siquiera "REVISIÓN". "Forzar" en la UI solo bypassea el intervalo del decorador; no
bypassea este segundo gate. Para forzar una revisión completa en debugging hay que además
vaciar `preservation_state.json` (borrar las claves `_last_run_Stock`/`_last_run_Crypto`).

**`revisiones_dia` no es un archivo** — es una clave dentro de `sesion.parameters` (JSON)
en MySQL, bloque `"preservation"`, por vehículo. Se edita desde la pantalla de parámetros
de la app, no a mano.

### 3. Ventana horaria 9-16h — dónde vive y por qué ahí (2026-08-24)

Con la fórmula vieja (`86400 / revisiones_dia`) el reparto era sobre el día corrido y quedaba
anclado a la hora de arranque de la app: con `revisiones_dia=2` daban 12h exactas, o sea
**los mismos dos horarios siempre** (13:49 y 01:49 en la corrida observada) — uno de ellos de
madrugada, con mercado cerrado y precios stale. Ahora `intervalo_min` reparte sobre la franja
9-16h y `_preservation_get_config()` no ejecuta fuera de ella.

**Por qué la ventana NO va en el decorador**, aunque `wait_rate` tenga el parámetro `ventana=`
y el `CLAUDE.md` lo recomiende como regla general: `ventana=` se saltea cuando el agente está
`_overdue` (atraso > `intervalo × 1.5`). De noche el atraso siempre supera ese umbral para
cualquier intervalo razonable, así que el rescate terminaría ejecutando la revisión de
madrugada — justo lo que la ventana intenta evitar.

**A cambio, la guarda interna sí consume el turno del decorador.** Por eso el agente pasó de
`@wait_rate(43200)` a `@wait_rate(1800)`: con 12h, el turno que caía fuera de ventana se
quemaba y reiniciaba el reloj, dejando **1 revisión por día en vez de 2** — y con otro
anclaje de arranque (p. ej. 20:00/08:00) los dos turnos caían fuera y no corría nunca. Con
30 min el decorador solo ofrece turnos baratos (`read_json_tmp` y vuelve) y el espaciado real
lo decide `intervalo_min`.

| Capa | Rol |
|---|---|
| `@wait_rate(1800)` | ofrece turnos frecuentes, sin ventana |
| `_preservation_get_config()` | ventana 9-16h + espaciado por `revisiones_dia` |

**Pendiente de decisión:** la ventana está en hora local (UTC-3) y hardcodeada. En horario de
verano ET equivale a 8:00-15:00 ET — incluye una hora de pre-market y se pierde la última hora
de rueda. Si se busca rueda real conviene `(11, 17)` local.

### Lección para próximas sesiones de debugging
Si el log de Preservation se corta después de "config cargada"/"REVISIÓN" sin avanzar,
antes de sospechar un hang: revisar `preservation_state.json` y `revisiones_dia` — es
frecuente confundir "no tocó" con "está trabado".

---

## Umbrales — por qué NO se pueden comparar con GainsCapture (2026-08-26)

**Decisión: los umbrales quedan como están.** Se evaluó subir `gainInversion` a $150 en ambos
vehículos para que quedara "justo debajo" de los $200/$300 de GainsCapture. Se descartó.

| | Preservation | GainsCapture |
|---|---|---|
| Stock | `roi_minimo` 10% · `gainInversion` **$70** | `min_roi` 20% · `min_ganancia` $200 |
| Crypto | `roi_minimo` 18% · `gainInversion` **$20** | `min_roi` 30% · `min_ganancia` $300 |

`roi_minimo` sale de `sesion.parameters.preservation` (JSON). `gainInversion` es una **columna de la
tabla `sesion`**, no una key del JSON — por eso `_preservation_run_vehiculo` la lee vía
`BDsystem.get_sesion_by_vehiculo()`. Los defaults del código (`100`/`20`) no se usan nunca.

### Los dos "$" no miden lo mismo y no pueden alinearse

`min_ganancia` (GainsCapture) es la ganancia de **la orden**: si vende, se realiza. `gainInversion`
(Preservation) es el `unrealizedpnl` de **la posición entera**, y el STOP toca solo el 33% de los
lotes en ganancia — y puede no ejecutarse nunca.

Descomposición sobre NOMD (46 un. en ganancia, `last` 12.41, 4 lotes entre 9.60 y 11.11):

| | | |
|---|---|---|
| GainsCapture clase 33% | 16 un × (12.41 − 9.60) | **$44.96** |
| 1) qty = 33% de **acciones**, no de lotes | 46 × 0.33 = 15 un | $42.15 · −$2.81 |
| 2) costo **mezclado** 10.19, no el mejor lote 9.60 | 15 × (12.41 − 10.19) | $33.36 · −$8.79 |
| 3) vende al **STOP** 11.42, no al mercado 12.41 | 15 × (11.42 − 10.19) | **$18.47** · −$14.89 |

El punto 3 es irreducible: un STOP vende `correccion_pct` por debajo del precio, siempre — es el
costo del seguro, no un parámetro mal puesto. Sobre el mismo símbolo y el mismo día Preservation
captura ~60% menos que GainsCapture. **Cualquier umbral que los ponga en la misma escala parte de
una equivalencia falsa.**

El punto 1 merece atención aparte: "33%" significa cosas distintas en cada módulo — 33% de **lotes**
en GainsCapture (elige el mejor grupo), 33% de **acciones** en Preservation (barre todos los
ganadores mezclados). Que en NOMD den 16 y 15 es casualidad del reparto.

### Los números empíricos que cerraron la discusión

Cartera Stock del 2026-08-26, 40 posiciones (29 con lotes en ganancia):

```
gate actual   roi_pos ≥10% y upnl ≥ $70   →  0 candidatos
gate actual   roi_pos ≥10% y upnl ≥ $150  →  0
sobre lotes   roi_lot ≥10% y gain ≥ $70   →  1   (NOMD)
sobre lotes   roi_lot ≥10% y gain ≥ $150  →  0
protegido real ≥ $30                      →  0   (máximo de la cartera: NOMD $18.19)
```

`upnl` máximo de toda la cartera: Stock **$81.11**, Crypto **$104.17**. **$150 está por encima de la
mejor posición de los dos vehículos** — no dispararía "poco", no podría disparar. Con $70 ya dispara
cero: `preservation_state.json` no tiene ni un símbolo desde que el agente corre (2026-08-03).

El candidato más cercano es BTG: ROI 22.7%, `unrealizedpnl` $67.27 — lo rechaza el gate por $2.73.

### Hallazgos que salieron de este análisis

1. **El gate mide la posición, el STOP actúa sobre los lotes.** `roi = unrealizedpnl / costobase`
   diluye con los lotes perdedores. Casos reales: TLRY (posición −69.1%) tiene lotes al **+16.4%**;
   CTRM (−38.2%) al **+13.4%**; NNDM (−26.2%) al **+11.2%**. Preservation no los ve nunca, y son
   justo donde la única ganancia que queda vale protegerse.

   **Decisión 2026-08-27: se deja así, a propósito.** No es lo mismo medir la ganancia de la
   posición entera que la de los lotes ganadores, y el gate de posición es el que expresa la
   filosofía del módulo: *¿esta posición vale la pena defenderla?* Un símbolo con la posición
   −69% no se defiende porque le quede un lote verde. Lo que sí se agregó es un segundo gate
   sobre lo que el STOP asegura de verdad (punto 3), para que el que sí pasa el primero no
   emita una orden que protege menos de lo que dice.

2. **`base_limit` era un parámetro muerto — CORREGIDO 2026-08-27.** `base_limit =
   unrealizedpnl * proteccion_base` (0.40) se pasaba a `DataHub.preservation_calc_qty()`, que lo
   ignoraba: la qty salía solo de `pct`. `proteccion_base` no afectaba la cantidad en nada; solo
   viajaba al contexto de Claude y al `json_audit_log`, donde se escribía como
   `"ganancia_protegida_usd"` — un campo que no era la ganancia protegida (en NOMD decía $32
   cuando lo real eran $18, y era positivo en posiciones donde lo real era negativo).

   Hoy `preservation_calc_qty()` ya no recibe `base_limit` y devuelve `(qty, costo_lotes)`. El
   audit escribe la `ganancia_protegida` real. `base_limit` sigue existiendo solo como dato de
   contexto para Claude.

### Cambios de cantidad y de gate (2026-08-27)

**La qty sale de las clases de venta, no de un porcentaje de acciones.** `preservation_calc_qty()`
llama a `maximiza_sell_lotes()` — las mismas clases que usa GainsCapture — sobre
`get_lotesGainLost(opcion="gain")`, que ya filtra ROI > 0 y ordena ROI DESC. Dos motivos:

- El cálculo por acciones promediaba todos los lotes ganadores y bajaba el costo base efectivo.
  En NOMD mezclaba el lote al 11.7% con el del 29.3% y la ganancia asegurada caía de $44 a $33
  antes siquiera del descuento del stop.
- El STOP vende lotes enteros, no fracciones de la posición. Una qty que no coincide con ninguna
  clase deja al agente pisando lotes que ninguna otra vista del sistema muestra.

**Ojo con el "33%":** acá y en GainsCapture significa 33% de los **LOTES** (por conteo, umbral
`0.336`), no de las acciones. Son números distintos salvo coincidencia.

**Piso de un lote entero.** La clase reparte por conteo: con 1-2 lotes en ganancia el `33%` no
alcanza ni un lote y la posición se quedaba sin proteger — incluido BTG, el mejor ROI de la
cartera (2 lotes al 22%). El piso es un lote entero, el de mejor ROI. Nunca una fracción.
Distribución medida en Stock: 11 posiciones sin lotes en ganancia, 13 con 1-2 lotes (la clase
quedaba vacía), 16 con ≥3.

**Gate nuevo sobre la ganancia protegida.** `qty * stop_final - costo_lotes >= gainInversion`.
Ni el gate de ROI ni el `unrealizedpnl` dicen cuánto queda asegurado si el stop se ejecuta —
el primero mide la posición diluida, el segundo supone vender al mercado. Este mide lo que
queda en el bolsillo al precio del stop.

Reusa `gainInversion` a propósito: es la misma pregunta en dólares, aplicada a las dos bases.
Consecuencia a tener presente al tocar la columna: **afloja los dos gates a la vez.**

**Efecto medido (Stock, con el piso aplicado):** desaparecen todos los negativos —
NOMD $18.05→$28.77, SWK −$9.68→$8.06, PFE −$26.28→$4.20, VALE −$9.67→$7.77, CTRM $5.79→$11.69.
Máximo de la cartera: NOMD $28.77, BTG $19.64. Con `gainInversion = 70` el gate nuevo rechaza
todo; el corte real está en $20-25.

**Decisión 2026-08-27: el parámetro se queda en 70.** Que hoy no dispare ninguna orden no es un
defecto a corregir bajando el umbral. El umbral no persigue a la cartera: la cartera lo alcanza a
medida que las posiciones maduran — varias ya están en tendencia alcista en temporalidad semanal y
anual. Bajarlo a $25 para "que funcione" pondría el módulo a proteger ganancias que no vale la pena
defender, y de paso aflojaría también el gate #2 sobre el `unrealizedpnl` de la posición.

**Que Preservation lleve semanas sin emitir una orden es el comportamiento esperado, no un bug.**
Antes de tocar el umbral en una sesión futura, releer este párrafo.

---

## Gate cruzado con GainsCapture (2026-08-29)

Antes de colocar un STOP, Preservation exige `qty + qty_comprometida <= position`, donde
`qty_comprometida` son las acciones del símbolo ya comprometidas en órdenes de venta vivas — las
ponga quien las ponga. Es un **séptimo gate**, después de los seis de § "Umbrales", y el único que
mira fuera de la posición.

Sin él, Preservation y GainsCapture podían comprometer entre los dos hasta el 133% de las acciones
en ganancia del mismo símbolo. Era H5 en
[[resultado-revision-opus-preservation-gainscapture]].

**El STOP propio no cuenta.** Preservation modifica la orden vigente en vez de agregar una nueva, así
que se excluye vía `excluir_order_id`; sin eso el agente se bloquearía a sí mismo cada ciclo y no
volvería a subir su propio stop nunca.

El diseño completo — por qué la fuente es `order_trader` y no el broker, por qué se descartó OCA, y
cómo se comporta en Crypto — vive en [[design-gains-capture]] § "Gate cruzado con Preservation", que
es el agente con las dos puertas de emisión. Acá solo queda la parte que toca a Preservation.

### El STOP fantasma — `sync_broker` (2026-08-29)

Si el gate lee `order_trader`, la tabla tiene que ser el censo completo de lo comprometido. **No lo
era.** `_preservation_run_vehiculo()` manda el STOP a IB y después escribe la fila; cuando IB no
devolvía `order_id`, el camino `[RETRY-FAIL]` / `[STATE-PRESERVED]` no escribía nada a propósito — sin
`order_id` no había con qué identificar la orden.

El resultado era un STOP vivo en el broker e invisible para el sistema: el gate no lo veía y
GainsCapture podía vender las mismas acciones. Justo el escenario que H5 existe para evitar, entrando
por la puerta de atrás.

Ahora la fila **se escribe igual**, con `sync_broker = 'SIN_CONFIRMAR'` y sin `clientOrderId`, y se
loguea `[SIN-CONFIRMAR]` a ERROR. El gate la cuenta como comprometida sin mirar su `status` — ante la
duda bloquea de más, nunca de menos.

No se usó `status` para esto: `status` traduce lo que dijo el broker, y un valor inventado mentiría
sobre lo único que esa columna debe decir. Semánticas distintas, columnas distintas.

`resolve_unconfirmed_orders()` (`Modulos_Mysql.py`, la llama `Agente_SyncOrders` cada 300s) la
resuelve contra IB. No puede cruzar por `clientOrderId` — es el dato que falta, y por eso
`sync_orders_from_ib()` nunca la encuentra —, así que cruza por símbolo y precio de stop, igual que el
reintento `[RETRY-OK]`; `order_trader.price` guarda el límite (`stop * 0.99`) y el stop se reconstruye
antes de comparar. Tras una hora sin aparecer, la marca `HUERFANA` y deja de contar: IB publica la
orden en live orders apenas la acepta, así que no estar significa que nunca entró **o** que ya se
ejecutó. Los dos casos no se distinguen sin consultar ejecuciones y para el gate dan lo mismo — en
ninguno quedan acciones comprometidas hacia adelante. El log es ERROR porque el segundo caso sí
importa para la auditoría.

La semántica de la columna está en `CLAUDE.md` § "Columnas con semántica propia" y repetida en el
COMMENT de MySQL.

**Lo que esto no cubre:** un STOP colocado por fuera del sistema — a mano en TWS, por ejemplo — sigue
sin estar en `order_trader` y el gate no lo ve.

**Commits:** `4803517`, `e258c3d`, `049b18e`.

---

## Pendientes / preguntas abiertas

- [x] **`ClaudeAPIP`** — resuelto: key propia, `BDsystem.get_sesion_by_vehiculo("ClaudeAPIP")` (`Class_AgentManager.py:979`).
- [x] **Claude dice "no activar" con ROI > 10%** — resuelto por diseño: las reglas fijas son el piso.
      Claude solo puede **subir** el stop (`stop_final = max(stop_final, stop_claude)`), nunca cancelarlo.
- [ ] rsi_d/macd en tiempo real desde `booktrading.indicadores` — el campo se graba al momento de la
      transacción y puede estar desactualizado.
- [ ] ¿`Agente_SyncOrders` maneja órdenes STOP o solo BUY/SELL?
