# Modo Autónomo — Doctrina de operación

**Estado:** Definición conceptual — nada de esto está implementado como tal.
**Fecha:** 2026-08-24 — marco planteado por el usuario.
**Alcance:** define **con qué mandato** vende el modo autónomo. No define el mecanismo de
ejecución (eso es Fase 3/4 de [[design-agente-ia]]), ni las reglas internas de cada módulo (esas
viven en el design de cada uno).

**Ver también:**
- [[design-preservation]] · [[design-gains-capture]] · [[ref-oportunidades]] — el detalle de cada módulo
- [[spec-botcrypto]] — estrategia cerrada, fuera de esta doctrina
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

## 3. Parámetros reales por vehículo (2026-08-24)

Los tres mandatos no se distinguen solo por el discurso: se distinguen por los números con los que
corren. **Fuente: `sesion.parameters` de cada vehículo** (vive en BD, no en archivo). Los defaults
del código solo aplican si falta la clave — hoy no falta ninguna en Stock ni en Crypto.

### 3.1 Preservation — `parameters.preservation`

| Parámetro | Stock | Crypto | Default código | Qué controla |
|---|---|---|---|---|
| `roi_minimo` | **0.10** | **0.18** | 0.10 | ROI mínimo del símbolo para entrar a evaluar (no es gatillo de venta) |
| `correccion_pct` | **0.08** | **0.12** | 0.08 | caída desde el máximo que dispara la protección |
| `atr_mult` | **2.0** | **2.5** | 2.0 | múltiplo de ATR para el colchón del STOP |
| `proteccion_base` | 0.40 | 0.40 | 0.50 | fracción de la ganancia que se asegura |
| `proteccion_qty_pct` | 0.33 | 0.33 | 0.33 | % de la posición que cubre el STOP |
| `revisiones_dia` | 2 | 2 | 2 | revisiones repartidas dentro de la ventana 9-16h |

Lo que dicen los números: **Crypto está calibrado más laxo en todo lo que mide movimiento** — pide
casi el doble de ROI para mirar el símbolo (18% vs 10%), tolera una corrección 50% mayor antes de
actuar (12% vs 8%) y deja un colchón de ATR más ancho (2.5 vs 2.0). Es la calibración correcta para
un activo más volátil: menos intervención por ruido.

⚠️ **Pero Crypto no corre.** El loop del agente es `for vehiculo in ("Stock",)` — H6, decisión
2026-08-21: `is_live` dependía de `vehiculo == "Stock"`, así que Crypto simulaba en `[DRY-RUN]` sin
protección real, y se lo sacó explícitamente en vez de dejarlo simulando en silencio. **Los
parámetros están, el mandato no.** Es el primer caso concreto de doctrina declarada sin ejecución
detrás.

Lo compartido entre vehículos (`proteccion_base`, `proteccion_qty_pct`, `revisiones_dia`) no es
casual: no describe el activo, describe **cuánto está dispuesto a resguardar el usuario**. Ese es
justamente el tipo de valor que la sección 4 discute mover a `variablesplan`.

### 3.2 GainsCapture — `parameters.gains_capture`

| Parámetro | Stock | Crypto | Qué filtra |
|---|---|---|---|
| `min_roi` | 0.20 | **0.30** | filtra **LOTES** — calidad: solo entran lotes maduros |
| `min_ganancia` | **200 USD** | **300 USD** | filtra **ESCENARIOS** — fricción: la orden debe justificar comisión e impuesto |

Los dos umbrales actúan en **capas distintas** y esa distinción sí es del módulo: uno decide qué
lotes entran, el otro decide si la orden completa vale la pena. Lo que es del vehículo son los
valores: Crypto pide más en las dos capas — 30% de ROI en el lote y 300 USD en la orden.

Corre en **Stock y Crypto** (`for vehiculo in ("Stock", "Crypto")`), ambos emitiendo orden real —
LMT SELL en IB, LIMIT SELL GTC en Binance — siempre previa confirmación `/ok`.

