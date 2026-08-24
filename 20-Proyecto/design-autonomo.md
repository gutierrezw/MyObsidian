# Modo Autónomo — Doctrina de operación

**Estado:** Definición conceptual — nada de esto está implementado como tal.
**Fecha:** 2026-08-24 — marco planteado por el usuario.
**Alcance:** define **con qué mandato** vende el modo autónomo. No define el mecanismo de
ejecución (eso es Fase 3/4 de [[design-agente-ia]]), ni las reglas internas de cada módulo (esas
viven en el design de cada uno).

**Ver también:**
- [[design-preservation]] · [[design-gains-capture]] · [[ref-oportunidades]] — el detalle de cada módulo
- [[design-agente-ia]] — el motor de decisión y su roadmap de fases (lado BUY)
- [[ref-estrategia]] — el foco del plan: ingresos pasivos + oportunidades de compra/venta

---

## 1. Las tres doctrinas

El sistema tiene tres módulos que pueden generar una venta. **No son tres formas de hacer lo
mismo: son tres mandatos distintos**, cada uno atado a un régimen de mercado diferente. Esta es
la definición del usuario (2026-08-24), textual:

| Módulo | Mandato | Régimen |
|---|---|---|
| **Oportunidades** | tomar ganancias que se adecuan al comportamiento **normal** del mercado | normal |
| **Preservation** | resguardo por **caídas repentinas** — asegura parte de lo que se ha logrado | defensivo |
| **GainsCapture** | aprovechar los **movimientos violentos** de un activo especulativo o un pico de volatilidad, donde sube un montón y vuelve a donde partió | especulativo |

La consecuencia operativa: **la pregunta correcta antes de agregar una regla nueva no es "¿en qué
módulo entra?" sino "¿en qué régimen estoy?"**. Si una regla no puede nombrar su régimen, no
pertenece a ninguno de los tres.

Esto ya está expresado a medias en el código, como dato en vez de como criterio:

| Módulo | Universo que mira hoy |
|---|---|
| Preservation | `categoriaActivo IN ('I','S')` |
| GainsCapture | `categoriaActivo = 'N'` |
| Oportunidades | todo `market` |

`categoriaActivo` clasifica por rendimiento de dividendo, y termina siendo un proxy razonable del
régimen esperado del activo (un `N` sin dividendos es más candidato a movimiento violento). **Es
un proxy, no el criterio.** Anotado como límite conocido: si algún día un `I` entra en un
movimiento violento, hoy ningún módulo tiene mandato sobre él.

Alineación con la lógica de fases por EMAs de [[ref-estrategia]]: *debajo de EMA 100/200* es fase
de acumulación/trading — territorio de Oportunidades; *encima* es fase long-term — territorio del
stop de Preservation, que por eso debe moverse con la tendencia sostenida (promedio de cierres
20-30 días) y no con ruido intradiario. Los dos marcos dicen lo mismo desde ángulos distintos.

---

## 2. Grado de autonomía por módulo

El grado de autonomía **no puede ser uniforme**: se deriva del mandato. Un mandato defensivo se
equivoca hacia la protección; uno especulativo se equivoca hacia la pérdida de oportunidad o hacia
una venta prematura. Eso justifica tratarlos distinto.

### Dónde está cada uno hoy (verificado en código)

| Módulo | Autonomía real hoy |
|---|---|
| Preservation | **Ejecuta**: envía y modifica órdenes STOP reales en Stock (`is_live`). Crypto excluido del loop (H6). |
| GainsCapture | **Propone**: mensaje Telegram con botones, la venta espera `/ok`. La propuesta expira a los 30 min. |
| Oportunidades | **Publica**: CSV + TOP10 + aviso. La orden la coloca el usuario. |

### Propuesta de grados — SIN CONFIRMAR

⚠️ Esta tabla la propone Claude a partir de la doctrina; **el usuario todavía no la validó.**

