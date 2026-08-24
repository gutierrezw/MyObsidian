# Preservation Dinámico con Claude — Diseño

**Estado:** Diseño inicial — pendiente implementar  
**Fecha:** 2026-06-02  
**Prioridad:** Alta

**Ver también:**
- [[design-gains-capture.md]] — agente complementario (especulativo, captura upside)
- [[analysis-agent-history-table.md]] — tabla unificada para aprender de ambos agentes

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

## Pendientes / preguntas abiertas

- [ ] ¿Nueva key `ClaudeAPIP` en tabla sesion o reusar `ClaudeAPIC`?
- [ ] ¿Cómo obtener rsi_d/macd desde booktrading.indicadores en tiempo real? (el campo se graba al momento de la transacción, puede estar desactualizado)
- [ ] ¿El `Agente_SyncOrders` ya maneja órdenes STOP de preservation o solo BUY/SELL?
- [ ] Definir qué hacer cuando Claude dice "no activar" pero ROI supera 10% → ¿reglas fijas igual?