### 3.3 Oportunidades — `parameters.gains_oportunidades`

| Parámetro | Stock | Crypto | |
|---|---|---|---|
| `min_roi` | 0.09 | **0.20** | umbral de rutina |
| `min_ganancia` | 90 USD | **150 USD** | |

Crypto recalibrado por el usuario el 2026-08-25 (antes: `min_roi` 0.09, `min_ganancia` 50). Ver 3.5.

### 3.4 La doctrina leída en los umbrales

| Umbral | Oportunidades | GainsCapture | Preservation |
|---|---|---|---|
| ROI que pide | 9% Stock / **20% Crypto** | 20% Stock / **30% Crypto** | 10% Stock / 18% Crypto — *para evaluar, no para vender* |
| Ganancia mínima | 90 / **150** USD | 200 / 300 USD | no aplica — no busca ganancia, protege la existente |
| Escalón Oport. → GC | — | ×2.2 Stock · ×1.5 Crypto | — |
| Qué dispara la acción | el umbral mismo | el umbral mismo | una caída (`correccion_pct`) |

Esto **confirma la doctrina en los números, en los dos vehículos**: cada uno tiene un piso de
rutina y un umbral más alto para el movimiento violento, medido sobre la base que le corresponde.
Stock parte de 9% y sube a 20%; Crypto parte de 20% —porque abajo de eso es ruido, no
oportunidad— y sube a 30%. Preservation sigue siendo el único cuyo disparador **no es un nivel de
ganancia sino una caída**: sus umbrales de ROI solo deciden a qué símbolos vale la pena vigilar.

Lo que no es igual entre vehículos es la **amplitud** del escalón: en Stock el salto es ×2.2, en
Crypto ×1.5. No es una contradicción —el escalón existe en ambos— pero tampoco está fundamentado:
nadie decidió que en Crypto los mandatos deban quedar más cerca.

### 3.5 Recalibración de Crypto — cerrada 2026-08-25

Dos ediciones del usuario en la misma sesión, en respuesta a lo que este documento dejó anotado.

**Punto de partida.** `min_ganancia` se movía en **direcciones opuestas** al pasar de Stock a
Crypto: GainsCapture 200 → 300 (Crypto exige más) pero Oportunidades 90 → 50 (Crypto exige menos).
Sin razón de doctrina que explicara el cruce.

**Primera edición** — Oportunidades/Crypto: `min_ganancia` 50 → **150**, `min_roi` 0.09 → **0.20**.
Cierra la asimetría: en Crypto una venta tiene que dejar más que en Stock, sea cual sea el módulo
que la proponga. Pero deja el `min_roi` de Oportunidades igual al de GainsCapture (0.20 ambos), y
con eso desaparece en Crypto el escalón que en Stock separa los mandatos.

**Segunda edición** — GainsCapture/Crypto: `min_roi` 0.20 → **0.30**. Restaura el escalón sobre la
base propia del vehículo: 20% es el piso de rutina de Crypto, 30% el umbral del movimiento violento.

**Estado final.** Los tres mandatos quedan ordenados en los dos vehículos y sin cruces de dirección:

| | Stock | Crypto |
|---|---|---|
| piso de rutina (Oportunidades) | 9% · 90 USD | 20% · 150 USD |
| movimiento violento (GainsCapture) | 20% · 200 USD | 30% · 300 USD |

Queda una diferencia sin fundamentar: el escalón entre ambos módulos es ×2.2 en Stock y ×1.5 en
Crypto. El orden es correcto en los dos casos; nadie decidió que en Crypto los mandatos deban
quedar más cerca entre sí. Anotado como pendiente de decisión y medido contra `booktrading` el
mismo día (abajo).

#### Medición contra `booktrading` — 2026-08-25

