---
tipo: prompt
estado: usado — hallazgos incorporados a design-agente-ia.md
---

# Prompt — revisión independiente (Opus) de la propuesta riesgos/salida_emergencia

**Fecha:** 2026-08-20
**Propósito:** pedido explícito del usuario — antes de implementar, pedirle a Opus en una sesión
nueva (sin el historial de quien diseñó el plan) que evalúe críticamente la propuesta y señale
cabos sueltos. Ver [[design-agente-ia]] (sección "Plan de trabajo — integración riesgos/salida_emergencia")
y [[debate-2026-08-19-integracion-tabs-datahub]] para el contexto completo.

**Resultado:** la revisión encontró 2 errores de hecho verificados en código (el agente corre
solo en Stock, no en 3 vehículos; la justificación de Capa 1 estaba invertida — scope por
`idcuenta` aísla, no comparte) y 1 problema de diseño no visto (`variablesplan.ditem` trunca a
50 chars, insuficiente para un criterio de decisión). Hallazgos completos incorporados a
`design-agente-ia.md`. Ver respuesta íntegra al final de este documento.

**Cómo usarlo:** copiar el contenido de la sección "Prompt" completo en una sesión nueva.

---

## Prompt

# Prompt para sesión de revisión independiente (Opus)

> Instrucciones de uso: pegar este prompt completo en una sesión NUEVA (sin el historial de la
> sesión que diseñó la propuesta), idealmente corriendo con el modelo Opus. El objetivo es una
> segunda opinión sin sesgo de quien escribió el plan original.

---

## Rol

Actuás como revisor técnico senior, independiente del equipo que diseñó esta propuesta, con mandato
explícito de dar una **opinión crítica**, no una validación. Tu default es cuestionar, no confirmar:
asumí que el plan tiene al menos un punto débil y tu trabajo es encontrarlo, aunque tengas que
forzar la lectura contra el plan en vez de a favor. No es un ejercicio de cortesía ni de "está bien
pensado, sigan así" — si terminás sin objeciones reales, releé el plan buscando qué no cuestionaste.
Cuando algo esté genuinamente bien resuelto decilo en una línea y avanzá; no rellenes con elogios.

## Contexto del proyecto

AppOO es una plataforma de automatización de inversiones (Tkinter + MySQL + brokers IB/Binance) con
visión de escalar de herramienta personal a servicio multi-cliente. Arquitectura de 6 capas para el
agente de IA de decisiones (`design-agente-ia.md`):

- **Capa 1 — Configuración**: `sesion.parameters.agente_ia` (JSON por vehículo) + tablas MySQL
  `plan`/`variablesplan` (por cuenta).
- **Capa 2 — Datos**: `DataHub`, hub en memoria que alimenta paneles UI en vivo.
- **Capa 3 — Señales**: Consenso Score (7 votos ponderados, determinístico), `inst_score`, Sentiment.
- **Capa 4 — Decisión**: `Agente_ClaudeIA` (`Class_DashBot.py`), corre cada 24h, arma un prompt con
  contexto (portfolio, oportunidades, rebalanceo, FCI) y llama a Claude para decidir BUY/SELL/HOLD.
- **Capa 5 — Trazabilidad**: tabla `ia_trace` + `symbol_decision_history`, panel "IA Trace" en la UI.
- **Capa 6 — Meta-monitoreo**: `ia_mejoras`, todavía sin construir (backlog propio del agente).

Roadmap declarado de 4 fases: Fase 1 "Ojos" (✅ completa — el agente observa y decide, no ejecuta),
Fase 2 "Voz" (🟡 parcial — ya notifica por Telegram con botones), Fase 3 "Manos" (❌ sin conectar
decisión→ejecución de orden real), Fase 4 "Autonomía supervisada" (❌ sin fecha).

## La propuesta a evaluar

Hoy `_armar_contexto_ia()` arma un prompt de 14 bloques (misión, fecha, portfolio, gains-candidates,
rebalanceo, ranking rebalanceo, oport BUY, oport SELL, candidatos externos, contexto FCI, rotación
FCI, límites, instrucciones de cruce, formato JSON). Ese contexto viene de `agente_ia.plan` (4 campos
numéricos: meta_capital, meta_año, ingreso_pasivo_pct, mision) y de datos de mercado/portfolio — pero
**nunca lee** las tablas `plan`/`variablesplan`, donde el usuario registra riesgos y criterios de
salida de emergencia en lenguaje cualitativo.

Propuesta: sumar 2 bloques nuevos al prompt (riesgos + salida_emergencia), leyendo
`select_variablesplan(idcuenta)` y filtrando por `tipo` en Python (la función no filtra por tipo en
su firma). El prompt pasaría de 14 a 16 bloques.

Antes de escribir código, se validó la propuesta contra las 6 capas (no solo Capa 4, que es donde
nació el pedido) — pedido explícito del usuario, con el argumento de que ya pasó antes en este
proyecto que un dato configurado en la UI (`agente_ia.deuda_max_pct`, restricciones de cartera)
quedaba sin consumidor real en el código durante meses. El resultado de esa validación fue:

