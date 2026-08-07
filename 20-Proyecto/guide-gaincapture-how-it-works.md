---
name: guide-gaincapture-how-it-works
description: Guía paso a paso — cómo funciona GainsCapture sin código
metadata:
  type: guide
  created: 2026-08-07
  status: EXPLICATIVO
---

# Guía: ¿Cómo Funciona GainsCapture? (paso a paso)

**Objetivo:** Capturar ganancias explosivas vendiendo **parcialmente** en niveles de ROI, en lugar de esperar a que caiga todo.

**Espíritu:** Especulativo. Operamos sobre activos volátiles que pueden hacer +50%, +100%, +150% en semanas.

**Complemento:** [[design-preservation.md]] protege activos estables. GainsCapture vende activos de crecimiento.

---

## El Problema que Resuelve

Tienes SKLZ comprada a $11.18 con 136 acciones.

### Sin GainsCapture (hoy)
- Precio sube a $16.77 (+50% ROI = +$175/acc)
- Sigue subiendo a $22.36 (+100% ROI = +$1,264 total)
- Luego cae a $9 (pérdida total)
- Resultado: esperaste, ganaste nada

### Con GainsCapture
- Precio sube a $16.77 (+50% ROI)
  - ✅ Vende 25% de la posición (34 acc) @ $16.69
  - 💰 Ganancia asegurada: ~$175
- Sigue subiendo a $22.36 (+100% ROI)
  - ✅ Vende 50% más (51 acc) @ $22.25
  - 💰 Ganancia asegurada: ~$562
- Sigue subiendo a $27.95 (+150% ROI)
  - ✅ Vende 25% final (25 acc) @ $27.81
  - 💰 Ganancia asegurada: ~$418
- Cae a $9
  - Las 30 acc restantes pierden, pero capturaste $1,155 en el camino

**Resultado:** GainsCapture aseguró ganancias en cada escalón, Preservation protege lo que queda.

---

## ¿Quién Puede Usar GainsCapture?

### Criterio 1: Categoría de Activo
Solo activos con `categoriaActivo='N'` en la tabla `market`:
- `N` = **Sin dividendos** = crecimiento puro, especulativo, volátil
- NO `I` (Infravalorado, estables) — esos usa Preservation
- NO `S` (Sobrevalorado, evitar)

**Ejemplos N:** SKLZ, PLUG, CRNT, CHPT (fintech, tech especulativa)

### Criterio 2: ROI Alcanzado
```
Nivel 1: ROI >= 50%  →  Vende 25% de la posición
Nivel 2: ROI >= 100% →  Vende 50% de lo que queda
Nivel 3: ROI >= 150% →  Vende 25% final
```

Todos estos umbrales son configurables en `sesion.parameters["gains_capture"]["niveles"]`.

---

## El Flujo Paso a Paso

### Paso 1: GainsCapture Detecta Oportunidad
**Cada 30 minutos** (o intervalo configurado), el agente revisa:

1. ¿Hay activos en ganancia?
2. ¿Alguno tiene `categoriaActivo='N'`?
3. ¿Su ROI alcanzó algún nivel (50%, 100%, 150%)?
4. ¿Ese nivel no fue ejecutado aún?

Si cumple todo → **entra a evaluación**.

---

### Paso 2: Claude Valida Momentum

Antes de vender, Claude analiza indicadores técnicos en tiempo real:

| Indicador | "Esperar más" | "Vender ahora" |
|-----------|---------------|----------------|
| **RSI diario** | < 65 (momentum sano) | > 75 (sobrecompra) |
| **RSI semanal** | < 65 | > 72 |
| **Precio vs EMA50** | Acelerando por encima | Tocando/debajo (fuerza se agota) |
| **Precio vs EMA200** | Lejos por encima | Cerca/debajo (tendencia débil) |

**Decisión de Claude:**
- ✅ `"ejecutar": true` → vende ahora, es buen timing
- ❌ `"ejecutar": false` → espera al próximo ciclo, hay más recorrido