Fuente: filas `codigo='C'` (cierres con `gprealizadas`), agrupadas por símbolo + día para que la
unidad sea la **orden** —que es lo que mide `min_ganancia`— y no el lote suelto. ROI de la orden =
`SUM(gprealizadas) / (SUM(producto) - SUM(gprealizadas))`. Solo el vehículo propio: Stock
`U4214563`, Crypto `B0000001`. BotCrypto queda afuera, no lo tocan estos módulos.

**La ventana importa más que la muestra.** El histórico completo arranca en 2020 y no describe la
operación de hoy: otro capital, otro tamaño de posición, y ninguna de las herramientas que existen
ahora. Una calibración hecha sobre 2020-2024 calibraría un sistema que ya no existe. **La ventana
válida es el año en curso, o los últimos 6 meses.** El histórico queda solo como contraste.

##### Stock — 2026 (44 órdenes, 2026-01-07 → 2026-08-12)

| | 2026 | últimos 6 meses (32 órdenes) | histórico 2020→ (276) |
|---|---|---|---|
| ROI de la orden p25 / p50 / p75 | 12% / 23% / 36% | 11% / 21% / 36% | 3% / 12% / 30% |
| ganancia p25 / p50 / p75 | 53 / 110 / 136 USD | 48 / 107 / 133 USD | 4 / 37 / 106 USD |
| ganancia p90 · máxima | 200 · 280 USD | 201 · 280 USD | — · 389 USD |
| pasan Oportunidades (ROI **y** ganancia) | **28** (64%) | **19** (59%) | 86 (31%) |
| pasan GainsCapture (ROI **y** ganancia) | **4** (9%) | **3** (9%) | 15 (5%) |
| franja solo-Oportunidades | 11 | 9 | 56 |