| Capa | Decisión tomada | Justificación |
|---|---|---|
| **1 — Configuración** | Sin cambios. `variablesplan` ya vive en MySQL scoped por `idcuenta`, compartida automáticamente entre Stock/Crypto/BotCrypto — a diferencia de `agente_ia.plan` (JSON en `sesion.parameters`, uno por vehículo, necesitó `_sync_plan_restricciones()` para replicarse). | `_edit_riesgos()` en `Class_gestion.py:967` guarda por `idcuenta`, no por vehículo. |
| **2 — Datos** | Sin cambios. Lectura directa a MySQL en cada corrida de `_armar_contexto_ia()`, sin pasar por `DataHub`. | El agente corre cada 24h (`@wait_rate(86400,...)`), no es un ciclo frecuente que justifique cachear en `DataHub`. |
| **3 — Señales** | Riesgos/salida_emergencia NO alimentan el Consenso Score. | Son cualitativos, solo interpretables por Claude en contexto; el Consenso Score es cuantitativo y determinístico — mezclarlos degradaría su reproducibilidad. |
| **4 — Decisión** | Cambio real: 2 bloques nuevos en el prompt entre el bloque 04 (gains-candidates) y el 05 (rebalanceo). También actualizar el bloque 13 (instrucciones de cruce) para que Claude trate riesgos/salida como criterio de corte de la decisión, no solo contexto informativo. | — |
| **5 — Trazabilidad** | Cambio real: extender el dict `gates_ok` (columna JSON ya existente en `ia_trace`, sin migración) con `riesgos_considerados` / `salida_emergencia_activa` al insertar cada trace. | `insert_trace()` en `Modulos_Mysql.py:7742` ya tiene `gates_ok` como JSON. |
| **6 — Meta-monitoreo** | No aplica todavía. | `ia_mejoras` sigue sin construir. |

Orden de ejecución propuesto: 1) implementar Capa 4 + Capa 5 en el mismo commit, 2) probar primero en
un solo vehículo (Crypto, cuenta B0000001) antes de propagar a Stock/BotCrypto — el cambio sube
tokens del prompt por igual en los 3 vehículos, 3) confirmar en vivo que los 3 vehículos efectivamente
leen la misma fila de `variablesplan` sin necesidad de sync.

## Lo que NO se resolvió en esta sesión (dejalo explícito en tu revisión si aplica)

- Si Fase 2 "Voz" (botones de Telegram, `_propuesta_supervisado()`) ya satisface esa fase del
  roadmap o falta algo — quedó como pregunta abierta sin cerrar.
- El debate abierto `debate-2026-08-19-integracion-tabs-datahub.md` tiene la sección "Perspectiva
  Desktop" y "Síntesis" sin completar — 6 pestañas de la UI (BuySell, Rebalanceo, Sell IA, Buy IA,
  Symbol Events, Alertas) con fricción de ciclo de vida (datos en memoria vs. persistidos) que no se
  tocó en esta propuesta puntual.
- No se implementó código todavía — todo lo de arriba es plan, no PR.

## Lo que te pido

1. **Evaluación crítica de la propuesta completa** — ¿el análisis capa por capa es sólido, o hay
   supuestos débiles? En particular: ¿la decisión de Capa 3 (no alimentar el Consenso Score) es
   correcta, o hay un argumento real para que riesgos SÍ influyan cuantitativamente en algo?
2. **Puntos de mejora concretos** — no genéricos. Si algo falta, decir exactamente qué archivo/función
   tocaría y por qué.
3. **Riesgos no considerados** — ¿algo en Capa 1/2/5 que el repaso pasó por alto? Ej.: ¿qué pasa si
   `variablesplan` está vacía para una cuenta (usuario nunca cargó riesgos) — el prompt debería
   omitir el bloque, o mandar un texto default? ¿Cuánto crece el costo/latencia del prompt con 2
   bloques más, y vale la pena medirlo antes de implementar en los 3 vehículos?
4. **¿Falta algo del roadmap de 4 fases que esta propuesta debería anticipar?** — por ejemplo, si
   Fase 3 (ejecución automática) llega a implementarse, ¿el gate de `salida_emergencia` debería ser
   un bloqueo duro en código (no solo texto en el prompt que Claude puede ignorar)?
5. **Veredicto final**: ¿implementarías esto tal como está planeado, con cambios, o lo pausarías
   hasta resolver algo primero? Un veredicto "está listo, sin cambios" solo es válido si en los
   puntos 1-4 realmente no encontraste nada — no lo uses como salida fácil.

Sé directo y crítico. Preferimos que señales un problema real aunque sea incómodo, a que confirmes
el plan por cortesía. No busques equilibrar cada objeción con un elogio — si el plan tiene 3 puntos
débiles, decí los 3.