**Razón que genera Claude:**
- "RSI diario sobrecomprado (78), señal de toma de ganancias"
- "EMA50 y EMA200 alineadas alcistas, pero EMA200 débil"
- etc.

---

### Paso 3: Dos Modos de Operación

Depende del modo que hayas elegido en el panel Agentes:

#### Modo 🟢 **Automático**
- Claude decide → se ejecuta directamente
- Orden LIMIT enviada a IB al precio de mercado × 0.99 (0.5% descuento)
- Notificación Telegram: "✅ SKLZ: vendido 34 acc @ $16.69"

#### Modo 🟠 **Autorizado**
- Claude prepara la propuesta
- Telegram te envía:
  ```
  📈 GainsCapture — SKLZ
  Nivel ROI 50% alcanzado
  Vender 34 acc LMT $19.50
  RSI_d=78 sobrecomprado — urgencia: alta
  
  /ok_SKLZ  |  /no_SKLZ
  ```
- Tú confirmas con `/ok_SKLZ` o rechazas con `/no_SKLZ`
- **Sin respuesta en 30 min** → propuesta cancelada (conservador)

---

### Paso 4: Orden Colocada

**Qué se envía a IB:**

| Campo | Valor |
|-------|-------|
| Tipo | SELL (venta) |
| Orden | LIMIT |
| Precio | last × 0.995 (0.5% descuento) |
| Cantidad | calculada (% de la posición) |
| TIF | GTC (Good Till Cancelled) |

**Por qué LIMIT, no MARKET:**
- MARKET ejecuta rápido pero regala precio
- LIMIT ejecuta en buen precio o espera
- En mercados activos, LIMIT × 0.99 se ejecuta igual, pero proteges si hay gap down

---

### Paso 5: Orden en Espera

La orden espera llenar. Mientras:
- **Preservation** sigue corriendo en paralelo (protege con STOP si la posición baja)
- GainsCapture espera al próximo ciclo
- Si el próximo nivel se alcanza, se coloca otra orden LIMIT sin tocar la primera

**Ejemplo SKLZ:**
- Nivel 50% @ $16.77: LMT SELL 34 acc @ $16.69 (esperando fill)
- Precio sigue subiendo a $22.36
- Nivel 100%: LMT SELL 51 acc @ $22.25 (nueva orden, la primera sigue abierta)
- Las dos órdenes pueden llenarse ambas, solo la primera, o ninguna — depende del mercado

---

### Paso 6: Order Fill (Venta Realizada)

Cuando IB ejecuta la orden:
- **Agente_SyncOrders** detecta que fue FILLED
- Actualiza estado en `gains_capture_state.json`
- Registra la venta en `agent_history` (tabla de auditoría)
- Guarda datos técnicos y decisión Claude para **análisis posterior**

---

## Estados Posibles de una Orden

```
normal
  ↓
  ROI alcanzado + Claude dice ejecutar
    ↓
ESCALON_PENDIENTE (orden en IB esperando fill)
  ├─→ FILLED → venta realizada ✅
  ├─→ CANCELLED (usuario o precio cambió)
  └─→ EXPIRED (tiempoagotado, GTC + límite del broker)
```

---

## Diferencia con Preservation

| Aspecto | **GainsCapture** | **Preservation** |
|---------|------------------|-----------------|
| **Espíritu** | Especulativo | Defensivo |
| **Activos** | `N` (volátiles, sin dividendos) | `I` (estables, con dividendo) |
| **Acción** | **VENDE** parcialmente | **PROTEGE** con STOP |
| **Cuándo** | ROI >= 50% (configurable) | ROI >= 10% (siempre) |
| **Decisor** | Claude valida momentum | Claude afina nivel del stop |
| **Objetivo** | Capturar upside antes de caída | Limitar pérdidas |

