---
name: ref-oportunidades
description: Sistema BUY/SELL — reglas, señales, umbrales, score híbrido TOP10 y ciclo IA. Referencia principal.
metadata:
  tipo: ref
  modulo: oportunidades
  version: 2.0
  fecha: 2026-06-29
  status: activo
---

# Sistema de Oportunidades BUY / SELL

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

## Historial
| Versión | Fecha | Cambio |
|---------|-------|--------|
| 2.0 | 2026-06-29 | Unificación buy+sell, score híbrido TOP10, elimina duplicados |
| 1.0 | 2026-04 | Documentos separados ref-regla-buy, ref-modelo-buy, ref-modelo-sell |
