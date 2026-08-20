---
tipo: debate
estado: abierto
---

# Debate — Integración BuySell/Rebalanceo/Sell IA/Buy IA/Symbol Events/Alertas

**Fecha:** 2026-08-19
**Propuesto por:** usuario
**Estado:** Abierto

---

## Tema / Pregunta

> El usuario, viendo el panel `DataHub - Símbolos` en vivo (screenshot BNBUSDT con `activos`,
> `datos_tecnicos`, `costobase` poblados), señaló: "aca es donde el sistema empieza a usar lo que
> venimos construyendo" — refiriéndose a que las pestañas `BuySell`, `Rebalanceo`, `Sell IA`,
> `Buy IA`, `Symbol Events` y `Alertas` comparten el mismo `DataHub.info[symbol]`, y que "comienza
> la gran integración". Contexto ampliado en [[design-agente-ia]] (sección "Observación en vivo").
>
> No es una decisión concreta todavía — el usuario está "lanzando ideas" para esta discusión.
> Preguntas abiertas a desarrollar la próxima sesión:
> - ¿Qué tan lejos está `Agente_ClaudeIA` de ejecución automática (Fase 3) vs. hoy, que solo
>   muestra candidatos para revisión humana? (nota: `Sell IA`/`Buy IA` en la UI es un modelo ML
>   distinto — ver hallazgo en Perspectiva Code)
> - ¿`Symbol Events`/`Alertas` es el lugar natural para enganchar el gate de `salida_emergencia`
>   (idea dejada en `design-agente-ia.md`, tabla `variablesplan`)?
> - ¿Qué falta conectar entre estas 6 pestañas para que el flujo Capa 2→3→4→5 (datos → señales →
>   decisión → trazabilidad) sea trazable end-to-end para un símbolo, no solo observable por partes?

---

## 🖥️ Perspectiva Code

*Lente: implementación, seguridad, convenciones, esfuerzo técnico, riesgos de ejecución*

**Verificado en código (2026-08-20)** — la premisa "todas leen el mismo `DataHub.info[symbol]`" es
solo parcialmente cierta. Hay dos naturalezas de dato distintas conviviendo en las 6 pestañas
(`Class_SystemStatus.py`):

| Pestaña | Fuente real | Naturaleza |
|---|---|---|
| BuySell (`manager_buysell_system`, L2031) | `DataHub.manager_buysell` | Vivo, en memoria |
| Rebalanceo (`rebalanceo_system`, L2278) | `DataHub.rebalanceo[vehiculo]` | Vivo, en memoria |
| Sell IA (`self.modeloia`, L3519) | Modelo ML entrenado + `oportunidades_sell()` | Mixto |
| Buy IA (`self.modeloiabuy`, L4100) | Modelo ML entrenado + `oportunidades_buy()` | Mixto |
| Symbol Events (`symbol_events_system`, L4275) | tabla `symbol_decision_history` | **Persistido**, por símbolo |
| Alertas (`alertas_system`, L3006) | tabla `system_alerts` | **Persistido**, cuenta completa (no por símbolo) |

**Hallazgo que reencuadra el debate:** "Sell IA"/"Buy IA" en la UI actual **no es** el
`Agente_ClaudeIA` de Capa 4 (`design-agente-ia.md`) — es un modelo ML clásico entrenado sobre
histórico de trades. Son dos sistemas de IA distintos con nombres parecidos; conviene desambiguar
antes de decidir alcance de la integración.

**Fricción real no es técnica, es de ciclo de vida:** 4 pestañas son snapshots efímeros
(se pierden al recargar) y 2 son historial persistido. Para trazabilidad end-to-end por símbolo
faltaría:
1. **Puente vivo → persistido** — hoy solo `Agente_ClaudeIA` escribe en `symbol_decision_history`
   (vía `ia_trace`); BuySell/Rebalanceo/Buy IA/Sell IA no dejan rastro histórico de sus oportunidades.
2. **Selector de símbolo compartido** — cada pestaña tiene su propio combo independiente; falta un
   selector único que refresque las 6 a la vez (patrón ya existe en `symbol_events_system`).