**Pueden convivir:**
- SKLZ tiene `categoriaActivo='N'` → GainsCapture vende 25% a $16.77
- SKLZ también tiene ROI >= 10% → Preservation coloca STOP protector en paralelo
- Si precio baja abruptamente antes de vender → STOP de Preservation se toca primero
- Si sube como se espera → GainsCapture vende en escalones

---

## Cómo Aprender de GainsCapture

Una vez en producción, cada orden genera un registro en [[analysis-agent-history-table.md]]:

**Preguntas que puedes responder después:**

1. **¿Vendimos en buen timing?**
   - Calculamos `precio_max_posterior` (qué precio alcanzó después)
   - Comparamos con `precio_fill` (a qué vendimos)
   - `pct_captura` = % del máximo que capturaste
   - Ejemplo: vendimos a $16.69, luego subió a $22.36
     - Ganancia teórica máxima: $5.67/acc
     - Ganancia real: $1.51/acc
     - pct_captura = 26.6% (vendimos temprano pero seguro)

2. **¿RSI > 75 es buen predictor de peak?**
   - Agrupamos todas las órdenes por RSI al momento de venta
   - Vemos cuáles tuvieron pct_captura > 60% (buen acierto)
   - Si RSI=78 → 80% de acierto, entonces sí es buen indicador

3. **¿Modo automático o autorizado erra más?**
   - Comparamos efectividad de ambos modos
   - Detectamos si tus /ok_SYMBOL tienen mejor timing que el automático

4. **¿Los niveles de venta (50/100/150%) son óptimos?**
   - Ajustar si muchas órdenes se quedan sin llenar
   - Subir a 60/110/160% si el mercado es muy explosivo

---

## Configuración Mínima

En `sesion.parameters["Stock"]`:

```json
{
  "gains_capture": {
    "niveles": [
      {"roi": 0.50, "vender_pct": 0.25},
      {"roi": 1.00, "vender_pct": 0.50},
      {"roi": 1.50, "vender_pct": 0.25}
    ],
    "modo": "automatico"
  }
}
```

Sin esta sección → agente deshabilitado sin error.

---

## Casos Reales

### SKLZ: El caso ideal
- Comprado a $11.18, 136 acc
- ROI 50% @ $16.77 → vende 34 acc → ganancia: +$175/total
- ROI 100% @ $22.36 → vende 51 acc → ganancia: +$562/total
- ROI 150% @ $27.95 → vende 25 acc → ganancia: +$418/total
- **Total asegurado: $1,155**
- Posición restante (30 acc) protegida por Preservation
- Si cae a $9 → pierdes solo en las 30 restantes, no en las 110 vendidas

### PLUG: El falso breakout
- Comprado a $5, 200 acc
- ROI 50% @ $7.50 → vende 50 acc → ganancia: +$250
- Precio cae a $6 (movimiento de 1 día)
- Nivel 100% nunca se alcanza
- **Resultado:** capturaste +$250 sin esperar, mientras el mercado duda

### CRNT: El momentum débil
- Comprado a $15, 50 acc
- ROI 50% @ $22.50 (Claude dice RSI=42, aún hay recorrido)
- Espera el próximo ciclo
- Precio se mantiene → nivel 50% nunca se completa
- Pero Preservation coloca STOP y al menos proteges ganancias parciales

---

## Checklist Antes de Activar

- [ ] ¿Tienes activos con `categoriaActivo='N'`?
- [ ] ¿Está configurado `gains_capture` en sesion.parameters?
- [ ] ¿Prefieres modo "automatico" o "autorizado"?
- [ ] ¿Los niveles (50/100/150%) tienen sentido para tu cartera?
- [ ] ¿Entiendes que puede vender antes del pico (eso es normal)?
- [ ] ¿Tiene Preservation activo para proteger lo que no se vende?

---

## Links relacionados

- [[design-gains-capture.md]] — especificación técnica completa
- [[design-preservation.md]] — cómo protege Preservation en paralelo
- [[analysis-agent-history-table.md]] — cómo analizar efectividad posterior