| Módulo | Puede decidir solo | Necesita confirmación | Límite duro |
|---|---|---|---|
| Preservation | mover el stop hacia arriba | bajar un stop, salir de la posición entera | `stop_final ≥ stop_calculado` (invariante ya vigente) |
| GainsCapture | nada — siempre propone | toda venta | tope contra la posición; no vender por debajo del costo base |
| Oportunidades | nada — siempre publica | toda venta | — |

El criterio detrás: **la autonomía se concede donde el error es recuperable.** Subir un stop, en
el peor caso, cuesta una salida temprana. Vender un lote especulativo en un pico que sigue
subiendo, no.

---

## 3. ¿Esto puede vivir como restricciones del plan de inversión?

Pregunta del usuario (2026-08-24). Respuesta corta: **una parte sí, y no es la parte de la
doctrina.** Hay que separar dos cosas que hoy se confunden:

| Qué es | Dónde debería vivir | Por qué |
|---|---|---|
| **Doctrina** — qué módulo tiene mandato en qué régimen | este documento + el código | No es un límite, es una asignación de responsabilidades. Un número no la expresa. |
| **Parámetros de operación** — umbrales, %, topes por vehículo | `sesion.parameters` (bloques `preservation`, `gains_config`) | Ya es donde viven y ya es donde los agentes leen. |
| **Límites declarados por el usuario** — "qué me haría vender todo", cuánto estoy dispuesto a arriesgar | `variablesplan` | Es la intención original de la tabla, y es lo que le falta a Fase 4. |

Lo que **sí** encaja como restricción del plan es la tercera fila: los límites que el autónomo no
puede cruzar aunque sus propias reglas se lo permitan. Eso ya está pensado en
[[design-agente-ia]] § *Idea — uso de `variablesplan` en modo autónomo*, con `salida_emergencia`
como el candidato más directo.

### Dos bloqueadores antes de poder usar `variablesplan` para esto

1. **`ditem` y `observaciones` truncan a 50 caracteres** (`insert_variablesplan_item()` /
   `update_variablesplan_item()`, `Modulos_Mysql.py:3766-3794`), y `_edit_riesgos()` limita a
   `MAX_FILAS = 10`. 50 chars alcanzan para un **título** ("caída fuerte del dólar"), no para un
   criterio de corte. Hay que verificar el ancho real de la columna en MySQL y migrar antes de
   escribir nada nuevo ahí. Ver Hallazgo 3 en [[design-agente-ia]].
2. **Hoy ningún agente lee `variablesplan`** — son puramente informativas. Que un límite exista en
   la tabla no lo convierte en un gate.

### Riesgo de diseño a evitar

Meter la doctrina como texto libre en un campo del plan y pasárselo a Claude en el prompt **no la
convierte en una regla**: un prompt no es un gate. El modelo va a completar el significado que
falte y a citarlo como fundamento de lo que ya iba a hacer. Si la doctrina va a restringir de
verdad, tiene que ser código que corra antes de la decisión — no contexto.

---

## 4. Cruces entre módulos — declarados, sin resolver

**Decisión explícita del usuario (2026-08-24): los cruces no se discuten hasta que cada módulo se
estabilice por separado.** Se listan acá para que nadie los "resuelva" por su cuenta al escribir
otra cosa:

| Cruce | Estado |
|---|---|
| Preservation tiene un STOP vivo sobre un símbolo que GainsCapture quiere vender | H5, BACKLOG #53 — bloqueado para PROD |
| Un mismo símbolo elegible por Oportunidades y por GainsCapture a la vez | sin analizar |
| Qué pasa cuando un `N` (especulativo) queda encima de sus EMAs y pasa a fase long-term | sin analizar |

---

## 5. Qué falta para que esto sea operativo

En orden, sin fechas:

1. Que los tres módulos corran estables por separado — es la condición que puso el usuario.
2. Resolver los cruces de la sección 4 (empezando por H5, que ya bloquea #53).
3. Confirmar o corregir la tabla de grados de autonomía de la sección 2.
4. Migrar el ancho de `variablesplan` y decidir qué límites del plan se vuelven gate ejecutable.
5. Recién ahí, conectar con Fase 3/4 de [[design-agente-ia]].