Medido sobre la operación actual, los umbrales de Stock **están razonablemente calibrados**, y eso
no se veía en el histórico. La ganancia mediana de una orden es 110 USD: `min_ganancia` = 90 en
Oportunidades deja pasar ~2 de cada 3 órdenes, que es lo que se le pide a un piso de rutina. Y
`min_ganancia` = 200 en GainsCapture cae **justo en el p90** — deja 4 órdenes de 44, que para un
umbral de "movimiento violento" es selectivo, no muerto. La lectura del histórico ("200 deja 15 de
276") venía de mezclar años con operaciones diez veces más chicas.

##### Crypto — 2026 (2 órdenes)

No hay muestra. Las únicas ventas del año son BTCUSDT y BNBUSDT del 22-23 de agosto, ambas de menos
de 30 USD de ganancia. **Crypto no se puede calibrar hoy**: no hay con qué. Lo único que el dato
dice es que esas dos órdenes cayeron en la franja 20-30% de ROI —la franja donde solo Oportunidades
tiene mandato— y ninguna llegó a los 150 USD.

##### Qué se decide y qué no

- **El escalón de ROI queda como está** (×2.2 Stock, ×1.5 Crypto). La franja solo-Oportunidades
  tiene 11 órdenes en Stock/2026; en Crypto la distribución es de cola gruesa y casi nada cae entre
  20% y 30%. Mover esa amplitud mueve unidades. Deja de ser un pendiente.
- **`min_ganancia` de Stock deja de ser sospechoso.** Sobre 2026 los dos valores (90 y 200) hacen
  lo que dice la doctrina. La nota del BACKLOG #53 se corrige con esto, no se elimina.
- **`min_ganancia` de Crypto (150 / 300) queda sin evidencia**, ni a favor ni en contra. Se revisa
  cuando haya ventas reales de Crypto del año; no antes, y no por analogía con Stock.

**Cómo leer esto con cuidado.** Son ventas que el usuario ejecutó, no propuestas que los módulos
hayan generado: dicen el **tamaño típico de la operación** —que es exactamente lo relevante para un
umbral en dólares— pero no dicen si la venta fue buena.

#### La medición como herramienta permanente

Lo que queda de esto no es el número sino el instrumento. `AppTest/run_booktrading_roi.py` (solo
lectura) acepta una ventana — `6m`, `ytd`, `12m` o una fecha — y reporta, por vehículo, la
distribución de ROI y ganancia por orden y cuántas habrían pasado cada umbral:

```
python AppTest/run_booktrading_roi.py 6m ytd
```

Desde 2026-08-25 la misma medición corre sola: `Agente_RoiVentas` (`Class_AgentManager.py`, cada 30 días, ventana fija de 6 meses) llama a `PlanInversion.roi_ordenes_venta(account, meses, umbrales)` para Stock y Crypto, con los umbrales vigentes de `DataHub.gains_config()`, y deja una línea por vehículo en el log `Agente.Infra`. El script sigue existiendo para consultas puntuales por otra ventana; el agente es la serie de tiempo. **Qué se hace con esa serie todavía no está definido** — se deja acumulando primero.

De acá en adelante **ningún umbral de venta se mueve sin correrlo primero sobre la ventana
vigente**. Y a medida que Oportunidades, GainsCapture y Preservation acumulen ventas propias, la
misma medición pasa de describir lo que hizo el usuario a describir lo que hicieron los módulos —
que es el dato con el que realmente se calibra el modo autónomo.

### 3.6 Lo que no tiene parámetros de venta

- **`agente_ia`** existe solo en Stock (`modo: SUPERVISADO`, `monto_por_trade: 170`, gates de consenso
  e inst_score). Es el lado **BUY** — ver [[design-agente-ia]]. Crypto no tiene motor de decisión IA.
- **BotCrypto** no tiene ninguno de los tres bloques. Sus parámetros son de otro tipo
  (`rsi_buy`, `tp1_pct`, `trail_mult`, `stop_loss_pct`): es una **estrategia cerrada**, no un módulo
  bajo esta doctrina. Ver [[spec-botcrypto]].

---

## 4. ¿Esto puede vivir como restricciones del plan de inversión?

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

## 5. Cruces entre módulos — declarados, sin resolver

**Decisión explícita del usuario (2026-08-24): los cruces no se discuten hasta que cada módulo se
estabilice por separado.** Se listan acá para que nadie los "resuelva" por su cuenta al escribir
otra cosa:

| Cruce | Estado |
|---|---|
| Preservation tiene un STOP vivo sobre un símbolo que GainsCapture quiere vender | H5, BACKLOG #53 — bloqueado para PROD |
| Un mismo símbolo elegible por Oportunidades y por GainsCapture a la vez | sin analizar |
| Qué pasa cuando un `N` (especulativo) queda encima de sus EMAs y pasa a fase long-term | sin analizar |

---

## 6. Qué falta para que esto sea operativo

En orden, sin fechas:

1. Que los tres módulos corran estables por separado — es la condición que puso el usuario.
2. Resolver los cruces de la sección 5 (empezando por H5, que ya bloquea #53).
3. Confirmar o corregir la tabla de grados de autonomía de la sección 2.
4. Migrar el ancho de `variablesplan` y decidir qué límites del plan se vuelven gate ejecutable
   (sección 4). El reparto Oportunidades/GainsCapture en Crypto quedó resuelto por umbrales
   (sección 3.5); la amplitud del escalón de ROI se midió contra `booktrading` y se deja como está.
5. **Calibrar `min_ganancia` de Crypto** cuando haya ventas del año con las que medir — hoy hay
   dos. Los de Stock quedaron validados sobre 2026 (sección 3.5, medición). Regla que queda: ningún
   umbral de venta se mueve sin correr `AppTest/run_booktrading_roi.py` sobre la ventana vigente.
6. Devolverle el mandato a Crypto en Preservation (H6) o declarar por escrito que no lo tiene.
7. Recién ahí, conectar con Fase 3/4 de [[design-agente-ia]].
