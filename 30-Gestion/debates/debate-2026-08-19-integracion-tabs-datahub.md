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
> - ¿Qué tan lejos está `Sell IA`/`Buy IA` de una decisión autónoma real (Fase 3/4) vs. hoy, que
>   solo muestra candidatos para revisión humana?
> - ¿`Symbol Events`/`Alertas` es el lugar natural para enganchar el gate de `salida_emergencia`
>   (idea dejada en `design-agente-ia.md`, tabla `variablesplan`)?
> - ¿Qué falta conectar entre estas 6 pestañas para que el flujo Capa 2→3→4→5 (datos → señales →
>   decisión → trazabilidad) sea trazable end-to-end para un símbolo, no solo observable por partes?

---

## 🖥️ Perspectiva Code

*Lente: implementación, seguridad, convenciones, esfuerzo técnico, riesgos de ejecución*

[Pendiente — completar en próxima sesión]

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