3. **`salida_emergencia` como gate real** — encaja con Alertas como disparador natural cuando el
   gate bloquea una decisión (idea ya registrada en `design-agente-ia.md`, sección "Idea — uso de
   `variablesplan` en modo autónomo").

---

## 🖱️ Perspectiva Desktop

*Lente: valor estratégico, experiencia de uso, ROI, simplicidad, visión de largo plazo*

[Pendiente — completar en próxima sesión]

---

## 🎯 Síntesis

**Consenso:** No — recién abierto
**Decisión:** Pendiente
**Condiciones acordadas:** Pendiente
**Acción concreta:** Pendiente

---

## Historial

| Fecha | Quién | Acción |
|-------|-------|--------|
| 2026-08-19 | usuario + Code | Abierto tras observación en vivo del panel DataHub — capturado en design-agente-ia.md |
| 2026-08-19 | usuario + Code | Caso concreto derivado: panel KPI (`DashMain.py`) tenía topes en duro (Leverage 30%, Deuda = límite del broker) desconectados de `agente_ia.deuda_max_pct` ya definido en Restricciones de cartera. Conectado Leverage/Deuda a `deuda_max_pct`. `leverage_max` (múltiplo) y `risk_real_max` (sin uso en el código) quedaron pendientes — unidades ambiguas, no se fuerza conversión sin confirmar significado. Detalle en design-agente-ia.md. |
| 2026-08-20 | Code | Perspectiva Code completada: verificado en código que las 6 pestañas NO comparten una única fuente viva — 4 son snapshots en memoria y 2 (Symbol Events, Alertas) son historial persistido en BD. Hallazgo clave: "Sell IA"/"Buy IA" en la UI es un modelo ML entrenado, distinto del `Agente_ClaudeIA` (Capa 4). Fricción real es de ciclo de vida (vivo vs. persistido), no técnica — 3 brechas identificadas: puente vivo→persistido, selector de símbolo compartido, `salida_emergencia` como gate. Falta Perspectiva Desktop y Síntesis. |
| 2026-08-20 | usuario + Code | Corregida la pregunta original (Tema/Pregunta y `design-agente-ia.md` líneas 452-462): no había ambigüedad entre `ModeloOportunidadesSell`/`Buy` (bien diferenciados entre sí) — la confusión era llamar "Sell IA/Buy IA" a la pregunta de autonomía, cuando esa pregunta apunta a `Agente_ClaudeIA` (Fase 3, ejecución automática), un sistema distinto. Pregunta reformulada: "¿qué tan lejos está `Agente_ClaudeIA` de ejecución automática (Fase 3)?" — queda como próximo punto a desarrollar. |
| 2026-08-20 | Code | Mapeado cómo funciona Fase 1 hoy (`Class_DashBot.py`): `Agente_ClaudeIA()` corre cada 24h, arma contexto (`_armar_contexto_ia`: portfolio, candidatos, rebalanceo, oport buy/sell, FCI), llama a Claude (`_claude_ia_eval`), inserta en `ia_trace`. Nunca ejecuta — según `agente_ia.modo` (OBSERVACION/SUPERVISADO/AUTONOMO) como máximo notifica por Telegram con botones, y el botón "Ejecutar" hoy es manual (línea 1587: "AUTONOMO pendiente"). **Hallazgo sobre contexto del plan:** `agente_ia.plan` (meta_capital/meta_año/ingreso_pasivo_pct/mision) SÍ llega al prompt hoy (línea 851-859 de `Class_DashBot.py`). Las tablas `plan`/`variablesplan` (objetivo narrativo, riesgos, `salida_emergencia`) NO llegan — `_armar_contexto_ia()` nunca llama `select_plan()`/`select_variablesplan()`. Propuesta del usuario: sumar ese contexto (riesgos + salida_emergencia) al prompt para que la IA decida en función del plan completo, no solo de las 4 métricas de `agente_ia.plan`. Queda como próximo cambio concreto a implementar. |
