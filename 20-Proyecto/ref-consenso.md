---
tipo: ref
modulo: consenso
version: 1.0
fecha: 2026-07-01
status: activo
---

# Consenso Score

> Detalle técnico del pipeline: [[spec-consenso]] (archivo histórico, no borrar)
> Flujo de sentimiento: [[design-sentimiento]]
> Señal IA que pasa por el gate: [[ref-oportunidades]]

---

## 1. Para qué sirve

Cada activo recibe hasta 8 votos de fuentes independientes. El resultado es un **tag** (UNANIME → SALIDA) que resume si los fundamentos apoyan comprar, mantener o salir.

El tag también actúa de **filtro** antes de enviar alertas a Telegram — evita que el modelo IA se confirme a sí mismo.

---

## 2. De dónde vienen los datos

| Paso | Qué hace | Frecuencia |
|------|----------|------------|
| NASDAQ discovery | Detecta símbolos con dividendo declarado | Semanal |
| Yahoo Finance | Precio, fundamentals, recomendación analistas | Diario |
| EDGAR 13F | Posiciones de ~9K fondos institucionales | Diario |
| ConvergIA Sentimiento | Noticias por símbolo → patrón comportamiento | 1×/día |

Todo converge en la tabla `market` de la BD.

---

## 3. Los 8 votos

Cada voto emite **+1** favorable · **0** neutral · **−1** desfavorable · `—` abstiene (no cuenta en el total).

| # | Nombre | Qué mide | +1 si… | −1 si… | Abstiene si… |
|---|--------|----------|--------|--------|--------------|
| 1 | **Net** | Balance compras vs ventas en 13F | top 33% de cartera | bot 33% | sin datos 13F |
| 2 | **Opt** | Proporción CALL vs PUT en 13F | CALL > 60% del total | CALL < 40% | sin opciones |
| 3 | **Flujo** | Fondos que entraron vs salieron este trimestre | top 33% flujo neto | bot 33% | sin datos 13F |
| 4 | **Ana** | Recomendación de analistas Wall Street | buy / strong_buy | sell / strong_sell | sin analistas |
| 5 | **Mod** ⚠ | Señal del modelo IA propio | señal BUY | señal SELL | sin señal |
| 6 | **Val** | Valuación del activo | I — infravalorado | S — sobrevalorado | ETF / fondo (X, T) |
| 7 | **Cob** | Cuántos fondos cubren el activo | ≥ 20 fondos | < 5 fondos | — |
| 8 | **Sent** | Patrón de noticias últimos 7 días | acumulacion / inflexion+ | distribucion | sin datos |

⚠ **Mod no cuenta en el gate Telegram** — si se incluyera, la señal IA se confirmaría sola empujando el tag hacia TENDENCIA con solo 1 voto extra.

---

## 4. Tags de resultado

```
pct = suma de votos activos / cantidad de votos activos
```

| Tag | Condición | Qué significa |
|-----|-----------|---------------|
| ★ **UNANIME** | todos +1 | Máxima convicción — acumular |
| ▲ **CONSENSO** | pct ≥ 0.60 | Comprar / aumentar posición |
| ↗ **TENDENCIA** | pct ≥ 0.20 | Mantener / pequeños aumentos |
| → **NEUTRO** | pct > −0.20 | Observar sin movimiento |
| ↘ **ALERTA** | pct > −0.60 | Reducir / revisar tesis |
| ▼ **SALIDA** | pct ≤ −0.60 | Salir o no entrar |

Ejemplo: `↗ TENDENCIA +4/6` → 4 votos positivos sobre 6 activos (2 abstuvieron).

---

## 5. Gate Telegram

Antes de enviar una alerta BUY o SELL a Telegram, el sistema verifica el tag del activo usando los **6 votos sin Mod**.

| Señal | Tags que permiten | Tags que bloquean |
|-------|-------------------|-------------------|
| **BUY** | UNANIME · CONSENSO · TENDENCIA | NEUTRO · ALERTA · SALIDA |
| **SELL** | ALERTA · SALIDA | UNANIME · CONSENSO · TENDENCIA · NEUTRO |

La señal que pasa (o no) por este filtro viene del modelo IA → ver [[ref-oportunidades]].

---

## 6. Voto Sent — cómo se calcula

Combina dos tablas: `market_sentiment` (lecturas 3×/día) y `market_sentiment_analysis` (patrón diario generado por Haiku).

| Situación | Fuente | Voto |
|-----------|--------|------|
| Patrón de hoy: `acumulacion` | market_sentiment_analysis | +1 |
| Patrón de hoy: `distribucion` | market_sentiment_analysis | −1 |
| Patrón de hoy: `neutro` | market_sentiment_analysis | 0 |
| Patrón de hoy: `inflexion` y última noticia positiva | ambas | +1 |
| Patrón de hoy: `inflexion` y última noticia negativa | ambas | 0 (abstención) |
| Sin análisis de hoy (agente no corrió) | market_sentiment directo | +1 / 0 / −1 |
| Sin ningún dato | — | abstiene |

Detalle completo del pipeline de sentimiento → [[design-sentimiento]]

---

## 7. Columnas del Screener

| Columna | Qué te dice |
|---------|-------------|
| **Last** | Precio actual |
| **Rotación** | Proxy de liquidez institucional: floatShares / volume |
| **Inst Score** | Score compuesto: 40% ownership + 20% cobertura + 20% dirección + 20% flujo neto |
| **Inst %** | Qué % del float libre tienen los fondos |
| **13F Inst** | Cuántos fondos declaran posición en este activo |
| **13F Buy%** | % de fondos que compraron o abrieron posición este trimestre |
| **13F Sell%** | % de fondos que redujeron o cerraron |
| **CALL / PUT** | Acciones en opciones CALL y PUT declaradas en 13F |
| **ΔCall / ΔPut** | Cambio trimestral — CALL creciente = acumulación silenciosa |
| **+Nuevos** | Fondos que entraron por primera vez este trimestre |
| **−Salidas** | Fondos que salieron completamente (no aparecen en Q actual) |
| **Flujo** | (Nuevos − Salidas) / total fondos, rango [−1, +1] |
| **Inst Señal** | Resumen: ACOMPAÑAR / MANTENER / REVISAR |
| **Analistas** | Recomendación Wall Street + media numérica |
| **IA Signal** | Señal del modelo propio: BUY / SELL / — (solo visual, no vota en gate) |
| **Sent** | Patrón de noticias: +1 / 0 / −1 |
| **Consenso** | Tag final con suma de votos activos |

---

## 8. Archivos clave

| Archivo | Qué hace |
|---------|----------|
| `Class_Screener.py` | Popup Consenso, Screener, `refresh_consenso_tags()` |
| `Class_DashBot.py` | `Agente_ConsensoCache` (refresca cada 5 min), gate Telegram |
| `Class_AgentManager.py` | `Agente_Sentimiento`, `Agente_InterpreteSentimiento` |
| `ConvergIA/Scanner_Sentimiento.py` | Descarga noticias → clasifica con Haiku → `market_sentiment` |
| `ConvergIA/Interprete_Sentimiento.py` | Lee historial → detecta patrón → `market_sentiment_analysis` |
| `ConvergIA/ThemeMapper.py` | `voto_tech_alignment()` — convierte patrón a +1 / 0 / −1 |
| `Modulos_Mysql.py` → `MarketScreen` | Lectura/escritura de sentimiento, `load_sentiment_features()` |

---

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-01 | Creado — reemplaza spec-consenso como referencia de lectura |
