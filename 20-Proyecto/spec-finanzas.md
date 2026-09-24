# Módulo Finanzas Personales + Gmail Co-work

> **Proyecto de segundo plano.** Se trabaja en sesiones cortas sin presión.
> Una fase a la vez. No se integra a AppOO hasta que cada pieza esté probada sola.

## 0. Norte del módulo — objetivos (2026-09-13)

> Definidos por el usuario el 2026-09-13, al revisar el tab Finanzas ya en funcionamiento.
> **Mandan sobre el roadmap de §9**, que queda como historia: el tab (Fase 4) ya existe y la
> carpeta vigilada reemplazó a Gmail como entrada de extractos.

### Objetivos

| # | Objetivo | Qué hace falta | Qué ya existe y se reutiliza |
|---|----------|----------------|------------------------------|
| 1 | Agregar otras cuentas | Un adapter por formato de extracto nuevo + alta en `fin_accounts` | 8 adapters (BBVA ×3, Santander, Citi, BDV, Banesco, Binance C2C) y detección por palabra clave. Banesco (2026-09-13) hereda de BDV y solo cambia la lectura: mide las columnas desde la cabecera del detalle, porque la hoja 1 trae titular y resumen a la izquierda y las siguientes no. Tres cuentas dadas de alta nunca recibieron filas: BBVA CA USD, BBVA CC ARS, Santander Débito |
| 2 | Categorías bien ajustadas y replicadas en todos los extractos | Reglas que comparan el mismo texto que se ve en pantalla, reglas específicas (no `TRANSFERENCIA` a secas) | Mapa plano de 31 categorías con tipo y clase (2026-09-13, ver "Mapa de categorías"). `fin_import_rules` con prioridad y aplicación retroactiva |
| 3 | Ejecución de mínimo esfuerzo, aunque sea importante | El usuario solo deja el PDF (más adelante ni eso: Gmail). Lo que las reglas no clasifican lo sugiere la IA y queda en una cola de un clic; lo que falta avisa solo | `Agente_ExtractosWatcher` (thread propio; un PDF reconocido del que no sale ninguna fila va a `desconocidos/`), `Agente_BinanceApiSync` (C2C y Pay de Binance por API, diario, sin descargar nada — ver "Extractos y cargas automáticas"), patrón Haiku de `Agente_ClasificadorCrypto`, `DataHub.add_alert` → Telegram. `needs_review` y `classification_confidence` ya existen en `fin_transactions`, sin uso |
| 4 | Encontrar el número de libertad financiera y trabajar en función de él: reducir gastos, potenciar inversiones | Gasto anual recurrente ÷ tasa de retiro (4% → ×25, configurable). Progreso = capital invertido ÷ número (base cambiada el 2026-09-20; antes a mercado). Años estimados según la tasa de ahorro. Variante a evaluar: ingreso pasivo ≥ gasto recurrente | `get_totales_inversiones()` ya da el patrimonio invertido en USD de todos los vehículos |
| 5 | Disponible en banco a fin de mes | Guardar el saldo final de cada extracto por cuenta | Los parsers ya leen `SALDO ANTERIOR` / `SALDO ACTUAL` y los descartan. `fin_accounts.balance` existe y está en 0 en todas las cuentas |
| 6 | Diferenciar gastos recurrentes de fortuitos | Detección de mismo comercio con frecuencia ~mensual | `fin_categories.expense_class` (fijo / variable / extraordinario, 2026-09-13). `is_recurring` y `recurring_group_id` ya existen: 0 de 813 filas marcadas |

### Puntos agregados en la revisión

- **Conciliación por saldo.** Saldo anterior + movimientos = saldo actual; si no cierra, faltan filas. Es el
  control de calidad que hoy no existe: los huecos de BBVA CA ARS (03-27→04-29 y 06-26→07-29) pasaron sin
  aviso. Sale casi gratis con el objetivo 5.
- **Monitor de cobertura.** Último dato por cuenta visible en el panel, y alerta cuando una cuenta pasa ~35
  días sin extracto o un PDF termina con 0 filas guardadas (caso `Santander mar-2026.pdf`, 2026-07-22).
- **Las transferencias propias se netean.** Pagar la tarjeta sale de la cuenta y entra en la tarjeta; comprar
  USDT sale del banco y entra en Binance. Si falta la contraparte, parece gasto. Depende del objetivo 1.
- **Deudas en el patrimonio.** El préstamo de 9M ARS (2026-05) tiene saldo pendiente. Patrimonio neto =
  bancos + inversiones − deudas; sin restar la deuda, el progreso hacia la libertad financiera se sobreestima.
- **Tipo de cambio diario.** `fin_exchange_rates` existe con 0 filas; la tasa sale hoy de las operaciones
  propias en `booktrading` (~6 puntos en agosto 2026).
- **Account-first.** Las tablas `fin_*` no tienen dueño (`account`). Hoy hay un solo usuario; si el módulo se
  ofrece a terceros, conviene agregarlo antes de que crezcan los datos.

### Plan consolidado (2026-09-15)

> Pedido del usuario: "necesito saber cómo van mis finanzas y reducir gastos; sin esto no tengo nada". Ordena según
> ese pedido lo pendiente de E2–E6, de los puntos agregados y de la revisión del 2026-09-13. **Reemplaza la tabla
> "Orden de trabajo"**, que queda abajo como historia. Un paso a la vez; cada paso cierra con un número verificado
> contra la BD.

**Foto del 2026-09-15** (consulta de solo lectura, USDT con la regla de signo de E1):

| Qué | Dato |
|-----|------|
| Ingreso | Sueldo 9.936 en jun–ago: ~3.300 por mes. Ingresos totales: jun 4.540, jul 2.746, ago 2.876 |
| Gasto | jun 2.865 · jul 3.281 · ago 1.754 — **agosto incompleto**: la cuenta Santander en pesos llega al 19/08 y las tarjetas al 14–27/08 |
| Gasto promedio por mes, jun–ago | Familia 661 · Alquiler 468 · Cuotas de préstamo 450 · **sin categoría 315** · Supermercado 165 · Expensas 95 · Otros gastos 79 · Suscripciones 63 · Servicios 51 · Restaurantes 49 · Banco e impuestos 42 · Intereses y financiación 25 |
| Distorsiones | 4 débitos "Pago tarjeta de credito visa/amex" de Santander sin categoría (1.033) cuentan como gasto y duplican las compras de la tarjeta · 2 `ACRED.PRESTAMO` de BBVA de abril (1.893) cuentan como ingreso · 6 `TITULOS …CUT` de BBVA de abril (786) cuentan como gasto |
| Cobertura | 3 cuentas nunca cargadas: Santander Débito, BBVA CC ARS, BBVA CA USD · más de 35 días sin dato: Santander CA USD 43, BDV 45, Banesco 46 |
| Higiene | 127 de 210 imports sin filas procesadas · 108 de 237 reglas activas sin aciertos · 0 ciclos de tarjeta · 0 tasas en `fin_exchange_rates` · `needs_review` y la IA sin uso |

*Lectura provisoria:* Familia, Alquiler y préstamo suman ~1.580 por mes, la mitad del gasto. Junio ahorró 37% del
ingreso; julio gastó más de lo que entró, en parte por dos pagos de tarjeta sin categoría. Los números valen recién
al cerrar la Fase 1.

**Fase 1 — Números en los que se puede confiar** (objetivos 1 y 5; ex E3 y parte de E2)

| # | Paso | Cierra cuando |
|---|------|---------------|
| 1.1 | Terminar la carga en curso: 6 extractos de Santander y 6 resúmenes de tarjeta (Santander Visa y Amex, BBVA Visa y Mastercard) con cierre, vencimiento y cuotas a vencer | 48 filas nuevas, 6 ciclos y 32 meses de cuotas en BD |
| 1.2 | Corregir las distorsiones de la foto con SQL autorizado: regla `Pago tarjeta de credito` → Pago tarjeta crédito; `ACRED.PRESTAMO` → Préstamo recibido; `TITULOS` a confirmar con el usuario (compra de títulos → Inversiones) | **Hecho el 2026-09-15** — ninguna fila sin categoría por encima de 100 USDT |
| 1.3 | Monitor de cobertura en el panel: último dato por cuenta y alerta a los 35 días. Decidir las 3 cuentas nunca cargadas (baja o extracto), traer lo que falta (Santander CA USD, BDV, Banesco agosto) y revisar los 3 PDF de Banesco de mayo–julio en `desconocidos/` | **Monitor hecho el 2026-09-16**; el umbral no se cumple todavía: 3 cuentas esperan extracto |
| 1.4 | Mes incompleto a la vista: el panel lo marca mientras alguna cuenta no tenga datos posteriores a su último día | agosto deja de parecer un mes barato |
| 1.5 | Cuotas de tarjeta con la fecha del resumen que las cobra, no la de la compra (pendiente de E2) | **Hecho el 2026-09-16** — 43 de 55 filas en cuotas cambiaron de mes; las cuotas de un plan caen en meses distintos |
| 1.6 | Saldo de fin de mes y conciliación: guardar `SALDO ANTERIOR` / `SALDO ACTUAL` de cada extracto y comprobar que saldo anterior + movimientos = saldo actual | cada extracto cierra o avisa que faltan filas |
| 1.7 | **Inversiones: conciliar contra el extracto del vehículo y ver la rotación** (pedido 2026-09-19). Hoy un rescate de FCI que se lleva a IB o a cripto se netea contra el aporte y el grupo `investment` muestra ~0 o negativo: la plata se movió y no se ve. Dos partes: (a) que una rotación se lea como rotación —de dónde salió y a dónde entró—, no como desinversión; (b) cruzar el flujo bancario hacia inversión contra el patrimonio por vehículo de la tabla `inversion`, mes a mes | el aporte del mes cuadra con el extracto del vehículo, y se ve qué se movió de un vehículo a otro |

**Fase 2 — Clasificación que se sostiene sola** (objetivo 2; resto de E2)

| # | Paso | Cierra cuando |
|---|------|---------------|
| 2.0 | **Bug: la aplicación hacia atrás no tocaba las filas sin clasificar** — `save_rule()` las seleccionaba con `classified_by IS NULL` y después las descartaba con `AND classified_by != 'manual'` (`NULL != 'manual'` es NULL, no TRUE). **Código corregido el 2026-09-18** (predicado único `_SQL_RECLASIFICABLE`, no alcanzaba con el `COALESCE`); falta la pasada única sobre el backlog que el bug dejó sin clasificar — eran 21 filas; las dos de junio del caso testigo se resolvieron el 2026-09-19 al volver a guardar la regla, **quedan 19** | guardar una regla mueve las N filas que el preview prometió **y** el histórico acumulado quedó clasificado |
| 2.1 | Popup de categoría: clave corta —el patrón por defecto era la descripción completa y no volvía a coincidir el mes siguiente—, aplicación hacia atrás mirando el detalle, prioridad 30 y `expense_class` (mejora 2 de la revisión del 2026-09-13). **HECHO el 2026-09-19:** combo `readonly` y sin foco (cambiaba de categoría con la rueda del mouse, causa D), retro con el detalle (causa E) y el default del patrón por `_clave_corta()` — 13 de 13 líneas reales dieron clave usable. El campo del patrón **queda editable** — sin eso hace falta una regla por medio de pago | un ajuste desde el popup clasifica todas las filas del mismo destinatario |
| 2.2 | Pantalla "Revisión de categorías" por destinatario + clic visible en `_CategoryBar` (mejora 3) | reasignar un grupo crea la regla y la aplica |
| 2.3 | Reglas: corregir las que tienen raya o sufijo de origen, desactivar las muertas, genéricas para planes de tarjeta, que gane la más específica (#269). **El criterio no puede ser `hit_count = 0`** — el contador mide coincidencias, no clasificaciones: sube con cada reimportación aunque no escriba nada, y el retro escribe sin subirlo (medido 2026-09-18, regla 490). Se detecta por el patrón: número de tarjeta, comprobante o fecha adentro | ninguna regla activa con la línea completa del resumen como patrón |
| 2.4 | Imports sin filas: revisar y borrar con autorización | 0 imports vacíos sin motivo |
| 2.5 | Comercios repartidos en más de una categoría: listado de destinatarios con 2+ categorías y cuántas filas en cada una, con reasignación del grupo entero. Caso testigo "panadería porteña" | ningún comercio recurrente queda partido sin que sea a propósito |
| 2.6 | Al recategorizar, mostrar lo que ya hay: categorías que ese comercio tuvo, filas en cada una y si ya existe una regla apuntándole. **Hecho el 2026-09-19** salvo la regla: el popup muestra el desglose por categoría y la lista completa de las filas que coinciden, con fecha, monto y categoría actual, y distingue las que puede mover de las que clasificó una persona. Falta mostrar **qué regla gana** el movimiento y poder editarla desde ahí | el popup deja de decidir a ciegas sobre una sola fila |
| 2.7 | **Deshacer el retro**: guardar categoría y `classified_by` previos antes de aplicar una regla hacia atrás. Abierto el 2026-09-19 en el primer uso real — tildar "incluir también las que clasifiqué a mano" pisa el `manual` y no hay vuelta atrás desde la pantalla | se puede revertir la última aplicación de una regla sin SQL |

**Fase 3 — Tablero "cómo voy"** (objetivos 5 y 6)

| # | Paso | Cierra cuando |
|---|------|---------------|
| 3.1 | Vista mensual de 12 meses: ingreso, gasto fijo / variable / extraordinario, ahorro y % de ahorro. Reutiliza `get_monthly_evolution` y `expense_class` | se ve en una pantalla si el mes cerró en positivo y por qué |
| 3.2 | Comprometido del mes próximo: cuotas a vencer por tarjeta (~~`fin_card_installments_due`~~ — sin tabla nueva, ver registro de Fase 3) + cuota de préstamo + fijos promedio | **Cuotas hechas el 2026-09-18**; faltan la cuota de préstamo y los fijos promedio |
| 3.3 | Resumen mensual por Telegram al cerrar el mes: ingreso, gasto, ahorro, 5 categorías principales y desvío contra el promedio (ex E4) | llega solo, sin pedirlo |

**Fase 4 — Reducir gastos** (objetivos 4 y 6)

| # | Paso | Cierra cuando |
|---|------|---------------|
| 4.1 | Recurrentes y suscripciones: `is_recurring` por comercio presente en 3 o más meses; lista con costo mensual y anual. Candidatos de la foto: Adobe, Anthropic, Claro, Netuno, Metrogas, comisiones de cuenta | cada ítem revisado con el usuario: se queda o se corta |
| 4.2 | Costos evitables: comisiones, intereses por descubierto, impuestos de tarjeta, Monthly Service Fee de Citi, intereses de planes; total anual | se conoce el total anual y qué cambio lo baja |
| 4.3 | Presupuesto por categoría = promedio de los 3 últimos meses cerrados; alerta al 80% y al 100% durante el mes (ex E6) | primera alerta real |
| 4.4 | Número de libertad financiera (ex E5): gasto base (`fixed` + `variable`) × 12 × 25; progreso = inversiones − deudas; años estimados con la tasa de ahorro. **HECHO el 2026-09-19:** la vista `Flujo` muestra el promedio por categoría en 1M/3M/6M/1Y, el número al pie (678.111 en 1Y) y el bloque PROGRESO — neto 56.136 = 57.896 a mercado − 411 de margen − 1.349 de cuotas, **8,3% cubierto, 26,8 años** al ritmo de 712/mes con 5% real supuesto. El avance es un **TECHO declarado en pantalla**: el saldo del préstamo no está en ninguna tabla `fin_*` (lo desbloquea 1.6). **Ajustado el 2026-09-20:** la base del avance pasa a ser el **capital invertido** (79.713) y no el valor de mercado (57.847) porque el usuario está en acumulación; el panel muestra `k 0,73` al lado y el techo nombra los dos motivos. Misma tanda: la deuda deja de salir de `SUM(inversion.deuda)` (411) y se lee de `DataHub`, la fuente de la barra `Deuda Total` (6.900) → **10,5% cubierto, 25,9 años** | número y progreso visibles en el panel |

**Fase 5 — Mínimo esfuerzo** (objetivo 3; ex E4 y E6): IA para lo que las reglas no clasifican + cola
`needs_review` de un clic, tasa de cambio diaria en `fin_exchange_rates`, captura por Gmail, `account` en las
tablas `fin_*`.

**Siguiente:** 1.4 (mes incompleto a la vista), con los **dos** bordes del mes: el mes en curso todavía abierto y,
hacia atrás, el mes anterior al primer extracto real de cada cuenta. Un mes sin extracto no es un mes barato, no es
un mes — noviembre 2025 lo mostró. Sin ese piso, el promedio de 4.3 y el costo de vida de 4.4 salen falsamente
optimistas. El piso se mide desde el primer extracto de cada cuenta, no se elige a dedo.

*Cambio de enfoque 2026-09-16:* el piso **no** sale del primer extracto de cada cuenta — un extracto en cero no deja
rastro en BD, así que "primer extracto" confunde cuenta sin movimiento con extracto faltante. Queda declarado por
cuenta en `fin_accounts.tracked_since` con default 2026-01. Motivo y medición en el registro de 1.4.

*Aclaración 2026-09-19:* **"el punto de saldos" que pide el usuario es el paso 1.6**, no una vista nueva. Los parsers
ya leen `SALDO ANTERIOR` / `SALDO ACTUAL` de cada extracto y los descartan, y `fin_accounts.balance` está **en 0 en
todas las cuentas**. Por eso quedó afuera de la vista `Flujo` del 2026-09-19: mostrar saldos hoy sería mostrar ceros.
La conciliación (saldo anterior + movimientos = saldo actual) es además el único chequeo que detecta filas faltantes
sin comparar contra el PDF a mano.

### Fase 1 — registro de lo hecho

**1.2 — distorsiones corregidas** (2026-09-15, SQL ejecutado por el usuario).

*Criterio de préstamo, decidido con el usuario:* devolver capital no es gasto, mueve patrimonio de un lado al otro.
Las categorías 160 y 162 (`Cancelación de préstamo`) pasan a `transfer`; la 43 (`Cuotas de préstamo`) queda
`expense` / `fixed`, porque la cuota mensual sí sale del ingreso del mes.

*Ejecutado:* 21 filas reclasificadas a mano y 11 reglas nuevas — #516 `COMPRA DE DOLARES`→151, #517
`IMPUESTO A LOS SELL`→46, #519 `IMPUESTO CEDULAR`→46, #520 `VISA PLAN V`→150, #521 `Tu pago en dólares`→20, #522
`CANCEL.DEUDA C/SALDO`→20, #523 `REINTEGRO PROMO`→17, #524 `GO-INNERLY`→41, #525 `CRUZ ROJA`→12, #528 `FARMACITY`→4,
más la #507 `Monthly Service Fee` que ya existía. Las 11 quedan con `hit_count = 0`: actúan sobre importaciones
futuras, las filas ya cargadas se corrigieron por id.

*Efecto:* mayo pasó de 4.391 a 2.435 de gasto y de −1.711 a +245 de ahorro. Sin categoría quedan **22 de 1.505
filas (1,5%)**, todas chicas — son las que espera el popup del paso 2.1. El paso cierra: ninguna por encima de
100 USDT.

*KPIs 2026 verificados después del SQL* (ingreso / gasto / ahorro / inversión, USDT):

| Mes | Ingreso | Gasto | Ahorro | Inversión |
|-----|---------|-------|--------|-----------|
| 01 | 3.575 | 1.970 | +1.605 | 807 |
| 02 | 3.280 | 2.237 | +1.043 | 1.189 |
| 03 | 2.586 | 2.012 | +574 | 997 |
| 04 | 3.004 | 3.008 | −3 | 1.427 |
| 05 | 2.680 | 2.435 | +245 | 4.243 |
| 06 | 4.440 | 2.475 | +1.965 | 1.582 |
| 07 | 2.746 | 2.863 | −117 | −136 |
| 08 | 2.834 | 1.740 | +1.093 | −250 |

*Nota del 2026-09-16 — estos totales ya no coinciden con la BD, y no por un error:* filas viejas se reclasificaron
después (las reglas nuevas de este paso pegando al reimportar) y al entrar al grupo `expense` subieron el total del
mes. Medido: **ninguna fila se creó después del 2026-09-15**, pero hay filas de abril a julio con `updated_at` del
2026-09-16. Mayo pasó de 2.435 a 2.587 y abril de 3.008 a 3.160 sin que se cargara un solo extracto nuevo. El gasto
vigente por mes está en el paso 1.5. Es la contracara esperada de 1.2: las reglas se escribieron para actuar sobre
importaciones futuras y lo están haciendo.

**1.3 — monitor de cobertura** (2026-09-16, código). `FinanceScreen.get_coverage()` devuelve una fila por cuenta
activa —último movimiento real, días de atraso, próximo cierre y semáforo— y `_CoverageTable` la muestra en el panel
derecho como segunda vista, alternando con las transacciones del período. No depende del mes ni de los chips: es
estado de carga, no del período.

**Corregido el mismo día — nació como quinta pestaña del notebook de categorías** (junto a Gastos / Ingresos /
Inversiones / Transferencias) y ahí estaba mal por dos motivos. De fondo: la cobertura **no es una categoría**, y el
notebook entero responde al mes y a los chips, así que colgarla ahí ligaba dos cosas que no se parecen. Visible: con
cinco pestañas en 280 px el título se truncaba a `Cobertura (` y el contador —lo único que hacía que el estado se
viera sin entrar, que era el objetivo del paso— quedaba invisible; `Ingresos` y `Transferencias` también se cortaban.

Hoy el encabezado del panel derecho tiene dos títulos clickeables, `Últimas transacciones` y `Cobertura (N)`: el
activo va en negro, el otro en gris, y el contador se pinta en rojo cuando hay cuentas esperando extracto. La tabla
gana el ancho completo del panel (columnas 260/120/90/130 en vez de 175/90/65/95) y el pie de estado pasa a decir
cuántas cuentas hay y cuántas esperan extracto. El filtro de categoría sigue viviendo en `_TxnTable` —expuesto como
`category_filter`— para que el botón `✕ categoría` reaparezca al volver sin duplicar el estado en el panel.

*Semáforo:*

| Estado | Cuándo | Color |
|--------|--------|-------|
| `sin_datos` | la cuenta no tiene ninguna fila cargada | gris |
| `atraso` | tarjeta cuyo **próximo cierre ya pasó**, o cuenta con más de 35 días sin dato | rojo |
| `aviso` | cuenta entre 29 y 35 días | naranja |
| `ok` | el resto | verde |

*Por qué una tarjeta no se mide en días.* Visa lleva 33 días sin movimiento y AmEx 27, y las dos están al día:
cerraron el 20/08 y su próximo cierre es el 24/09. Medirlas por días las pintaría de rojo todos los meses en la
ventana entre cierre y cierre. Lo que delata un resumen faltante es que `fin_card_cycles.next_closing_date` haya
quedado en el pasado. Si la tarjeta no tiene ciclo cargado, cae al criterio de días.

*Las filas sintéticas no cuentan como dato cargado.* `sync_binance_investment` escribe la fila de inversión con
fecha de fin de mes (`classified_by = 'synthetic'`, hoy hasta 2026-09-30, una fecha futura). Si contara, Binance
figuraría siempre al día. El monitor excluye `classified_by = 'synthetic'` y las fechas posteriores a `CURDATE()`.
Es el mismo motivo por el que `_binance_api_desde()` filtra por `import_id`, pero ese método mira por sección y no
sirve por cuenta.

*Estado medido el 2026-09-16:* atrasadas Banesco CC VES (31/07, 47 días), BDV CA VES (01/08, 46) y Santander CA USD
(03/08, 44). Las otras 9 al día, tarjetas incluidas, y ninguna en gris tras la baja de abajo. **Ningún
resumen de tarjeta está atrasado.**

*Cuentas nunca cargadas* — el usuario decidió darlas de baja: Santander Débito (5), BBVA CC ARS (6) y BBVA CA USD
(7). `UPDATE fin_accounts SET is_active = 0 WHERE id IN (5, 6, 7);` — **ejecutado por el usuario el 2026-09-16**: el
monitor pasó de 15 cuentas a 12 y de `Cobertura (6)` a `Cobertura (3)`, sin ninguna fila en gris. Santander Débito
importaba con `row_count = 0` en cada corrida; las dos de BBVA nunca se importaron. Reversión:
`UPDATE fin_accounts SET is_active = 1 WHERE id IN (5, 6, 7);`

*Los 3 PDF de Banesco en `desconocidos/` no son datos faltantes:* son mayo, junio y julio de 2026, meses ya
cargados (90, 14 y 15 filas). Llegaron ahí por la rama "reconocido pero sin filas extraídas" de
`procesar_extractos`. Se pueden borrar.

*Extractos que faltan bajar para cerrar el umbral de 35 días:* BBVA Mastercard Black y Visa Signature del cierre
~27/07 (el hueco entre los imports 182 y 191 explica que falte la cuota 04/06 de PRESTIGIO y que julio traiga solo
dos filas); Santander CA/CC ARS desde el 20/08; Santander CA USD de agosto y septiembre; BBVA CA ARS desde el
29/08; Banesco CC VES y BDV CA VES de agosto y septiembre.

**1.5 — la fecha de las cuotas es material, no un detalle.** Medido el 2026-09-15: las cuotas se cargan con la
fecha de la compra original, así que un plan entero cae en un solo mes. PRESTIGIO SA tiene las cuotas 01, 02, 03 y
05 las cuatro el 2026-04-02, aunque vinieron de los imports 127, 149, 182 y 191 (resúmenes de mayo, junio, julio y
septiembre). Pasa en los dos bancos: VAYPOL cuotas 3 a 6 todas el 2025-11-09; SPORTING cuotas 1, 3, 4, 5 y 6 todas
el 2026-01-22; "Look shopping adrogue" 6 cuotas el 2026-02-13. El peor caso es `Merpago/juliannahuelcapec`: 3
cuotas de 191.088 ARS las tres el 2026-01-30, ~360 USDT cargados a enero que en realidad se pagaron en febrero,
marzo y abril. Mientras siga así, el gasto mensual de cualquier mes con compras en cuotas está inflado y los
siguientes vacíos.

*Resuelto el 2026-09-16.* La fila cuenta en el mes del **cierre** del resumen que la lista, no en el del vencimiento
(decisión del usuario): el cierre es lo que el resumen afirma, el vencimiento depende de cuándo se pague.

*Por qué no se mueve `date`.* `uq_tx` es `(account_id, date, amount, raw_description)`, así que reescribir la fecha
puede colisionar con otra fila del mismo plan, pierde la fecha real de la compra y deja que una reimportación del
mismo resumen vuelva a insertar con la fecha vieja. En su lugar va **`fin_transactions.billing_date`** (`DATE NULL`,
después de `date`): el cierre del resumen que cobra la fila, NULL cuando la fecha del movimiento ya es la del cobro.
Los KPIs agrupan por `_SQL_FECHA = "COALESCE(t.billing_date, t.date)"` — `get_kpis`, `_get_categories_by_type`,
`get_monthly_evolution`, `get_transactions` y `get_last_loaded_period`. `get_coverage` **no** lo usa a propósito:
mide cuándo se cargó movimiento real, no cuándo se cobra.

*Quién sella.* `_set_billing_dates()` (`Class_Finance.py`) marca las cuotas del import al cerrarlo. Lo llaman
`BbvaArTarjeta.load()` en sus dos ramas —la normal y la del resumen ya importado—, `SantanderAr.load()` una vez al
final sobre todos los imports del PDF, y `SantanderArTarjetaResumen.load()` al grabar el ciclo, que re-sella lo ya
cargado de esa tarjeta. Santander trae los consumos de tarjeta dentro del extracto de cuenta, con `import_id` distinto
del resumen que aporta el ciclo, así que el ciclo no le sirve como fuente en el momento de la carga. `CitibankUs` no
lo llama, no tiene cuotas. **Los dos bancos entran por caminos distintos a propósito:** BBVA pasa `closing` explícito
—lo imprime en el resumen— y Santander no, así que cae a la rama que lo deriva. Detalle en el registro de 1.5.

*El proxy está medido, no supuesto.* `fin_card_cycles.closing_date` cubre solo 6 de las 55 filas —los ciclos se
empezaron a leer con los resúmenes de julio/agosto 2026 y las cuotas arrancan en noviembre 2025—. El resto sale de
`fin_statement_imports.period_to`, que es `MAX(date)` de las filas del import: cobertura 100%, y contra el cierre real
coincide exacto en 5 de los 6 casos donde existen los dos (el sexto difiere 5 días, mismo mes). Como solo se usa el
mes, alcanza.

*Y quedó validado end-to-end después del backfill:* comparando el gasto mensual con el proxy contra el gasto con el
`billing_date` realmente escrito, los 12 meses dan **idéntico**. De las 6 filas que tienen ciclo real, 3 traían
`period_to` 5 días antes del cierre (22/08 contra 27/08) y 3 coincidían, y ninguna cruza el borde del mes. El proxy
no cambió ni un resultado.

*Backfill, ejecutado por el usuario el 2026-09-16* (subquery en vez de `LEFT JOIN` porque nada garantiza un solo ciclo
por import y un `UPDATE` multi-tabla elegiría cualquiera de los dos):

```sql
UPDATE fin_transactions t
  JOIN fin_statement_imports i ON i.id = t.import_id
   SET t.billing_date = COALESCE(
       (SELECT MIN(cc.closing_date) FROM fin_card_cycles cc WHERE cc.import_id = t.import_id),
       i.period_to)
 WHERE t.installment_current IS NOT NULL;
```

Reversión: `UPDATE fin_transactions SET billing_date = NULL WHERE installment_current IS NOT NULL;` y después
`ALTER TABLE fin_transactions DROP COLUMN billing_date;`

*Verificado tras el SQL:* 55 filas selladas y ninguna sin sellar; 0 filas fuera de cuotas tocadas; ningún cierre
anterior a su compra (0 a 171 días); **43 de 55 cambiaron de mes, 1.055,64 USDT reubicados**, y las 12 restantes se
quedan porque el resumen cerró dentro del mes de la compra. El peor caso quedó repartido: de las 3 cuotas de
`Merpago/juliannahuelcapec` que estaban las tres en enero, 252,50 USDT pasaron a febrero.

**Noviembre 2025 quedó en 0 y desapareció del panel**, y eso es el hallazgo lateral que abre 1.4: sus únicas filas
eran cuotas de VAYPOL que llegaron por resúmenes de 2026. No hay ningún extracto cargado de ese mes — era un mes
fantasma que se veía como el mes más barato del histórico.

*Gasto mensual como lo lee el panel después del cambio (USDT):* 2025-06 33,14 · 2025-07 116,17 · 2025-12 122,49 ·
2026-01 1.675,40 · 02 3.291,38 · 03 2.089,76 · 04 3.159,41 · 05 2.617,92 · 06 2.521,13 · 07 2.928,14 · 08 1.962,56.

Noviembre 2025 no era el único mes fantasma: **2025-06 (33,14), 2025-07 (116,17) y 2025-12 (122,49)** siguen en la
lista con un gasto que no es el de un mes de vida. Son el material del piso de 1.4 — el piso se mide, no se elige.

*Pendiente, fuera de este paso:* 87 consumos de tarjeta **sin** cuota (1.872,75 USDT) también caen en un mes distinto
al del cierre que los cobra, contra 312 (8.301,42) que coinciden. Y dos filas de la cuenta 4 categorizadas
`Otros gastos` que matchean el patrón de pago de tarjeta (14,78 USDT en 2026-06 y 14,46 en 2026-07): si son pagos van
a `transfer`, hoy suman como gasto.

**1.4 — los dos bordes del mes** (en curso desde 2026-09-16). Medido en solo lectura antes de tocar el panel.

*Techo (mes en curso).* 2026-09 da **gasto 0,00**: sus 11 filas son de Citi y Binance, ninguna de gasto, porque los
extractos de Santander y BBVA cierran el 20 y el 27. El panel abre en ese mes (`get_last_loaded_period()`), así que
muestra cero como si fuera el mes. Agosto también está incompleto (1.962 contra 2.500-3.300 de la serie).
`fin_transactions` llega a 2026-09-30 por las filas sintéticas de Binance (fin de mes): toda marca de "hasta dónde hay
datos" las excluye, igual que `get_coverage()`.

*Piso (histórico).* 2025-06 y 2025-07 son escombro de cuotas viejas que trae el resumen de enero; 2025-08 a 2025-11 no
existen; 2025-12 es parcial (los extractos arrancan entre el 18 y el 30/12). Desde 2026-01 todas las cuentas que
existían tienen movimiento. El primer movimiento por cuenta **no sirve de piso**: Banesco aparece en 2026-05, pero el
usuario tiene sus extractos y están en cero hasta junio — no es un hueco, es una cuenta sin movimiento.
`process_pdf()` manda a `desconocidos/` todo PDF con `rows_found = 0`, así que un mes en cero no deja rastro y la BD
no lo distingue de un extracto que falta (anotado para 1.6).

*Decisión del usuario:* fecha de inicio declarada por cuenta, **default 2026-01** — 2025 no se reconstruye (los bancos
dan ~12 meses). Banco de Venezuela sigue activa aunque tiene gasto 0 desde junio: darla de baja crearía otro hueco.
SQL ejecutado por el usuario el 2026-09-16:

```sql
ALTER TABLE fin_accounts
  ADD COLUMN tracked_since DATE NOT NULL DEFAULT '2026-01-01'
  COMMENT 'Desde cuando la cuenta cuenta en el historico de Finanzas: antes de esta fecha sus meses vacios no son hueco de carga. Default 2026-01 = piso del modulo; 2025 no se reconstruye (los bancos dan ~12 meses)'
  AFTER is_active;
```

Reversión: `ALTER TABLE fin_accounts DROP COLUMN tracked_since;`. Todavía no la lee ningún código: la consume la marca
de bordes del panel, pendiente de diseño.

*Enero en estudio.* Da 1.675 contra 2.768 de promedio feb-jul. **No falta ningún extracto**: todas las cuentas tienen
enero con filas normales (Santander ARS 47 contra 44-76; tarjetas dentro de su rango). El faltante se concentra en
gastos fijos — Familia −558, Alquiler −193, Intereses −162, Préstamo −124, Expensas −99 — y Tecnología +291 lo
compensa en parte. Hipótesis: pagos de enero corridos a febrero (el mes más caro de la serie, 3.291; enero + febrero
= 2.483 por mes, en línea con el resto). Si se confirma, enero es un mes válido y el piso queda en 2026-01. Para
medirlo se cargó el extracto de Santander 21/11/2025–23/12/2025.

*Enero resuelto (fila por fila, sin necesitar diciembre): es un mes real, el piso queda en 2026-01.* Los fijos no
faltan, **subieron durante el año**: alquiler 346 → 438-484 desde mayo (abril trae 2 pagos, 1.097, y eso infla el
promedio); el préstamo BBVA de ~195 se canceló en mayo (1.956) y desde junio hay otro de 421-507; expensas (BDV, luego
Banesco) 42 → 86-271. Lo único corrido es **una transferencia a Familia** de Santander: la de enero salió el 03/02
(224) y febrero tiene dos. La hipótesis "febrero alto por pagos de enero" explica solo esos 224.

*Hallazgo que infla febrero (pendiente, es residuo de 1.5):* Intereses y financiación da 958 en febrero con 18 filas.
Las cuotas de los planes de refinanciación traen el número en la descripción con guion — `CUOTIFICACION 2-06`,
`VISA PLAN V 2-06`, `Visa plan v 2-04` — y los lectores solo reconocen `C.02/06` (BBVA TC, `RE_CUOTA`) y `N de M` en la
columna de cuota (Santander, `RE_CUOTA_SAN`). Quedan con `installment_current` NULL, `_set_billing_dates()` no las
sella, y las cuotas 1 a 5 de cada plan caen todas en la fecha de la operación, 07/02 y 01/02. Además hay dos filas
`CR.$ CUOTIFICACION` (200) y `CR.$ PLAN V` (206) contadas como gasto que parecen el crédito del refinanciamiento: falta
confirmar con qué signo las lee el panel.

*Banesco mayo explicado:* las 90 filas del import 252 son reales — comisiones de SMS y de mantenimiento atrasadas
(`COM,SERV.MTO` de 2024 y 2025, ~0 USDT) cobradas al reactivar la cuenta el 05/05, más una entrada de 47 y una salida
de 15,63. No hay duplicado.

*Extracto de diciembre (verificado):* el PDF cargado fue `2026-01-22_00720175007003707199.pdf` — el nombre lleva la
fecha de cierre, así que es el período 24/12–22/01, ya cargado —, no el de diciembre (`2025-12-23_…`). No entró ninguna
fila y dejó 4 imports fantasma (331-334, `row_count` 39/0/14/13, `processed_count` 0), mismo caso que 327-330. El de
diciembre ya no hace falta: enero se resolvió sin él y diciembre queda antes del piso. Los 3 PDFs de Banesco de
`desconocidos/` coinciden por sha256 con los imports 252-254: son re-cargas de extractos ya importados.

*Por qué el techo no se mide con lo que hay en BD:* `fin_statement_imports.period_from`/`period_to` guardan la primera y
la última fecha de movimiento (`MIN`/`MAX(date)` de las filas del import) en todos los lectores salvo el resumen de
tarjeta de Santander, que guarda el ciclo real. La BD no sabe hasta qué día cubre cada extracto: una cuenta de poco
movimiento parece atrasada, un extracto en cero no existe y una re-descarga no se reconoce como el mismo período. El
techo de 1.4 y los tres hallazgos de 1.6 tienen la misma raíz.

*Cuotas con guion medidas (2026-09-17, `cuotas_guion.py`, solo lectura):* 24 filas en 7 planes de 4 tarjetas, todas en
Intereses y financiación, sin `installment_current`. Cada cuota vino del resumen de su mes (import distinto), pero
todas llevan la fecha del alta del plan. Aporte al gasto hoy contra el mes del resumen que la cobra, en USDT:
2025-06 33→0, 2025-07 116→0, 2026-01 0→72, **02 522→148**, 03 0→148, 04 176→145, 05 0→145, 06 0→76, 07 30→78,
08 0→65. El escombro de 2025-06/07 que se vio en el piso **es esto mismo**: cuotas 7 a 9 de planes de 2025 cobradas en
enero-marzo. Las filas `CR.` se leen bien: entran como `income` y restan del gasto — `CR.$ CUOTIFICACION` (−200) y
`CR.$ PLAN V` (−206) son el saldo que pasa al plan, y el plan lo cobra en cuotas con su interés. Faltan cuotas que no
llegaron por ningún formato (6-06 de los dos planes de BBVA, 1-06 de `FINANC DE SALDO`): pendiente de revisar al
arreglar el lector. No se toca sin decisión.

*Opción A aplicada (2026-09-17) — los bordes que no dependen del período del extracto:*

- `get_last_loaded_period()` abre el panel en el último mes con movimiento real **anterior** al mes en curso y excluye
  las filas sintéticas. Antes abría en septiembre con gasto 0.
- `get_monthly_evolution()` cuenta cada fila desde el `tracked_since` de su cuenta (primer consumidor de la columna) y
  la ventana son N meses calendario enteros: antes arrancaba en `CURDATE() - N meses`, así que el primer mes entraba
  cortado a mitad. Devuelve `en_curso` para el mes abierto.
- `_EvolucionChart` pinta el mes en curso rayado en gris con la leyenda "en curso" y no le pone la franja roja de
  gasto mayor que ingreso: con el mes abierto esa comparación no dice nada.

Lo que **no** resuelve A: agosto sigue viéndose como mes normal aunque le faltan los extractos del 20 y el 27. Marcar
el mes anterior como incompleto requiere guardar el período real de cada extracto (opción B, primer paso de 1.6).

*Lector de cuotas con guion arreglado (2026-09-17):* `parse_plan_installment()` (`Class_Finance.py`) reconoce la cuota
del plan escrita en la descripción (`CUOTIFICACION 3-06 (TNA 79,00)` → 3 de 6). La usan los dos lectores que traen
planes: `BbvaArTarjeta._parse_cuota()` cuando no hay `C.02/06` y `SantanderAr._process_tc_line()` cuando la columna de
cuota no trae "N de M". Con la cuota reconocida, `_set_billing_dates()` la sella en el resumen que la cobra, igual que
el resto de las cuotas. Decisiones:

- La cuota no lleva cero adelante (`1-06`, `9-09`): así una fecha `07-12` no pasa por cuota. Se exige además
  cuota ≤ total.
- En BBVA la cuota del plan **queda** en `description`; la de `C.02/06` se quita. Las filas de un plan solo se
  distinguen por ese número: sin él la tabla muestra seis filas iguales.
- Las 24 filas ya cargadas no se reprocesan: `scan_extractos()` borró los PDFs. Se corrigen por SQL con la misma regla
  (cuota, total y `billing_date` = cierre del ciclo del import, o su último movimiento si no tiene ciclo).

*Verificado (2026-09-17, `verifica_cuotas_plan.py`):* no queda ninguna cuota de plan sin reconocer. Los cuatro planes
de Santander tienen una cuota por mes y no les falta ninguna; la 6-06 del plan de TNA 83,10 cae en el resumen de
septiembre, todavía sin cargar. En BBVA quedan dos cosas abiertas:

- Faltan CUOTIFICACION 6-06, VISA PLAN V 6-06 y `FINANC DE SALDO` 1-06: **no está cargado el resumen de fines de
  julio de ninguna de las dos tarjetas**. Entre los imports 182/183 (archivos `2026-07-02`) y 191/192 (cierre 27/08)
  no hay ninguno, tampoco sin filas. Es la misma ausencia que la cuota 04 de PRESTIGIO medida en 1.5.
- El import 183 (Visa) cuenta en julio y **está bien**. Tiene un consumo del 01/07 (LUCCIANOS), así que su cierre es
  del 01/07 o posterior, en línea con el archivo `BBva VISA 2026-07-02.pdf`. Una primera lectura lo daba por resumen
  de junio "por la secuencia de cuotas": la secuencia solo dice que es el siguiente al de mayo, no en qué mes cierra.
- Queda abierto el import 182 (Mastercard). Su último movimiento es del 26/06, así que sus cuotas cuentan en junio.
  Se cargó junto con el 183 desde `BBva Master 2026-07-02.pdf`, y en agosto las dos tarjetas cerraron el mismo día.
  Si también compartieron el cierre del 02/07, sus cuotas son de julio. Se resuelve con el cierre impreso en el PDF.

*BBVA cerrado (2026-09-17, mismo día):* las dos cosas abiertas de arriba quedan resueltas.

- **Entró el resumen del 30/07** (imports 335 Mastercard y 336 Visa, con ciclo) y trajo las cuotas que faltaban. La
  1-06 del plan de TNA 69,00 figura como `CUOTIFICACION`; la 2-06 de agosto, como `FINANC DE SALDO`. Tres cuotas de
  plan entraron sin reconocer: las dos 6-06 de TNA 79,00 y esa 1-06. Las 6-06 caían en febrero, el mes del alta.
  Causa probable, no confirmada: la app que corre el watcher arrancó a las 02:20 y el lector nuevo se guardó a las
  02:27. El regex actual sí las reconoce. Se corrigieron por SQL con la misma regla (cuota, total y cierre del ciclo,
  30/07). **Un lector nuevo entra en el watcher recién al reiniciar la app.**
- **El import 182 cierra en julio.** El homebanking lista los cierres de las dos tarjetas, el mismo día para
  Mastercard y Visa: 02/07, 30/07 y 27/08. Las 5 cuotas del 182 pasaron de junio a julio con `billing_date` =
  2026-07-02: SPORTING 6/6, CUOTIFICACION 5-06, PRESTIGIO 3/6 ×2 y ASTERISCO 1/2. Las 3 del 183 van a la misma fecha
  sin cambiar de mes. El último movimiento no alcanzaba: el 182 no tuvo consumos entre el 27/06 y el cierre.
- **Consecuencia de contar la cuota en el mes del cierre:** BBVA cerró dos veces en julio y ninguna en junio. Junio
  queda sin cuotas de BBVA y julio carga dos resúmenes por tarjeta. No es un hueco de carga.

*El proxy sí cruza de mes en Santander.* Los imports 290, 305 y 315 sellaron 5 cuotas un mes antes del resumen que las
cobra: 305 en febrero y no en marzo, 290 en mayo y no en junio, 315 en junio y no en julio. En Santander el cierre de la
tarjeta es la fecha del nombre del archivo, que coincide con el último movimiento del PDF entero. Los dos ciclos leídos
lo confirman: 23/07 y 20/08. El riesgo no es solo de las recargas: `SantanderAr` sellaba con el último movimiento de la
**sección** (`_set_billing_dates(cursor, import_id)` sin `closing`, `Class_Finance.py`), y a una tarjeta con poco uso le
podía pasar lo mismo en una primera carga. Así era hasta el fix del 2026-09-18, más abajo en este mismo paso.

*Corregido y verificado el 2026-09-18.* `billing_date` de esas 5 cuotas pasó a `MAX(date)` de **todos** los imports que
comparten el `file_hash` — el PDF entero, no la sección. La dedup de Santander es por `(file_hash, section)`, así que
las 5 secciones de un mismo PDF son 5 imports con el mismo hash.

- **El proxy quedó validado antes de escribir:** el `MAX(date)` del PDF entero coincide **exacto** con la fecha del
  nombre del archivo en los tres casos — 25/06, 19/03 y 23/07. Es lo que habilitó el UPDATE.
- **Corrección de lo que decía esta entrada:** no son "una segunda carga de los extractos 249, 111 y 244". Los tres son
  resúmenes distintos de la misma tarjeta (`2026-06-25_`, `2026-03-19_`, `2026-07-23_`), y 96, 111, 234, 239, 244, 249,
  285 y 310 son otros resúmenes, no su primera carga. Lo que sí es cierto es el mecanismo: la sección `santander_visa`
  de esos PDFs entró con **3 filas** cada una porque el resto ya estaba cargado (`_is_near_duplicate`), y con tan pocas
  filas el último movimiento de la sección queda semanas antes del cierre.
- **Síntoma que lo delata, y que sirve de test:** dos cuotas del mismo plan cobrando el mismo mes y un hueco al mes
  siguiente. Los cuatro planes afectados quedaron con una cuota por mes y en orden — Look shopping (6 cuotas) 03, 04,
  05, 06, 07; Merpago (3) 02, 03, 04; Simplicity232 (3) 06, 07, 08; Simplicity290 001304 (3) 05, 06, 07.
- **No había gasto duplicado.** Cada cuota aparece una sola vez en su plan, y el barrido de la tarjeta entera por
  `(account_id, date, amount, currency, raw_description, installment_current, installment_total)` dio vacío.
- **Un chequeo de duplicados sobre `fin_transactions` tiene que incluir `installment_current`.** La clave de `uq_tx`
  —`(account_id, date, amount, raw_description)`— no distingue un duplicado real de otra cuota del mismo plan: las 6
  cuotas de Look shopping son 28.183,33 del 13/02 con la misma descripción y el mismo comprobante. El primer chequeo
  se escribió sin la cuota y dio 5 falsos positivos.
- **Impacto medido en el gasto por mes (ARS):** febrero −191.088, marzo +191.088, mayo −50.016, junio +7.658,
  julio +42.358. A tasa ~1.500 son −127, +127, −33, +5 y +28 USD: solo febrero y marzo se notan en el gráfico.
- **Abierto (a):** falta la cuota **1/6** de Look shopping adrogu (comprobante 005844, ~28.183 ARS). La serie arranca
  en 2/6 y el resumen que la cobraría —el de febrero, import 96, archivo `2026-02-19`— sí está cargado.
- **Abierto (b) — CERRADO el 2026-09-18 por el fix de código de abajo.** El proxy seguía corrido en otros imports de la
  tarjeta, pero **sin cruzar de mes**, así que quedaron fuera del UPDATE. Ejemplo: la cuota 3/6 de Look shopping
  (import 310, archivo `2026-04-23`) cobraba el 05/04, dieciocho días antes del cierre.

*El fix de código, 2026-09-18.* Los tres UPDATE manuales de este paso eran parche: `SantanderAr` seguía sellando con
el último movimiento de la **sección**, así que el próximo resumen volvía a ensuciar el dato
(ver `10-Memoria/feedback_saneamiento_termina_en_codigo.md`). Ahora la rama `closing is None` de `_set_billing_dates()`
hace lo que antes se hacía a mano, en tres piezas:

1. **Deriva el cierre del PDF entero, no de la sección** — `MAX(date)` de todos los imports que comparten el
   `file_hash`, y si hay un cierre real de la cuenta en `fin_card_cycles` dentro de `CIERRE_MAX_DIAS` lo prefiere.
2. **`SantanderAr.load()` sella fuera del bucle de secciones**, sobre todos los imports del `file_hash`. Antes el
   sellado vivía dentro del bucle, y la rama de sección ya importada hace `continue` sin sellar — justamente el caso
   de las secciones de 3 filas que originaron el problema.
3. **`SantanderArTarjetaResumen.load()` re-sella la tarjeta al grabar el ciclo.** El resumen de tarjeta (otro PDF,
   otro `file_hash`, misma `account_ref`) llega después del extracto de cuenta; ahora lo ya cargado con la
   aproximación se corrige solo, sin volver a soltar el PDF — que además cambiaría de `file_hash` al re-descargarlo.

*De dónde sale `CIERRE_MAX_DIAS = 10`.* La BD tiene varios cierres por tarjeta y hay que elegir cuál corresponde a
este resumen. Medido sobre los 36 resúmenes con cuotas: cuando el cierre es el propio está a **0, 1, 1, 2, 5 o 5
días** del último movimiento del PDF, y el valor siguiente salta a **28** — el cierre del mes próximo. Nada en el
medio, así que el corte en 10 separa los dos grupos sin ambigüedad. Resuelve el caso 249-contra-315, que rompía una
versión anterior de la regla: el 249 queda a 28 días y cae al último movimiento (25/06, junio), el 315 toma el cierre
real (23/07, julio). Cada uno en su mes.

*BBVA no entra, y es deliberado.* `BbvaArTarjeta` lee el cierre impreso en su propio resumen y lo pasa explícito, así
que nunca llega a la rama modificada. La asimetría no es descuido: es que los dos bancos dan datos distintos y el de
BBVA es mejor. Verificado por las malas — aplicado a todos, el import 182 se movería del 02/07 al 26/06, hacia atrás
y a un mes que el homebanking desmiente.

*Re-sellado de lo ya cargado, ejecutado por el usuario el 2026-09-18* con un script que llama al mismo
`_set_billing_dates()` del lector, así que lo que aplicó es exactamente lo que hará el código de acá en adelante.
Sobre 36 resúmenes con cuotas: **20 re-sellados** (Santander), **16 intactos** (BBVA, cierre propio), **11
cambiaron**, corrección máxima **18 días** (imports 300 y 310) y **ninguno cruzó de mes** — la serie mensual de gasto
ya era correcta y lo que se gana es precisión de día para el comprometido del mes próximo. El script abortaba con
rollback si no reproducía las 7 correcciones manuales de este paso (imports 290, 305, 315, 182, 183, 335, 336); las
reprodujo todas.

Los 11 cambios, con el origen del cierre: el límite encontró el cierre real del banco en los imports 239, 240 y 244
(a 1 día del último movimiento) y los demás cayeron al último movimiento del PDF, que es lo que corresponde cuando el
resumen de tarjeta de ese mes no está cargado. Ningún caso quedó en la zona gris.

*Primera corrida del verificador: falso positivo propio, vale como advertencia.* Decidía si BBVA tenía cierre propio
buscándolo en `fin_card_cycles`, y los resúmenes de BBVA de enero a julio no tienen ciclo grabado (los ciclos se
empezaron a leer en agosto). Con eso mandó 182 y 183 al fallback y el guardarraíl abortó, correctamente. El defecto
era del script, no del fix: **el cierre de BBVA vive en el PDF y no hay forma de recuperarlo desde la BD**, así que
cualquier verificación futura tiene que excluir las secciones cuyo lector pasa `closing` explícito, no intentar
reconstruirlo.

### Orden de trabajo (2026-09-13 — reemplazada por el plan consolidado del 2026-09-15)

E1 quedó hecho; E2 pasa a la Fase 2 y al paso 1.5, E3 a la Fase 1, E4 al paso 3.3 y la Fase 5, E5 al paso 4.4 y E6
al paso 4.3 y la Fase 5.

| Etapa | Contenido | Objetivos | Estado |
|-------|-----------|-----------|--------|
| E1 | Números correctos: los KPIs respetan entradas y salidas (regla abajo) + corrección de datos de la categoría Financiación | base de todo | Hecho. Código `24a8ddc`; los datos de Financiación se corrigieron con el mapa de categorías (2026-09-13) |
| E2 | Categorías y reglas: árbol, tipos, las 38 reglas con sufijo de origen (0 aciertos), `TRANSFERENCIA` demasiado amplia, marca fijo / variable / extraordinario | 2, 6 | En curso. Mapa y marca hechos en BD (2026-09-13). Pendiente: reglas e importadores (lista en "Mapa de categorías") |
| E3 | Saldos, conciliación, cobertura y alta de cuentas nuevas | 5, 1 | Pendiente — falta la lista de cuentas |
| E4 | Mínimo esfuerzo: IA para lo que no matchea, cola de revisión, alertas y resumen mensual por Telegram | 3 | Pendiente |
| E5 | Libertad financiera: número, progreso, años estimados, gasto recurrente base | 4 | Pendiente — depende de E1–E3 |
| E6 | Presupuestos derivados del gasto recurrente, tasa diaria, captura por Gmail | 3, 4 | Pendiente |

### Regla de cálculo de los KPIs (E1)

- **Grupo** (ingreso / gasto / inversión / transferencia): lo decide la categoría. Sin categoría, lo decide `t.type`.
- **Signo:** suma si la fila va en el sentido del grupo (en ingresos entra; en los demás sale) y resta si va en
  contra. Un reintegro dentro de una categoría de gasto baja los gastos; un rescate de FCI baja lo invertido.
- **Invertido** es neto (suscripto − rescatado) y puede ser negativo. **Balance** = ingresos − gastos, con el % de ahorro.
- **Antes del 2026-09-13** se sumaba el monto sin mirar `t.type` y lo que no tenía categoría contaba como gasto:
  agosto 2026 mostraba Gastos 745 / Invertido 726 cuando eran 689 / −157.
- Vive en `FinanceScreen._SQL_GRUPO` / `_SQL_NETO` (`Modulos_Mysql.py`), compartida por `get_kpis`,
  `_get_categories_by_type` y `get_monthly_evolution`.

### Mapa de categorías (E2, 2026-09-13)

> Migración de datos en BD, sin cambios de código. Respaldo previo en `fin_categories_bak_20260913`,
> `fin_txn_cat_bak_20260913` y `fin_rules_cat_bak_20260913`.

**Estructura de `fin_categories`:**
- `category_type` decide el grupo de los KPIs: `expense` / `income` / `transfer` / `investment`. Las
  transferencias no suman ni en ingresos ni en gastos.
- `expense_class` (nueva, solo gastos): `fixed` + `variable` = costo de vida base del número de libertad
  financiera; `extraordinary` se promedia aparte. NULL en los demás tipos. El comentario de la columna en
  MySQL repite la regla.
- Se borró la columna `type`: nada la leía y `uq_cat_name (name, type)` permitía repetir un nombre. Hoy
  `uq_cat_name (name)`.
- `parent_id` quedó en NULL en todas: el mapa es plano. La columna sigue existiendo.

**Mapa vigente (35 categorías; Mudanza se sumó el 2026-09-17 y las tres `transfer` de liquidez el
2026-09-21, ver las revisiones de esas fechas):**

| Tipo | Clase | Categorías |
|------|-------|-----------|
| expense | fixed | Alquiler, Servicios, Suscripciones, Expensas, Familia, Cuotas de préstamo, Intereses y financiación, Banco e impuestos |
| expense | variable | Supermercado, Restaurantes, Transporte, Salud, Médico, Hogar, Ropa y compras personales, Otros gastos |
| expense | extraordinary | Tecnología, Viajes, Mantenimiento, Cancelación de préstamo, Mudanza |
| income | — | Sueldo, Reintegros y bonificaciones, Intereses ganados, Otros ingresos |
| transfer | — | Pago tarjeta crédito, Entre cuentas propias, Cambio de moneda, Préstamo recibido, Liquidez FCI, Compra de USD, Deposito garantia alquiler |
| investment | — | Aportes a IB, Retención USDT, ~~Inversiones~~ (vaciada el 2026-09-21: 1 fila de 0,00) |

Médico, Mantenimiento, Viajes y Otros ingresos no tienen filas: quedan por decisión del usuario.

**Categorías fundidas:**

| Antes | Ahora | Nota |
|-------|-------|------|
| Comisiones bancarias + Impuestos bancarios | Banco e impuestos | |
| Gastos Caracas + Gastos Tucacas | Expensas | Expensas y mantenimiento de las propiedades en Venezuela. Se pierde la separación entre las dos |
| Mesada Daniel A + Remesa Pay | Familia | Remesa Pay = Binance Pay a Paredes Leonel |
| Ropa y calzado + Shopping + Deportes | Ropa y compras personales | |
| Taxi/Uber/Cabify | Transporte | |
| Transferencia entre cuentas + Interno Citi | Entre cuentas propias | |
| Compra USDT + Ingreso USDT + Venta USDT USD + Remesa VES | Cambio de moneda | Incluye los Zelle recibidos en Citi: son USDT→USD para invertir o pesos→USD que manda un tercero (normalmente Nelson Ruiz), no ingreso |

**Renombres:** Freelance/Honorarios → Sueldo · Bonificaciones → Reintegros y bonificaciones · Cuotas crédito →
Cuotas de préstamo · Intereses cobrados → Intereses y financiación · Transferencia IB → Aportes a IB.

**Filas que cambiaron de categoría una por una:**
- **Financiación** (dato pendiente de E1) era `income` y mezclaba el préstamo con sus pagos: el desembolso
  inflaba los ingresos de mayo. Hoy el desembolso es Préstamo recibido (`transfer`), las cuotas van a Cuotas
  de préstamo y la cancelación a la categoría nueva Cancelación de préstamo (`extraordinary`).
- **Planes de tarjeta** (VISA PLAN V, CUOTIFICACION, FINANC DE SALDO, EXPR.PLAN AMEX) y sus abonos `CR.$ PLAN V`
  / `CR.$ CUOTIFICACION` van juntos a Intereses y financiación: en la misma categoría el capital se netea y no
  se cuenta dos veces.
- 4 filas de Bonificaciones pasaron a Intereses ganados.

**Cifras:** 11 categorías renombradas, 1 creada y 11 borradas; 79 filas y 32 reglas reasignadas; filas sin
categoría 49 → 32. KPIs 2026 verificados contra la BD después de migrar:

| Mes | Ingresos antes → después | Gastos antes → después |
|-----|--------------------------|------------------------|
| 2026-02 | 3.279,69 (sin cambio) | 2.258,55 → 1.852,45 |
| 2026-05 | 6.822,64 → 2.665,71 | 451,78 → 2.407,36 |
| 2026-06 | 3.916,50 → 4.423,11 | 293,64 → 800,25 |
| **Total 2026** | **25.785,99 → 22.135,66** | **8.657,62 → 10.713,70** |

**Pendiente de E2 (código):**
- Las reglas comparan contra un texto distinto del que se ve (sufijo de origen ` — 100`): 38 reglas sin aciertos.
- Debe ganar la regla más específica: hoy #269 y #320 quedan tapadas por otras. *(2026-09-14: la #320 se desactivó
  en el paso 4 y las contrapartes que tapaba tienen regla propia a prioridad 30; la #269 sigue igual.)*
- La aplicación retroactiva solo debe completar filas sin categoría, sin pisar las clasificadas.
- Reglas genéricas para los planes de tarjeta.
- El popup de categorías no muestra ni edita `expense_class`.
- Las cuotas de tarjeta se importan con la fecha de la compra original: CUOTIFICACION 1-06…5-06 viene de cinco
  extractos distintos y las cinco filas quedan el 2026-02-07.
- Re-aplicar las reglas una vez corregidas (escritura en BD, con autorización).

**Revisión del 2026-09-21 — mover plata de un bolsillo a otro no es invertir.**

El KPI "Invertido" daba negativo en julio y agosto. No era un bug: la categoría `Inversiones`
(`investment`) juntaba tres cosas que no son inversión, y el rescate de FCI entraba como `income` del
grupo, restando.

| Familia | Movs | Sale | Entra | Qué es |
|---|---|---|---|---|
| FCI Santander — suscripción / rescate | 76 | 10.180,96 | 6.539,26 | caja de ahorro de facto |
| FCI BBVA — oper. fondo común | 58 | 3.913,99 | 3.676,37 | rotación pura |
| Títulos y liquidaciones | 13 | 251,47 | 180,24 | compra de USD para pagar la tarjeta |
| Títulos BBVA del 15/04 | 2 | 713,82 | — | depósito de garantía del alquiler |
| `COMPENSACION DE FONDOS` | 1 | 0,00 | — | lo único que sigue en `Inversiones` |

Tres categorías nuevas, todas `transfer`: **Liquidez FCI** (134 filas), **Compra de USD** (13) y
**Deposito garantia alquiler** (2). La inversión real vive en `Aportes a IB` (2.450,00) y
`Retención USDT` (2.681,97).

**Invertido: 9.596 → 5.132 sobre el total histórico de las filas, no sobre el año** — corregido el
2026-09-21 al cruzar contra los extractos. Los 2.450 de `Aportes a IB` son las 9 filas de todos los años;
en 2026 hay **2.250**, así que el período 2026 queda en **4.931,97** (IB 2.250 + Binance 2.681,97). El
"antes" con ese mismo recorte está sin medir: las filas ya cambiaron de categoría, y reconstruirlo es el
neto 2026 de las seis categorías involucradas. Julio pasa a +126,72 y agosto a +258,31 — el panel muestra
**un mes**, no el año (`_period()` en `Class_Finance.py`), y esos dos meses son `Retención USDT` sola.
El ahorro, el costo de vida y el número de libertad **no se mueven**: `ahorro = ingreso − gasto` y el
grupo `investment` no participa de ninguno de los tres.

**Cuadra contra los extractos en la única mitad donde puede cuadrar.** `Aportes a IB` da 2.250 en 2026
contra 2.250 de `extractos.depositos` de `U4214563`, **mes por mes**: 400 en enero, 250 en febrero, 750 en
mayo, 850 en junio. IB es el único vehículo donde esa columna es flujo de dinero real — sale de
`Deposits & Withdrawals` del CSV filtrado por `Electronic Fund Transfer` (`Class_DataFrame.py:771`). En
los otros tres significa otra cosa y el cruce no aplica: Crypto escribe las compras (`producto` con
`codigo='O'`), BotCrypto un `0.0` fijo y FCI las compras divididas por `factor_cambio`. Es el BACKLOG 90.
Por eso `Retención USDT` (2.681,97) **no tiene contraparte medible**, y que el extracto de `B0000001` dé
2.698,58 no la valida: son compras, no transferencias.

Dato al pasar, no un error de datos: **IB no recibe un aporte desde junio.**

Por qué el FCI es `transfer` y no `investment`: el usuario intentó tenerlo en un fondo aparte y se confundía
al regularizar —sacaba o metía donde no era—, así que dejó uno solo, y los movimientos que se ven son los de
cuando toca pagar el alquiler o cubrir un gasto. Ese gasto ya se registra como gasto cuando se paga; contar
además el rescate como desinversión lo cuenta dos veces. La compra de USD tampoco empobrece: el gasto es el
consumo de la tarjeta, que el resumen ya trae.

El depósito de garantía va en categoría propia porque **vuelve** al terminar el contrato. No es gasto hoy; el
día que no lo devuelvan, ese día sí lo es, y por eso tiene que quedar identificable para cruzarlo.

**Efecto lateral: los 3.641,70 USD que quedaron dentro del FCI de Santander desaparecieron del panel.** Antes
se veían inflando "Invertido" —mal, pero se veían—. Hoy "Invertido" dice la verdad y el saldo del fondo no
tiene dónde mostrarse, porque `fin_accounts.balance` sigue en 0 en las 10 cuentas. Eso le sube la prioridad al
paso **1.6**: sin saldos, la plata que está en el fondo es invisible.

**Reglas de import.** Reapuntadas a las categorías nuevas: 252, 309 y 350 (FCI); 68, 69, 158, 264 y 515 (compra
de USD). Borradas 7 — `TEST_PATRON_NO_EXISTE` (365), las duplicadas 253 y 304, y cuatro (356, 357, 449, 450)
cuyo patrón era el número de comprobante completo: es único por operación, así que no podían volver a coincidir
nunca.

**El instalador puede deshacer esto, y además está roto.** `insert_rules()` de `AppTest/run_finanzas_setup.py`
hace `ON DUPLICATE KEY UPDATE category_id = VALUES(category_id), is_active = 1` sobre
`uq_rule (pattern, match_type)`: correrlo devuelve las reglas 68 y 69 a `Inversiones` y las reactiva. Hoy no
llega a ejecutarse, pero por accidente del orden —`insert_categories()` inserta en la columna `type`, borrada el
2026-09-13, y aborta con error 1054—. Eso mismo significa que **el instalador no puede crear el módulo Finanzas
en una versión hijo**: su seed de categorías quedó en el mapa anterior a las fusiones. Pendiente de E2.

**`amount_usdt` no usa el tipo de cambio de la operación.** La compra del 15/04 fueron 1.048.480,49 ARS que el
módulo convirtió a 713,75 USD — rate implícito 1.469. Si de ahí salieron los 750 USD del depósito, el MEP
efectivo fue 1.398: un 5% de diferencia. Sin confirmar contra el contrato, pero si el rate de `amount_usdt` no
es el de la operación, todos los montos en USD del panel arrastran ese sesgo.

**Revisión del 2026-09-13 — mesadas de Daniel y Yolemi.**

*Datos corregidos* (SQL ejecutado por el usuario): reglas #494 `YOLEMI ALBARRACIN` y #495 `0960311796` (cuenta de
Daniel) → Familia, prioridad 30. Se desactivaron #315 y #380, que nunca acertaron. 13 filas pasaron de Cambio de
moneda a Familia (8 de Yolemi, 5 de Daniel) y la fila 2170 a mano (`manual`, ver "Santander — el detalle es la
primera línea sin monto"). Daniel tiene su mesada de 200.000 ARS de abril a julio; agosto entra con el próximo
extracto de Santander.

*Pedidos del usuario:*
1. Un ajuste de categoría tiene que replicarse en todo movimiento que cumpla la misma condición.
2. Una opción global para ver qué cae en cada categoría y ajustarlo desde ahí.
3. En la barra de categorías no se distingue que la fila se puede cliquear.

*Por qué hoy un ajuste no se replica* — tres fallas entre `_CategoryEditPopup` y `save_rule`:
- **(a) Patrón precargado imposible.** Es la descripción completa con el sufijo ` — detalle`, pero `apply_rules`
  compara `concepto + " " + detalle`, sin la raya. #315 y #380 eran de ese tipo: 0 aciertos.
- **(b) Hacia atrás no mira el detalle.** La aplicación retroactiva (`save_rule`) y la vista previa
  (`save_rule_count_retro`) comparan solo `raw_description`. Una regla por cuenta de destino clasifica lo nuevo y
  no encuentra nada de lo ya cargado. Solo el adapter de Santander pasa el detalle a `apply_rules`; BBVA no trae
  destinatario en el extracto.
- **(c) Empata con la #320.** Las reglas del popup se guardan con prioridad 50, la misma que `TRANSFERENCIA` →
  Cambio de moneda; con `ORDER BY priority, id` gana la #320 por id menor. Complementa "Debe ganar la regla más
  específica". Desde el 2026-09-14 la #320 está inactiva (paso 4), pero el empate sigue valiendo contra cualquier
  regla genérica de prioridad 50.

*Mejoras recomendadas, en el orden acordado:*
1. ~~Datos de Daniel y Yolemi~~ — hecho (arriba).
2. **Popup de categoría** (commit chico): sugerir una clave corta (cuenta de destino o nombre, no la descripción
   entera), aplicar hacia atrás y previsualizar mirando también el detalle, y guardar las reglas del usuario con
   prioridad 30.
3. **Pantalla "Revisión de categorías"** (pedido 2): vista global por categoría y, dentro, agrupada por destinatario
   o cuenta, con cantidad, total y la regla que clasificó el grupo. El error casi nunca es un movimiento suelto sino
   un destinatario entero: reasignar el grupo crea la regla con la clave corta y la aplica hacia atrás. En
   `_CategoryBar` (pedido 3): cursor de mano, "›" al final de cada fila y color de fondo al pasar el mouse.
4. ~~Revisar la #320 `TRANSFERENCIA`~~ — hecho el 2026-09-14 (SQL ejecutado por el usuario). Decía "todo lo que
   contenga TRANSFERENCIA es Cambio de moneda" y ganaba siempre: tapaba 156 filas, entre ellas cinco meses de alquiler
   y dos expensas que no contaban como gasto. Se desarmó por contraparte, sin esperar la vista del paso 3:
   - **Reglas nuevas, prioridad 30:** #497 `2 3959239959` (CUIL propio en el detalle de Santander) → Entre cuentas
     propias; #498 `2 0182540472` (Sergio Ariel di Sabato; el alquiler desde abril, cuando termina Avellaneda Pet
     Mall) → Alquiler; #499 `3 0716676737` (Fideicomiso 25 de Mayo) → Expensas; #500 regex
     `^TRANSFERENCIA( [(][0-9]+[)])?$` → Entre cuentas propias. La #500 es para BBVA 196-009369/5, cuyo extracto no
     trae contraparte: 20 de las 21 "TRANSFERENCIA | 100" tienen su par exacto (mismo monto, ±3 días) entrando en
     Santander desde el CUIL propio. No coincide con COE, CAP/CCP, TC ni Santander.
   - **Desactivadas:** #320; #62-65 (`TRANSFERENCIA RECIBIDA/REALIZADA/INMEDIATA/NO GRAVADA` → Entre cuentas propias,
     prioridad 100 — sin la #320 hacían lo mismo); #393, #396, #399 y #491 (patrón con raya, reemplazadas).
   - **Filas, por id y solo las que estaban en Cambio de moneda por regla:** 139 → Entre cuentas propias (84 de
     Santander con el CUIL propio, 35 BBVA "TRANSFERENCIA" y 20 BBVA COE, que ahora toma la #323), 6 → Alquiler,
     2 → Expensas.
   - **Gastos USDT, todas las cuentas:** abr 2.616 → 3.714, may 4.116 → 4.676, jun 2.283 → 2.853, jul 2.563 → 3.041,
     ago 1.305 → 1.748. Ingresos sin cambio.
   - **Criterio que queda:** no hay regla genérica para transferencias. Una transferencia a alguien nuevo queda sin
     categoría y cuenta como gasto hasta clasificarla, en vez de esconderse como transferencia. Hasta que se arregle
     el popup (paso 2), clasificarla desde ahí no crea una regla que acierte.
   - **Las 9 filas que quedaron en Cambio de moneda sin regla**, decididas una por una con el usuario el 2026-09-14:
     - Brian Nahuel Villan (2215), reintegro de un restaurante → Restaurantes. Ivess el Jumillano (2100), agua en
       bidones → Supermercado, con regla `3 0537882871` porque es un proveedor que se repite (por Mercado Pago ya
       caía en Supermercado con las #367 y #385). Ejecutado.
     - Nelson Juvenal Ruiz hace dos cosas: instaló el aire acondicionado y a veces le vende USD al usuario, que le
       paga en pesos y recibe por Zelle en Citi. La 2030 (11/05, 214.500) tiene su par exacto — Zelle de 150 USD de
       NELSON RUIZ el mismo día en Citi (fila 1919), 1.430 ARS/USD — así que es compra de dólares y se queda en
       Cambio de moneda. La 2073 (25/07, 370.000) fue el aire, confirmado por el usuario; no tiene Zelle en Citi,
       cargado hasta septiembre → Mantenimiento, ejecutado. Nelson queda sin regla: sus pagos se deciden cruzando
       contra un Zelle entrante del mismo día. Brian tampoco lleva regla: la próxima transferencia no tiene por qué ser
       lo mismo.
     - Promo de MODO (2314, +10.000 el 25/06) → Reintegros y bonificaciones, con regla #502 `PROMOMODO`: la #72
       `PROMO MODO` no la ve porque la transferencia escribe la palabra junta. Ejecutado.
     - La 2079 (100.000 el 29/07, "Transferencia no gravada") → Entre cuentas propias. Se importó antes de `ddb99fb`
       y quedó con el pie legal como detalle; las otras 15 "no gravadas" de Santander van todas a nombre del usuario.
       Con el parser corregido una fila así trae su detalle y la toma la #497. Ejecutado.
     - Las BBVA CAP chicas (1696, 1852, 1957) quedan como están.
   - **Hallazgo — las COE de BBVA 196-009369/5 son compras de USDT, no transferencias propias.** De 33, 24
     (9,76 M ARS) tienen una compra `C2C BUY ARS` en Binance por el mismo monto y ±2 días; 9 chicas (521.400) no
     tienen par. El usuario ya había pasado 1274 y 1275 a Cambio de moneda a mano en enero. La #323 las manda a Entre
     cuentas propias, y este paso movió ahí 20 de ellas: para los KPIs da lo mismo (las dos categorías son transfer),
     pero al netear transferencias propias (E3) esas filas no tendrían par. Reemplaza la duda anterior, "18 de las
     20 COE no tienen par en Santander": no lo tienen porque no van a Santander.
     *Decidido y ejecutado el 2026-09-14:* la #323 pasa a Cambio de moneda y las 22 COE con par P2P que estaban en
     Entre cuentas propias se mueven ahí; las 9 sin par se quedan. Seis COE coincidían además en monto con un ingreso
     propio y se revisaron una por una. En 1320, 1927 y 1716 ese ingreso de Santander (motivo "hab") tiene su par en
     un `HABERES` de BBVA del mismo día: la transferencia propia es el HABERES. En 1275 y 1279 el ingreso cae en
     tarjetas de crédito. El 29/01 salieron de BBVA dos 850.000, la COE 1262 y la `TRANSFERENCIA` 1263, clasificadas
     a mano al revés del patrón (TRANSFERENCIA → Santander, COE → P2P): se intercambian. Lo que queda de patrón para
     BBVA 196-009369/5: `TRANSFERENCIA` y `HABERES` van a Santander (Entre cuentas propias), `COE` a un vendedor P2P
     (Cambio de moneda).
   - Las reglas con raya 308, 311, 312, 313, 364, 394, 395, 434, 451 y 455 quedan para el paso 2.

**Revisión del 2026-09-17 — por qué los gastos de abril a julio casi igualan a los ingresos.**

Medido con un script de solo lectura que repite el cálculo del gráfico de evolución (`_SQL_NETO` sobre `amount_usdt`,
mes de cobro, `tracked_since`); los totales cuadran con el gráfico. Valores en USD, **antes** de mover filas a Mudanza:

| Mes | Ingresos | Gastos | Fijo | Variable | Extraord. |
|-----|----------|--------|------|----------|-----------|
| Ene | 3.575 | 1.747 | 1.040 | 415 | 292 |
| Feb | 3.280 | 2.088 | 1.620 | 455 | 0 |
| Mar | 2.586 | 2.206 | 1.744 | 463 | 0 |
| Abr | 3.000 | 2.972 | 2.261 | 712 | 0 |
| May | 2.680 | 2.611 | 1.891 | 677 | 27 |
| Jun | 4.440 | 2.523 | 1.953 | 560 | 0 |
| Jul | 2.751 | 3.374 | 2.456 | 550 | 242 |
| Ago | 2.834 | 1.950 | 1.442 | 494 | 0 |

- **No es efecto dólar.** La tasa se movió entre 1.460 y 1.580; el gasto en pesos pasó de 2,2–2,6 M (ene–mar) a
  3,2–4,6 M (abr–jul).
- **Abril es la mudanza** (confirmado por el usuario). Al firmar pagó la reserva de 750.000 (#1545, 01/04), 860.000 el
  17/04 (#1583) y 750 USD de garantía en efectivo. La reserva queda en Alquiler como el mes de abril. Los 860.000, el
  retiro de 200.000 del 16/04 (#1582) y la cerrajería del 17/04 (#1584) pasan a la categoría nueva **Mudanza**
  (`expense` / `extraordinary`, filas `manual`; SQL ejecutado y verificado por el usuario). El reparto entre alquiler
  y gastos de entrada es aproximado: no hay desglose del contrato. En abril el total no cambia; el fijo baja de 2.261 a
  1.676, el variable de 712 a ~565 y el extraordinario sube de 0 a ~732.
- **La garantía es inversión, no gasto.** Los 750 USD salieron de un dólar MEP del 15/04: compra de AL30 por
  1.046.021 (#1946, 1.048.480 con gastos) y venta por U$S 742,39 (la #1947, 100,86 ARS, lleva el número de esa venta).
  Para el usuario comprar dólares MEP es inversión y la #1946 queda en Inversiones: los dólares siguen siendo suyos y
  vuelven al terminar el contrato. Se descartó una categoría "Depósito en garantía" (`transfer`). Dos límites:
  `amount_usdt` convierte a la tasa USDT (~1.469) y da 714 USD en vez de 742; y la punta en dólares no está en la BD —
  **BBVA CA USD (id 7) está inactiva y sin filas**, y `BbvaArCuenta` guarda todo como ARS.
- **La mudanza se pagó con dos préstamos BBVA:** 1.800.000 el 15/04 (#1943) y 1.000.000 el 20/04 (#1952), en Préstamo
  recibido. Probable, no confirmado: sus cuotas son la suba de Cuotas de préstamo desde junio (débito
  `196-96-00127796`: 507, 421 y 421 USD en jun–ago, contra 194 del `00091422` en abril). Una cuota de 670.000–790.000
  es mucha para 2,8 M: puede juntar también el préstamo desembolsado en mayo (ver "Filas que cambiaron de categoría").
- **Lo que subió y se quedó:** Alquiler ~360 → ~460 desde mayo (di Sabato; hasta marzo, Avellaneda Pet Mall),
  Servicios ~27 → ~60 y Otros gastos ~80 → 160–250. Familia viene alta desde febrero (~780): no es de la mudanza.
- **No es suba real:** Intereses y financiación (el −228 de febrero baja el promedio de ene–mar) y Expensas (abril en 0
  y mayo en 137: un pago corrido de mes).
- **Julio:** Familia 986, el aire acondicionado de 370.000 (#2073, ya en Mantenimiento), 126 USD sin categoría y los dos
  cierres de BBVA (las 5 cuotas del 182, ~90.700 ARS).
- **Agosto (1.950) probablemente está incompleto:** Expensas en 0 y Servicios en 21.

Queda abierto: los 126 USD sin categoría de julio; qué préstamo es el `00127796`; y cargar BBVA CA USD, que pide un
lector que distinga la moneda.

### Extractos y cargas automáticas (2026-09-13)

**Carpeta vigilada.** `Agente_ExtractosWatcher` pasó de la lista inline a un thread propio de
`register_threads()`, con el patrón de los demás agentes lentos. Cada adapter extrae las filas
**antes** de registrar el import; un PDF reconocido que no da ninguna fila ya no deja un import vacío en
`fin_statement_imports` y va a `desconocidos/` para revisarlo, en lugar de borrarse como si se hubiera cargado.
Los 71 imports vacíos que dejó el comportamiento anterior siguen en la tabla (limpieza pendiente, con autorización).

**Banesco (cuenta 35, `BANESCO-0611`).** Reglas 475-483:

| Texto al inicio | Categoría |
|-----------------|-----------|
| `COM,TRF.CR.INM`, `COM,SERV.M`, `ComisionServicioSMS`, `EMISIONEDOCUENTA`, `EMISIONDEESTADODECUENTA` | Banco e impuestos |
| `NETUNOREC,ENLINEA` (internet), `CORPOELECREC,ENLINEA` (electricidad) | Servicios |
| `TRANS,CTAS`, `TRFCRINM` | Expensas |

Los abonos `TRANS,CTAS` de terceros son ventas de USDT (decisión del usuario) y quedan en Cambio de moneda. Las
tres transferencias a Luz (V-12.357.122, 2026-06-30) son un caso puntual: se pasaron a mano a Familia, sin regla
propia. Una transferencia futura a esa cédula caería en Expensas por la #483.

**Santander.** También se detecta por el CBU de la cuenta (`0720175888000037071990`), para los PDF donde la
palabra "Santander" no aparece entre las primeras 200 palabras que mira `detect_adapter()`.

**Santander — el detalle es la primera línea sin monto.** La línea que sigue a un movimiento trae la contraparte
(destinatario y cuenta, comercio y tarjeta, fondo o período) y ocupa siempre una sola línea: verificado sobre las 264
filas de la cuenta en pesos. Hasta el 2026-09-13 cada línea sin monto pisaba la anterior, así que un movimiento que
cerraba la hoja quedaba con la última línea del pie legal ("que utilicen la marca Santander.") y la regla por cuenta
de destino no lo veía. Caso testigo: la mesada de Daniel del 2026-06-26 (fila 2170) cayó en Cambio de moneda por la
#320. Hoy vale solo la primera, en pesos y en dólares; la tarjeta (`_process_tc_line`) sigue como antes. Las filas
1981, 2251 y 2079 conservan el pie como detalle — su categoría sale del concepto y no cambia. La deduplicación no
mira el detalle, así que un extracto superpuesto no duplica filas.

**Binance — tres fuentes para la cuenta `BINANCE-USDT`:**

| Fuente | Sección | Clase | Comprobante |
|--------|---------|-------|-------------|
| PDF "Historial de órdenes C2C" (carpeta vigilada) | `binance_c2c` | `BinanceC2c` | número de orden |
| API C2C `/sapi/v1/c2c/orderMatch/listUserOrderHistory` | `binance_c2c` | `BinanceC2cApi` | `orderNumber` = número de orden del PDF |
| API Pay `/sapi/v1/pay/transactions` (solo egresos) | `binance_pay` | `BinancePay` | `transactionId` |

- **PDF C2C.** Se detecta por "Historial de órdenes C2C". Binance cambió el año de la hora de `26-03-28` a
  `2026-03-28`; la regex acepta los dos. La cuenta sale de la regla de detección, como en los demás adapters.
- **API C2C.** Solo órdenes `COMPLETED` (la API también devuelve las canceladas), BUY y SELL, de a 100 por página.
  Hereda `_build_transactions` del PDF: neto = `amount` − `takerCommission` en compras y + en ventas. Verificado el
  2026-09-13 contra lo cargado por PDF: 58 órdenes desde 2026-03-01, 0 diferencias en fecha, tipo, neto y descripción.
  Las dos fuentes comparten comprobante, así que se deduplican entre sí.
- **API Pay.** Usaba `startTimestamp`/`endTimestamp`: Binance ignora esos nombres y devuelve los últimos 90 días.
  Hoy manda `startTime`/`endTime` en tramos de hasta 90 días (un rango mayor da `403004`). Ya no descarta un rango
  "ya importado": lo deduplica el comprobante.
- **Carga compartida** `_load_api_rows()`. Una API se vuelve a consultar sobre la misma ventana todos los días,
  así que primero se descartan los comprobantes ya cargados — antes de clasificar, porque `apply_rules` suma
  `hit_count`. El import se registra **solo si hay filas nuevas**, con el SHA-256 de esos comprobantes como
  `file_hash`; si una corrida registró el import y cortó antes de insertar, el mismo hash lo reutiliza. Una fila
  que el índice único descarta en silencio (`INSERT IGNORE`) cuenta como error y se loguea.
- **Automatización.** `sync_binance_api()` corre desde `Agente_BinanceApiSync` cada 24 h. Cada fuente arranca 3 días
  antes de su último movimiento cargado (cubre cortes y el desfase UTC / hora local) o 90 días atrás si no tiene
  ninguno. La fila sintética `BINANCE-INV-YYYY-MM` no tiene import y no adelanta la ventana. Una fuente que falla
  no frena a la otra. Primera verificación: 0 filas nuevas en las dos ventanas — todo estaba cargado por PDF.
  **Desde acá el PDF de C2C ya no hace falta.**

**1.7 — ABIERTO, anotado el 2026-09-19: la rotación entre vehículos no se ve, y nadie concilia contra el extracto
del vehículo.**

Pedido textual, después de mirar la vista `Flujo`: *"ahora sobre las inversiones. A ver, debemos conciliar con los
extractos mensuales. Hay dinero que reinvierto, saco de FCI y puedo llevar a IB o cripto, eso no se está viendo"*.

**Qué muestra hoy la vista y por qué no alcanza.** El grupo `investment` tiene tres categorías —`Inversiones`,
`Aportes a IB`, `Retención USDT`— y la vista muestra el **neto** de cada una. Un rescate de FCI entra como `income`
(negativo con la convención de signo) y el aporte a IB o a cripto sale como `expense` (positivo): los dos tramos se
cancelan y el mes queda en ~0. Es exactamente lo que se ve en la foto del 2026-09-19: `Inversiones` **−508** en 1M y
**−216** en 3M contra **+635** en 1Y. Leído sin contexto parece que se desinvirtió; probablemente —**no
confirmado**— es una rotación de FCI hacia IB o cripto, donde el patrimonio no cambió y solo cambió de lugar.

**Lo que falta no es un número, es el hecho.** Hoy quedan registrados los dos tramos por separado y en ninguna parte
queda que fueron **el mismo movimiento**: salió de un vehículo y entró en otro. Sin eso no se puede contestar "cuánto
invertí de verdad este mes" ni "cuánto roté", que son dos preguntas distintas.

**Por qué esto le pone un techo al progreso del número (4.4).** El objetivo se calcula con el gasto; el **avance** se
calcula con el patrimonio, y el patrimonio invertido vive en la tabla `inversion` (vía `get_totales_inversiones()`), no
en `fin_transactions`. *Actualizado el 2026-09-19:* el 4.4 se cerró **cruzando las dos** desde el panel —
`FinancePanel._freedom_progress()` — así que el avance ya se muestra (8,3%). Lo que sigue faltando es la conciliación:
el cruce de hoy **suma saldos**, no verifica que cada peso que salió del banco haya llegado al vehículo. Ese chequeo es
lo que permite detectar el caso que más importa: plata que
salió de la cuenta bancaria hacia un vehículo y **no** apareció en el vehículo, o un aporte registrado en el vehículo
que no salió de ninguna cuenta.

**Posible bloqueador de datos, sin verificar:** `Aportes a IB` y `Retención USDT` nombran el destino, pero
`Inversiones` es un cajón único que mezcla FCI, títulos y el resto. Si el destino no está en el dato, la conciliación
por vehículo no se puede armar sin abrir esa categoría o agregar el vehículo a la fila. Hay que medirlo antes de
diseñar: cuántas filas tiene `Inversiones` y qué comercios/conceptos hay adentro.

**Chequeo previo obligatorio, también sin verificar:** que ningún rescate de inversión esté categorizado como
`income` de verdad (una categoría del grupo `income`, no del grupo `investment`). Si alguno lo está, **infla el
ingreso y por lo tanto la tasa de ahorro y el ahorro del mes** — el 22,7% del 1Y quedaría alto. El mapa dice que las
tres categorías de inversión son `investment`, así que en principio no pasa; lo que no está medido es si alguna fila
de rescate cayó en otra categoría por regla.

**Orden natural:** va detrás de 1.6. Son el mismo tipo de trabajo —conciliar contra una fuente externa— y 1.6 es el
más simple de los dos: el extracto bancario trae el saldo impreso, mientras que acá hay que cruzar dos tablas de dos
módulos distintos.

#### Paso 1.6 — saldos: la infraestructura y la captura (2026-09-21 → 2026-09-22)

Cerrado: el PDF importado **ya no se borra**. `scan_extractos()` hacía `os.remove(pdf_path)` en la rama de éxito y
movía a `desconocidos/` solo los que fallaban, así que `PROCESADOS_DIR` estaba declarado desde el primer día y nunca
se usó — un solo hit en todo el repo. Hoy las dos ramas archivan, y el PDF queda como única fuente de los saldos y de
cualquier dato que el parser todavía no lee.

Los archivados se renombran con la cuenta y el período que quedaron en BD para ese `file_hash` —
`Sant-CC-ARS_2026-08.pdf`—, no con el nombre que les puso el banco. El PDF multi-cuenta de Santander (trae cuenta y
tarjeta en el mismo archivo) cae al nombre del banco, `Santander_2026-08.pdf`, porque no hay una sola cuenta que lo
identifique; los 36 imports de la cuenta de débito, con `period_to` en NULL, quedan como `sin-periodo`. Si la consulta
falla se conserva el nombre original y se loguea: un nombre feo es mejor que uno inventado. Pedido del usuario en la
misma tanda — sin esto `procesados/` se vuelve inutilizable a los tres meses.

`fin_statement_imports` tiene ahora `balance_prev` y `balance_curr`, `decimal(18,2)` nullable. Van en la fila del
extracto y no en `fin_accounts` porque el saldo cambia con cada extracto; `fin_accounts.balance` queda para el saldo
**vigente**, el del último import, que es lo que el panel necesita y sigue en 0 en las 15 cuentas. No hizo falta tabla
nueva: `fin_transactions.import_id` ya tiene FK hacia el import, así que la conciliación es
`balance_prev + SUM(movimientos del import) = balance_curr`. **NULL no significa descuadre** — significa que el
extracto no publica saldo (Binance, tarjetas) o que el import es anterior a la captura.

**Bloqueado: los parsers no se pueden calibrar sin un PDF, y no queda ninguno.** Los borró el propio
`scan_extractos()`, que es justo lo que se arregló, con lo que el arreglo llega tarde para este paso.

> **Revertido el 2026-09-22.** No estaba vacío: `EXTRACTOS_DIR` sale de `APPOO_TMP`, que `.vscode/launch.json`
> apunta a `deploy\tmp`, no a `AppOO\tmp` — y ahí había **18 PDFs en `desconocidos/`**, ninguno en `procesados/`.
> El paso 3 sigue sin material para tres de las cuatro clases de cuenta (BBVA, Santander, Citi), pero ya lo tiene
> para `BanescoVes` (los 3 de Banesco traen `SALDO MES ANTERIOR` con importe) y para las tarjetas, que sí publican
> saldo — ver la corrección del párrafo siguiente. Las coordenadas
de columna están calibradas contra el PDF real de cada banco (`X_SALDO_MIN = 515` en BBVA) y la línea `SALDO ANTERIOR`
**no tiene fecha**, así que su monto puede caer en la columna de saldo o en la de crédito. Escribir los cuatro parsers
suponiendo eso garantiza reescritura. Queda `AppTest/run_saldo_probe.py` como instrumento: imprime las líneas con
`SALDO` y las últimas del detalle con la coordenada X de cada palabra, sin tocar BD ni mover archivos.

**El paso que sigue son cuatro clases, no seis.** El cierre del import está copiado nueve veces —un `SELECT id` +
`INSERT` + `UPDATE ... status='processed'` por adaptador—, pero las cuentas con saldo real se concentran en cuatro:
`BbvaArCuenta` (de la que hereda `BbvaArAhorro`), `SantanderAr`, `CitibankUs` y `BdvVes` (de la que hereda
`BanescoVes`). Los tres adaptadores de Binance van en NULL a propósito: no hay extracto con saldo.

**Corrección del 2026-09-22: las tarjetas no van en NULL.** Acá se escribió que "la tarjeta publica el total del
resumen, que es otro concepto". Medido sobre los resúmenes reales, publica el saldo y encima concilia:

| Resumen | `balance_prev` | `balance_curr` |
|---|---|---|
| BBVA Visa (cierre 27-Ago-26) | `SALDO ANTERIOR 328.810,27 / 0,00` | `SALDO ACTUAL 255.102,61 / 0,88` |
| Santander Visa y Amex | `SALDO ANTERIOR 137.115,68 / 22,99` | **NULL** — el recuadro `SALDO ACTUAL $ U$S` de la copia informativa viene sin importe |

La conciliación del BBVA cierra exacta: `0` (saldo anterior cancelado con el pago de 328.810,27) `+ TOTAL CONSUMOS
254.702,92 + DB.RG 399,69 = 255.102,61`. Las dos líneas están hoy en `SKIP_MARKERS` y se descartan.

> **Revertido el 2026-09-22 — el `balance_curr` de las tarjetas Santander no es NULL.** El recuadro sin importe de la
> copia informativa existe, pero no es el único lugar donde aparece la etiqueta: el saldo de cierre está en la
> **última hoja**, en la línea que lleva `TNA` y `TEM` junto a `SALDO ACTUAL`. Capturado en **15 de 15** resúmenes y
> con la cadena entera cerrada (tabla en el paso 3, más abajo). La conclusión anterior salió de mirar solo la hoja 1.

Un detalle de implementación que el párrafo anterior no podía anticipar: en el BBVA el importe del saldo está en
**x = 444**, por debajo de `X_PESOS_MIN = 450`, así que `_classify_word` lo mandaría a la columna de cupón. El saldo
se lee por regex de la línea, no por columna.

En `BbvaArCuenta` la columna de saldo **ya se recolecta** en cada línea (`cols["saldo"]`, x ≥ 515) y se descarta sin
usarse; `SALDO ANTERIOR` está en `SKIP_MARKERS`, así que el `continue` la saltea antes de mirarla. `SALDO ACTUAL` no
figura en los marcadores de ese adaptador, de donde se sigue —**sin confirmar**— que el resumen no lo publica y que
`balance_curr` habrá que tomarlo del saldo de la última fila del detalle.

> **Revertido el 2026-09-22.** Sí lo publica: la redacción en la cuenta no es `SALDO ACTUAL` sino **`SALDO AL 28 DE
> ABRIL`**, con el día y el mes del cierre. Por eso no figuraba en los marcadores, y por eso la sospecha apuntaba a la
> última fila del detalle, que no hizo falta. Medido en los tres extractos de caja de ahorro; detalle en el paso 3.

**La cuenta de débito de Santander queda fuera de la conciliación** aunque se capture el saldo: sus 36 imports tienen
`period_from` y `period_to` en NULL, y sin período no se sabe a qué extracto pertenece cada uno. Está inactiva, así
que no es urgente.

#### La tabla de imports tenía 99 filas que ningún extracto trajo (2026-09-22)

Pedido del usuario, con el motivo: *"si arreglá eso, sino con el tiempo menos se entiende"*. `fin_statement_imports` es
donde uno va a mirar *"¿cargué el resumen de agosto?"*, y de las ~100 filas de las dos tarjetas de Santander **60 no
eran nada**: `status='processed'`, contadores en 0, período NULL, cero transacciones. Un PDF de AMEX generaba una fila
`santander_visa`.

**Los dos adaptadores multi-sección creaban la fila antes de saber si el PDF traía esa sección.** `SantanderAr`
(cuenta ARS, cuenta USD, visa, amex, débito) y `CitibankUs` (checking, savings) tienen el mismo bloque, copiado: el
primer bucle recorre `SECTION_NAME_MAP` e inserta el import de toda sección que tenga cuenta en `fin_accounts`. La
única guarda era sobre el **total** del PDF (`if not stats["rows_found"]`), que no dice nada de la sección. Después el
segundo bucle la marcaba `processed` con `row_count=0` y el período de `MIN(date)` sobre cero transacciones, es decir
NULL. Los adaptadores de una sola cuenta nunca tuvieron el problema: cortan con `if not raw_rows: return stats` antes
del `INSERT`.

El arreglo es un `continue` por sección sin filas, en las dos clases. Nada más cambia: el PDF sigue registrado como
importado en las secciones que sí cargaron, así que `uq_import (file_hash, section)` sigue frenando el reproceso.

**Medido sobre toda la tabla, no solo las tarjetas: 187 filas con período NULL, y son dos poblaciones distintas.**
Separarlas era el punto — la mitad es un defecto y la otra mitad es un hecho:

| | Filas | Qué son | Decisión |
|---|---|---|---|
| `row_count = 0`, `processed_count = 0` | **99** | la sección no venía en el PDF | borradas |
| `row_count > 0`, `processed_count = 0` | **88** | el PDF sí traía filas y **todas ya estaban en BD** | se conservan |

Las 88 son reintentos reales de cargar un extracto ya cargado, y son el registro de que ese PDF pasó. Su período queda
NULL por el mismo `MIN(date)`: mide las transacciones **de ese import**, y si todas se saltearon no hay ninguna. Que
quede NULL es correcto; lo que estaba mal era no poder distinguirlas de las 99.

Las 99 se reparten por sección así: `santander_debito` 36, `santander_cuenta_usd` 18, `santander_cuenta_ars` 14,
`santander_visa` 14, `santander_amex` 14, `citi_checking` 2, `bdv_cc` 1. El `DELETE` lleva `status = 'processed'` para
no pisar un import en vuelo, y dos `NOT EXISTS` redundantes contra `fin_transactions` y `fin_card_cycles` — las dos
únicas tablas que apuntan a un import. El chequeo previo de filas candidatas con algo colgando dio **vacío**.
Ejecutado el 2026-09-22: **99 filas**, el número exacto que había medido el control.

> **Las 36 de la cuenta de débito no eran lo que decía este documento.** Arriba quedó escrito que sus 36 imports con
> período NULL la dejaban fuera de la conciliación por ser datos sin fecha. Medido: **no tienen una sola
> transacción**. Eran las mismas filas fantasma del defecto de arriba. La cuenta sigue fuera de la conciliación, pero
> porque está inactiva y sin movimiento cargado, no porque le falte el período.

#### Paso 3 del 1.6 — los parsers capturan el saldo (2026-09-22 → 2026-09-24)

Una sola función de módulo, `parse_balances(lines, x_max)`, al lado de `parse_installments_due()`. Devuelve
`(balance_prev, balance_curr)` buscando dos etiquetas: `SALDO ANTERIOR` y, para el cierre, `SALDO ACTUAL` **o**
`SALDO AL <dd> DE <MES>` — las dos redacciones que usan los extractos reales.

**Lee por regex de la línea, no por columna.** El párrafo anterior lo anticipó desde el lado del BBVA (el importe cae
en `x = 444`, por debajo del umbral de la columna de pesos) y la medición lo confirma desde el otro lado: en la caja de
ahorro el saldo está en **x = 521**, que `_classify_word` manda a `saldo`, mientras que en el resumen de tarjeta la
misma etiqueta aparece dos veces en la misma línea, una por moneda. Una regla por columna necesitaría calibración
distinta por banco y por producto; la etiqueta es estable en los cuatro formatos medidos.

`x_max` resuelve las dos monedas: se toma el primer importe a la izquierda de ese x. Para la tarjeta vale 500 —el mismo
corte que `_classify_tc` ya usa entre pesos y dólares—, así que **se guarda el saldo en pesos y el de dólares no se
persiste**: la tarjeta es una sola fila en `fin_accounts` y no hay dónde ponerlo. Si hiciera falta, son dos cuentas
separadas, como ya existen para la cuenta ARS y la USD de Santander, no una columna más. La función también quita los
guiones bajos antes de mirar la línea, por lo mismo que `parse_installments_due`: el Visa los intercala entre los
dígitos del importe.

**Dónde quedó enganchada.** Tres de las cinco clases que cierran un import con saldo real:

| Clase | De dónde sale el saldo | Estado |
|---|---|---|
| `BbvaArCuenta` (y `BbvaArAhorro`, que hereda) | las líneas que `_extract_rows()` ya recorre — sin reabrir el PDF | verificado |
| `SantanderArTarjetaResumen` | `_extract_balances()`: hoja 1 + última hoja | verificado, 15/15 |
| `SantanderAr` (PDF unificado) | por sección, dentro del state machine de `_extract_all()` (`22da1b3`) | implementado — **sin verificar en BD** |
| `CitibankUs` | — | **sin material** |
| `BdvVes` (y `BanescoVes`) | `SALDO MES ANTERIOR`, con importe | **sin implementar** |

`CitibankUs` y `BdvVes` (con `BanescoVes`) siguen escribiendo NULL, que es la semántica ya definida: NULL es *el
extracto no publica saldo o el import es anterior a la captura*, nunca un descuadre. No se escriben a ciegas: de Citi
no hay material y el `SALDO MES ANTERIOR` del BDV sigue sin implementar.

**`SantanderAr` — el saldo se ata a su sección, y el signo no viaja en el importe** (`22da1b3`). El PDF unificado trae
las dos cuentas en un solo archivo, así que el saldo no puede salir de una función que mire el documento entero:
`_saldo_linea()` se llama desde el state machine de `_extract_all()`, con `state` diciendo a qué sección pertenece la
línea, y `_balances` se resetea ahí mismo para que una segunda pasada sobre el mismo PDF no arrastre el saldo de la
anterior. Toma el **último** importe de la línea y le pone el signo del `-$` que lo precede: el descubierto en pesos
publica el guion pegado al símbolo, así que **el signo no viaja en el token del importe** y leerlo con
`parse_amount_ar` a secas devuelve el descubierto en positivo. La línea `SALDO INICIAL` no lleva `continue` a
propósito — también fija la fecha que arrastran los movimientos sin fecha propia.

**El resumen de tarjeta cae a `TOTAL A PAGAR` cuando no publica `Saldo actual`.** Los resúmenes del formato nuevo
dejaron de titular el cierre, y `parse_balances()` devolvía `balance_curr` en NULL sobre un PDF que sí tiene el dato.
`_total_a_pagar()` lo busca **en todo el PDF** —la línea cae en una hoja del medio, no en la primera ni en la
última— y **solo para el cierre**: `Saldo anterior` también titula el saldo de la cuenta que paga el resumen, y esa
línea entraría como saldo de la tarjeta.

**La rama "ya importado" también completa el saldo.** `SantanderAr.load()` llamaba `_fill_missing_balances()` solo en
el camino de alta; hoy lo llama también cuando `uq_import` frena la sección, que es el caso de todo extracto cargado
antes de la captura. Re-soltar el PDF completa el saldo sin duplicar movimientos.

**Tarjetas — la cadena cierra sola.** El `balance_curr` de cada resumen es el `balance_prev` del siguiente, en las dos
tarjetas y sin un hueco:

| Visa 0925 | | Amex 9541 | |
|---|---|---|---|
| 347.708,66 → | 501.782,81 | 202.835,89 → | 205.801,10 |
| 501.782,81 → | 533.299,51 | 205.801,10 → | 232.260,89 |
| 533.299,51 → | 447.934,00 | 232.260,89 → | 180.420,40 |
| 447.934,00 → | 397.117,02 | 180.420,40 → | 61.963,25 |
| 397.117,02 → | 458.117,74 | 61.963,25 → | 80.537,13 |
| 458.117,74 → | 462.099,26 | 80.537,13 → | 137.115,68 |
| 462.099,26 → | 323.168,54 | 137.115,68 → | 258.140,96 |

Ocho PDF de Visa para siete eslabones: dos son el mismo cierre (19-Mar-26) con `file_hash` distinto, el duplicado ya
conocido. El Amex trae además saldo en dólares (22,99 / 27,99 / 32,99 / 49,99) que no se guarda; el Visa lo trae en
0,00.

**Caja de ahorro — la conciliación del paso 4 ya cuadra, sin haber escrito el paso 4.** Los tres extractos encadenan
entre meses y encima la aritmética cierra exacta contra los totales de movimientos que publica el propio PDF:

| Extracto | `balance_prev` | movimientos | `balance_curr` | |
|---|---|---|---|---|
| Abr-2026 | 2.795.199,26 | −5.919.670,67 +7.286.939,18 | 4.162.467,77 | exacto |
| May-2026 | **4.162.467,77** | −15.312.055,12 +13.054.047,42 | 1.904.460,07 | exacto |
| Jun-2026 | **1.904.460,07** | −9.714.622,79 +9.559.865,07 | 1.749.702,35 | exacto |

La línea `TOTAL SALDOS DISPONIBLES` del mismo PDF no contamina: dice "SALDOS", no `SALDO ANTERIOR` ni `SALDO AL`.

**Esos tres extractos son de BBVA, no de Santander.** El usuario los nombró como caja de ahorro de Santander; el PDF
dice `CBU 0170…` (017 es BBVA Argentina) y trae `REINTEGRO PROMO BBVA`. La cuenta es `196-009369/5`, que
`DETECTION_RULES` mapea a `bbva_ahorro` — correctamente, y con la regla `("Santander", "santander", None)` por delante
en la lista, así que el PDF no contiene la palabra. Importa porque decide en qué adaptador va el código: el saldo de la
caja de ahorro quedó en `BbvaArCuenta`, y el extracto de Santander que falta es el del PDF unificado.

> **Lo que el paso 3 no arregla hacia atrás.** El saldo se escribe en el `UPDATE` que cierra el import, así que los
> imports ya cargados se quedan en NULL: `uq_import (file_hash, section)` frena el reprocesamiento del mismo PDF, y el
> PDF de casi todos lo borró la versión de `scan_extractos()` anterior al paso 1. Los tres extractos de caja de ahorro
> medidos acá entraron así —el usuario los dejó en `extractos/`, la app los procesó con el código viejo y los borró—,
> y por eso ese camino se verificó contra las líneas medidas antes de que desaparecieran, no ejecutando el parser
> entero. Tienen saldo, de acá en adelante, los extractos nuevos y los quince resúmenes que siguen en `desconocidos/`.
> Para el resto hay que volver a bajar el PDF del homebanking.
>
> **Revertido el 2026-09-22 — re-soltar el PDF sí completa el saldo.** `uq_import` frena el alta, no al adaptador:
> `BbvaArTarjeta.load()` ya tenía una rama que sigue trabajando cuando el PDF está importado. Las tres clases con
> saldo completan hoy por `_fill_missing_balances()`. Ver "El saldo de la tarjeta BBVA, y por qué re-soltar el PDF
> alcanza". Sigue siendo cierto lo demás: hay que volver a bajar el PDF del homebanking.

#### El resumen de tarjeta cambió de formato en agosto de 2026, y el parser sigue leyendo solo el nuevo (2026-09-22)

Pedido del usuario: *"revisa el extracto de amex, cambia de formato en agosto, creo que los anteriores no te van a
servir"*. El hecho es cierto y la conclusión no: los quince resúmenes anteriores se leen enteros.

`SantanderArTarjetaResumen` (escrito el 2026-09-16, contra los resúmenes que había entonces) buscaba el ciclo en
**una línea con exactamente seis fechas `dd/mm/yy`** de la hoja 1. Los resúmenes de 2025 y del primer semestre de
2026 no tienen ninguna: **cero líneas con tres o más fechas `dd/mm/yy`** en los quince. Traen el ciclo en tres
líneas, con el mes en español y el año del cierre pegado a la etiqueta siguiente:

```
1824 LANUS OESTE   CIERRE 25 Jun 26VENCIMIENTO 07 Jul 26
Cierre Ant.: 21 May 26      Vto. Ant.: 02 Jun 26
Prox.Cierre: 23 Jul 26      Prox.Vto.: 04 Ago 26
```

Como `_extract_cycle()` devolvía `None`, `load()` salía con `rows_found = 0` y `process_pdf()` los mandaba a
`desconocidos/` **sin error**: el log dice "reconocido pero sin filas extraídas" y la detección por contenido había
funcionado bien (`1067995586` → Visa, `1068053384` → Amex). Los quince estaban identificados y descartados.

**El soporte del formato viejo no está en el parser.** Decisión del usuario: *"leelo de manera temporal ya que el
formato que queda es de agosto"*. El formato anterior no vuelve a aparecer nunca — todo resumen futuro trae las seis
fechas —, así que dejarlo en `SantanderArTarjetaResumen` sería cargar para siempre una rama que corre una vez. Vive en
`AppTest/run_carga_resumenes_viejos.py`, que **subclasea el adaptador y solo reemplaza `_extract_cycle()`**: escribe
por el `load()` real, así que respeta `uq_import`, el `INSERT IGNORE` de `fin_card_cycles` y el recálculo de
`billing_date`. `Class_Finance.py` quedó sin un solo cambio. Cuando los quince estén cargados, el script se borra.

El script se comitea igual que cualquier otro de `AppTest/` —el directorio está versionado, 59 archivos— y se borra
en su propio commit cuando los quince estén cargados. Queda el registro de cómo se hizo la carga; el hallazgo se
documenta acá con el detalle suficiente para rehacerlo si hiciera falta.

Lo que el script hace distinto del parser, y por qué:

- **Lee cada etiqueta de la primera línea donde aparece.** `CIERRE` y `VENCIMIENTO` se buscan en mayúsculas a
  propósito: en minúsculas colisionan con `Prox.Cierre:` y con `Vto. Ant.:`, que son otras dos fechas. La fecha se
  toma después de la etiqueta, no al principio de la línea, porque las tres líneas llevan dos fechas cada una.
- **Lee las dos primeras hojas**, porque las cuotas a vencer están en la hoja 2. **Una hoja a la vez, nunca
  concatenadas**: las coordenadas `top` arrancan de cero en cada hoja, así que un `lines` concatenado cruza la fila
  de meses de una con los importes de la otra (medido: da un falso positivo de una cuota en el Visa de diciembre).
- **Quita los guiones bajos del importe.** El Visa los intercala entre los dígitos: `_$_7_7_._8_5_0_,_8_5_` es una
  sola palabra para pdfplumber. Se limpian sobre una copia de las palabras y se llama al `parse_installments_due()`
  de producción, sin duplicar su lógica. Sin esto, diez de los quince tenían la fila de meses y ningún importe
  debajo, y parecía que no había cuotas.
- **Chequea el cierre contra `fin_card_cycles` antes de insertar.** Dos de los PDF son el mismo resumen (cierre
  19-Mar-26) con distinto `file_hash`, así que `uq_import` no los frena. No se confía en el `INSERT IGNORE` de
  `_save_card_cycle()`: no está verificado que `fin_card_cycles` tenga un UNIQUE sobre `(account_id, closing_date)`,
  y sin él ese `IGNORE` no atrapa nada.

Resultado medido en modo lectura sobre los dieciocho PDF de `desconocidos/`:

| | Visa (0925) | Amex (9541) | Banesco |
|---|---|---|---|
| PDF | 8 (uno duplicado: cierre 19-Mar-26) | 7 | 3 |
| Ciclo extraído | 8/8 | 7/7 | — omitidos, no son tarjeta |
| Cuotas a vencer | 6/8 | 5/7 | — |

Los cuatro sin cuotas no son un fallo del parser: tres Amex (feb, mar, abr 2026) **no traen el bloque** "Cuotas a
vencer" en ninguna hoja, y el Visa de diciembre 2025 lo trae con los seis meses y **sin la fila de importes** — 16
caracteres en la franja, `SALDO ACTUAL$U$S`, nada más. La serie de cierres queda completa y encadenada de
**20-Nov-25 a 25-Jun-26** en las dos tarjetas: el `prev_closing` de cada resumen es el `closing` del anterior.

Importa más allá del ciclo: `fin_card_cycles` es lo que `_set_billing_dates()` usa para decidir en qué resumen se
cobra cada cuota. Sin estos quince, las cuotas del primer semestre caen en el mes de la compra y el gasto de ese mes
queda inflado — el mismo problema que `billing_date` existe para resolver, en el tramo donde todavía no había ciclo.

> **Abierto, no medido:** en el formato de agosto las cuotas a vencer podrían estar también en la hoja 2, y el parser
> de producción lee solo `pages[0]`. No hay ningún resumen del formato nuevo en disco para verificarlo — los borraba
> `scan_extractos()`. Se resuelve mirando `fin_card_installments_due` de los imports de agosto y septiembre, o con el
> próximo resumen que llegue. Si estuvieran en la hoja 2, el parser nunca leyó una cuota a vencer.

#### El saldo de la tarjeta BBVA, y por qué re-soltar el PDF alcanza (2026-09-22)

Dos huecos del paso 3, encontrados al contestar qué meses de BBVA hay que soltar.

**1. `BbvaArTarjeta` había quedado afuera.** El enganche del paso 3 cubrió `BbvaArCuenta` (con `BbvaArAhorro`, que
hereda) y `SantanderArTarjetaResumen`; la tarjeta BBVA —`bbva_tc`, dieciséis de los veinticinco imports de BBVA— no.
Era el caso ya medido más arriba en este doc (`SALDO ANTERIOR 328.810,27`, `SALDO ACTUAL 255.102,61`, conciliación
exacta), así que el hueco era solo de código.

Captura con `x_max = X_DOLARES_MIN` (530): el resumen trae las dos monedas en la misma línea y el saldo en dólares no
tiene dónde ir, porque la tarjeta es una sola cuenta en `fin_accounts`. El límite es defensa, no mecanismo — las
palabras vienen ordenadas por `x0`, así que el primer importe ya es el de pesos. Si el de pesos cayera pasado 530
devuelve NULL, que es la respuesta correcta: NULL es "no publica saldo", nunca un descuadre.

> **Verificado solo en parte.** No hay ningún PDF de BBVA en disco (`desconocidos/` tiene dieciocho: quince de
> Santander y tres de Banesco), así que el reparto `prev`/`curr` se probó sobre las líneas transcriptas en este doc,
> con el `x0` **supuesto** según las franjas de columna de la clase. Falta confirmar la coordenada real, y la confirma
> `AppTest/run_saldo_probe.py` sobre un resumen cualquiera antes de soltar los dieciséis.

**`AppTest/run_saldo_probe.py` no existía.** Este doc lo citaba dos veces como instrumento disponible y
`MEMORY.md` como el paso que desbloquea el 1.6; nunca se escribió. Creado el 2026-09-22: por cada PDF imprime el
adaptador detectado, las líneas que mencionan `SALDO` con la coordenada X de cada importe y lo que devolvería
`parse_balances()` con el `x_max` de ese adaptador. Avisa en dos casos —cuando el PDF publica saldo y el adaptador
todavía no lo guarda, y cuando leer solo la hoja 1 y la última da otro resultado que leerlas todas, porque el
adaptador de tarjeta lee solo esas dos. Verificado contra un resumen Santander: `balance_prev 501.782,81` de la
hoja 1 y `balance_curr 533.299,51` de la línea con `TNA`/`TEM` en la hoja 7, con el de dólares (`0,00@x559`)
quedando afuera por el `x_max` de 500.

**2. `uq_import` no era el bloqueo que parecía.** Se documentó que un extracto ya importado no puede aportar el saldo
porque el alta no se repite. Es verdad del alta y falso del adaptador: `BbvaArTarjeta.load()` ya tenía una rama que
**sigue trabajando** cuando el PDF está importado —graba el ciclo, con el comentario "los resúmenes cargados antes de
existir `fin_card_cycles` se recuperan re-soltándolos"—. El saldo es el mismo caso: una columna que no existía cuando
se cargaron.

Las otras dos clases salían de esa rama con `cursor.close(); return stats`. Las tres completan hoy por
`_fill_missing_balances()`, que escribe con `COALESCE`: rellena lo que falta y **nunca sobreescribe lo cargado**, así
que volver a soltar un extracto es idempotente.

**Eso reemplaza el script de relleno por `file_hash` que se había ofrecido.** No hace falta: el camino es el de
siempre —soltar el PDF en `extractos/`— y sirve igual para los meses ya importados y para los que faltan. Un script
aparte habría sido un parche externo al import, con la lógica del saldo escrita dos veces y sin garantía de que el
próximo camino de entrada la respete.

**Estado de BBVA al cierre del día.** Los veinticinco imports de las tres secciones (`bbva_ahorro`, `bbva_cuenta`,
`bbva_tc`) están **todos** con `balance_prev` y `balance_curr` en NULL: ninguno se cargó con el código nuevo. Se
llenan re-soltando los PDF. Los meses que además no tienen import son la caja de ahorro **2026-07** —septiembre está
parcial, `period_to` 14-09—, la Visa **2026-06** y la Master **2026-09**.

Dos rarezas anotadas sin tocar: el import 78 (enero, cuenta `196-009369/5`) entró con sección `bbva_cuenta` en vez de
`bbva_ahorro`, y la Visa tiene **dos julios** — el import 183 con `period_to` 01-jul y el 336 con 28-jul.

#### `short_name` pasó a ser obligatorio, y el 2.8 estaba mal enunciado (2026-09-21)

Siete de las quince cuentas tenían `short_name` vacío y **`Visa` estaba repetido en dos** (Santander id 3 y BBVA Visa
Signature id 9): en el panel las dos se veían igual. El `UPDATE` de las quince es el parche; la garantía es del
schema, porque el script no es el único camino de entrada. `fin_accounts.short_name` es hoy `varchar(20) NOT NULL`
con `UNIQUE KEY uq_short_name` y `CHECK (short_name <> '')`, y **sin `DEFAULT`**: un `INSERT` que omita la columna
falla en vez de crear una cuenta muda. El `MODIFY` copió el tipo literal de `SHOW CREATE TABLE` y no uno inferido —
la lección de `inversion.deuda` de la misma mañana.

`AppTest/run_finanzas_setup.py` quedó alineado: declara la columna en su DDL con las tres restricciones, las diez
cuentas del dict la traen y el `INSERT` la escribe. A propósito **no** lleva `setdefault` como `credit_limit`: una
cuenta nueva sin `short_name` tiene que reventar con `KeyError`, no crearse muda.

**El 2.8 no es "el instalador tira error 1054".** El release del hijo no corre ese script: `export_hijo.bat` genera
`hijo_estructura.sql` con un `mysqldump --no-data` de las seis tablas `fin_*` **desde la BD viva**, así que ya se
lleva `short_name`, `tracked_since` y las dos columnas de saldo sin tocar nada, y `hijo_datos.sql` exporta
`fin_banks`, `fin_categories` y `fin_import_rules` — **`fin_accounts` no se exporta**. Un hijo instala Finanzas con
bancos, categorías y reglas y **sin ninguna cuenta**, y no hay forma de darlas de alta sin entrar a MySQL. Eso es
correcto por diseño —sus cuentas no son las del padre— y es el hueco real: falta el alta de cuentas. El error 1054 de
`fin_categories.type` sigue existiendo y afecta al setup local del padre, no al release.

Dos columnas de la BD viva no estaban en el DDL del script —`short_name` y `tracked_since`— más cuatro bancos y cinco
cuentas que solo existen en la BD (Citi ×2, BDV, Banesco, Binance). El script describe un esquema de Finanzas que ya
no existe; se le agregaron las dos columnas y no las cuentas, que son la parte con decisión de diseño pendiente.

------------------------------------------------------------------------

### Fase 2 — por qué reclasificar a mano no se pega

**Caso testigo: "panadería porteña"** — pedido del usuario el 2026-09-18: *"la reclasifico de manera manual en casi
todos los meses"*, y de ahí tres cosas que quiere: *"mirar los movimientos que caen en más de una categoría"*,
*"por qué no se replican todos los movimientos hacia atrás y adelante cuando reclasifico"* y *"que cuando se vaya a
recategorizar un movimiento se muestren las categorías que ya tiene"*.

No es un pedido de función nueva solamente: la replicación hacia atrás **ya existe** en el código y no funciona.
Leído y **medido contra la BD** el 2026-09-18, son **tres causas distintas** y cada una rompe un tramo diferente.
La que produce el caso testigo es la B; la A es un bug aparte que se encontró buscándola.

**Lo que se midió primero** (1.539 filas de `fin_transactions`): `rule` 1.258, **`manual` 214** — el trabajo a mano,
inmune a toda regla —, `classified_by` NULL **con** categoría 37, NULL **sin** categoría solo **21** (205 USD),
`synthetic` 9.

*Corregido el 2026-09-18, mismo día:* de esas 21 filas se concluyó acá que "las filas nuevas llegan con una categoría
puesta por una regla y el usuario la corrige". **Es falso**, y lo desmintió el propio caso testigo (ver Causa B): las
filas del comercio llegan **sin** categoría y el trabajo a mano es ponerla, no corregirla. Las 21 no son un backlog
residual: son la punta visible de lo que el retro roto dejó sin tocar. Se deja anotado porque el número —21 sobre
1.539— invita a la conclusión equivocada.

**Causa A — CORREGIDA el 2026-09-18 — hacia atrás: el UPDATE descartaba justo las filas que venía a arreglar.**
Pesa menos de lo que parecía (21 filas, 205 USD), pero el arreglo destapó algo peor: al corregir solo el `NULL`,
el retro quedaba habilitado a pisar las **37 filas con categoría y `classified_by` NULL** — categorías puestas por
SQL durante el saneamiento, decisiones tan deliberadas como las manuales. De esas 37, **26 coincidían con alguna
regla activa**: **4.361 USD** expuestos (6 filas en *Pago tarjeta crédito* = 1.038 USD, 2 en *Préstamo recibido* =
1.894 USD, 6 en *Inversiones* = 785 USD, y el resto repartido) contra 205 USD de backlog legítimo. *El valor 4.156
que figuró acá hasta el 2026-09-18 estaba mal: era la suma menos los 205 del backlog, restados por error.* Así que
el predicado correcto no es "no pisar lo manual" sino
**"no pisar nada que ya tenga categoría"**, y vive una sola vez en `_SQL_RECLASIFICABLE`, usado por el SELECT de
candidatos, por el UPDATE y por el preview. Que fueran dos predicados distintos **era** el bug.

Diagnóstico original: `save_rule()`
(`Modulos_Mysql.py`) elige candidatos con `WHERE classified_by IS NULL OR classified_by = 'rule'` y después aplica
con `WHERE id IN (...) AND classified_by != 'manual'`. En SQL `NULL != 'manual'` **no es verdadero, es NULL**, así
que las filas sin clasificar —la mayoría del histórico, y las que el `IS NULL` acababa de seleccionar— nunca se
actualizan. Solo se mueven las que ya tenían `classified_by = 'rule'`. El segundo filtro es además redundante: el
`SELECT` ya había excluido las manuales. Efecto visible: el preview del popup (`save_rule_count_retro()`, que usa
el mismo `SELECT` y **no** el segundo filtro) promete *"coinciden N transacciones"* y se aplican muchas menos, o
ninguna. Es un bug de una línea, no un rediseño.

**Causa B — CONFIRMADA CON FECHAS — hacia adelante: el patrón por defecto es la descripción completa.** El comercio
no se llama "panadería" en la BD (`LIKE '%PANADER%'` no devuelve nada): es **"La portena"**. Sus reglas, las tres
creadas por el usuario y las tres apuntando a *Restaurantes* con prioridad 50:

| id | patrón | tipo | creada | aciertos | último acierto |
|---|---|---|---|---|---|
| 317 | `Compra con tarjeta de debito — La portena - tarj nro. 2861` | contains | **2026-04-11** | **0** | nunca |
| 490 | `La portena` | contains | **2026-09-13** | 14 | 2026-09-16 |
| 530 | `Compra con tarjeta de debito — La portena - tarj nro. 2861` | exact | **2026-09-17** | **0** | nunca |

*Reversión anotada:* la primera versión de esta sección describía el bucle como "la 490 clasifica el movimiento a
*Restaurantes*, el usuario lo corrige a otra categoría y crea reglas inútiles". **No es eso lo que pasa.** Las 6 filas
del comercio están todas en *Restaurantes* o sin categoría — ninguna en otra categoría, así que no hay criterio
corregido. El usuario no corrige una categoría equivocada: **pone a mano la que la regla debería haber puesto**.

El bucle real, con las fechas de las tres reglas y las 6 filas del comercio:

| fecha | hecho |
|---|---|
| 2026-04-11 | se crea la 317 con la línea entera del resumen, número de tarjeta adentro → **0 aciertos en 5 meses** |
| may, jul, ago | 4 filas entran sin categoría y se clasifican a mano → *Restaurantes* |
| jun | 2 filas entran sin categoría y **quedan así** (ids 2327 y 2330) |
| 2026-09-13 | se crea la 490 con el patrón corto `La portena` → coincide, pero junio sigue sin categoría (**Causa A**) |
| 2026-09-17 01:57 | se crea la 530: otra vez la línea entera, ahora `exact` → **0 aciertos** |

Eso es el *"la reclasifico en casi todos los meses"* medido: **cinco meses con una regla activa que nunca coincidió con
nada.** Y la 530, creada de madrugada cuatro días después de la 490, es el intento de arreglar junio con las dos causas
pegando juntas — el popup volvió a proponer la línea completa, y aun con el patrón corto el retro no habría movido esas
filas.

**`hit_count` cuenta coincidencias, no clasificaciones** (medido el 2026-09-18, cambia el criterio del paso 2.3). La 490
tiene 14 aciertos en 3 días sobre un comercio con **6 filas** en la BD: son reimportaciones del mismo resumen —
`apply_rules()` suma el contador **antes** del `INSERT IGNORE`, así que la regla acertó 14 veces sin escribir una sola
categoría (las 6 filas siguen en `manual` o NULL). Al revés también: el retro de `save_rule()` evalúa el patrón en
Python y **no** pasa por `apply_rules()`, así que cambia filas sin sumar nada. Consecuencia: **no se pueden borrar reglas
solo por `hit_count = 0`** — una regla con 0 aciertos puede haber clasificado por retro. Para la 317 y la 530 da igual
(el patrón con número de tarjeta no sirve en ningún caso), pero el paso 2.3 necesita otro criterio.

De ahí que el pedido *"que se muestren las categorías que ya tiene"* sea más exacto de lo que parecía: lo que falta
no es solo ver la categoría, es **ver si hay una regla y cuál gana** — acá habría mostrado que ya existían dos reglas
muertas y una viva, y que no hacía falta crear una tercera.

`_CategoryEditPopup` arranca con
`self._var_pattern = StringVar(value=raw_desc.strip())` y `match_type = "contains"`. Si la descripción del comercio
trae comprobante, sucursal o fecha —y las de tarjeta los traen—, ese patrón entero no vuelve a coincidir nunca con
el resumen del mes siguiente: `apply_rules()` no lo encuentra, la fila entra sin categoría y hay que corregirla a
mano otra vez. Es el mismo fenómeno que partió el plan `Look shopping adrogu` / `adrogue` en dos (§ Fase 3): **la
descripción no es clave estable**. Por eso el paso 2.1 pide "clave corta": el patrón que se guarda no debe ser la
línea del resumen, sino la parte del nombre del comercio que se repite.

**Causa C — una fila tocada a mano queda inmune para siempre.** `update_txn_category()` estampa
`classified_by='manual'` sobre **una sola fila**, y desde ese momento ninguna regla la vuelve a evaluar. Esto es
deliberado y debe quedar así —la decisión del usuario no se pisa con una automatización—, pero tiene un costo que
nadie había anotado: si más adelante cambia de criterio, las filas viejas que corrigió a mano conservan la
categoría vieja — el mismo comercio queda repartido en dos o tres categorías y ninguna pantalla lo muestra hoy.
La salida no es dejar que las reglas pisen lo manual, es poder **ver** el reparto y reasignar el grupo entero a
propósito (pasos 2.2 y 2.5).

*Atribución corregida el 2026-09-18:* acá se dijo que esto era "el origen del primer pedido". **No está medido.** El
caso testigo no está repartido —las 6 filas de "La portena" son todas *Restaurantes* o nada—, así que el pedido de ver
los movimientos que caen en más de una categoría sigue siendo válido pero **todavía no tiene un caso que lo
justifique**. Lo mide el paso 2.5 cuando se implemente: comercios con filas en más de una categoría, contadas.

**Causa D — CORREGIDA el 2026-09-19 — el popup dejaba cambiar la categoría sin que el usuario lo viera.** El
`Combobox` de categoría se creaba con `state="normal"` —texto libre— y con `focus_set()` puesto encima al abrir la
ventana, así que una flecha del teclado o la rueda del mouse seleccionaba otra categoría sin ninguna acción
deliberada. Peor cuando la fila **no tenía** categoría: el `set()` inicial solo corre si la categoría actual está en
la lista, así que el combo quedaba **vacío** —exactamente el caso en que se usa el popup— y el primer scroll lo
llenaba con cualquier valor. Pasó de verdad: al ir a arreglar las dos filas de junio de "La portena", una quedó
guardada en *Otros gastos*. Hoy el combo es `readonly` y el foco arranca en el campo **Patrón**, que es el que se
edita siempre. El patrón sigue siendo texto libre a propósito: *"no importa el medio que use para pagar en La
porteña, siempre debe ir a Restaurantes"* — si el patrón no se pudiera acortar a mano, haría falta una regla por
medio de pago y por número de tarjeta.

**Lo que hoy no existe.** El popup muestra la categoría *de esa fila* (`txn_row.get("category")`) y nada más: no
muestra qué otras categorías tuvo ese comercio, ni cuántas filas hay en cada una, ni si existe ya una regla
apuntándole. Sin eso el usuario decide a ciegas y cada corrección es una fila, no un criterio.

**Qué cierra esto, en orden.** (1) La pasada única sobre las 21 filas sin clasificar (paso 2.0) — el fix del retro
evita que se vuelva a ensuciar pero no limpia lo acumulado, y ahí están las dos filas de junio del caso testigo;
(2) el paso 2.1 con clave corta — sin eso cada corrección sigue dejando basura; (3) el popup mostrando las reglas que
ya matchean el movimiento, con la opción de **editar o desactivar** la que gana (paso 2.6); (4) limpieza de reglas
muertas (paso 2.3), con un criterio que no sea `hit_count = 0` — las 317 y 530 se van por tener el número de tarjeta
en el patrón, no por el contador. Lo que **no** hay que hacer es dejar que las reglas pisen lo manual: las 214 filas a
mano son decisiones, y el problema es que haya que tomarlas 214 veces, no que se respeten.

**Caso testigo cerrado y fix validado en vivo (2026-09-19).** El usuario reinició la app, desactivó las dos reglas de
patrón largo (`is_active = 0`, no borradas: la traza de por qué existieron se conserva) y volvió a guardar la regla con
la clave corta. Estado final del comercio: **las 6 filas en *Restaurantes***, y las dos de junio —2327 y 2330, tres
meses sin categoría— ahora con `classified_by = 'rule'`. Es el retro corregido moviendo filas que antes seleccionaba y
después descartaba. Queda **una sola regla activa**: `La portena` / contains / *Restaurantes*, que no depende del medio
de pago ni del número de tarjeta.

Ese mismo cierre **confirmó lo de `hit_count`**: la 490 clasificó las dos filas de junio y quedó igual, en 14 aciertos
con `last_hit_at` el 2026-09-16. El retro escribe sin tocar el contador — ya no es una medición indirecta.

*Reversión anotada el 2026-09-19, mismo día:* **el caso testigo no estaba cerrado y "las 6 filas" era un número
falso — son 37.** El `SELECT` con el que se dio por cerrado buscaba el comercio en `COALESCE(raw_description,
description)`, y en Santander el comercio no vive ahí: vive en `raw_description_detail`. La consulta de diagnóstico
tenía el mismo defecto que el código que estaba diagnosticando, así que confirmó un cierre que no existía. Lo que sí
queda en pie de ese párrafo: las dos reglas de patrón largo desactivadas, las filas de junio movidas por el retro y
la confirmación de `hit_count`. Lo que se cae es el alcance. La causa real es la E.

**Causa E — LA CAUSA RAÍZ, corregida el 2026-09-19 — el import y el retro leían textos distintos.** `apply_rules()`
evalúa el patrón contra `(description + " " + detail).strip()`; el retro de `save_rule()` y el preview lo evaluaban
contra `COALESCE(raw_description, description)`, **sin el detalle**. En las cuentas que parten el movimiento en dos
columnas eso significa que el patrón del comercio no coincide nunca hacia atrás: la regla clasifica bien la fila que
entra y no toca ninguna de las viejas. Es la explicación completa de *"lo reclasifico a mano todos los meses"*, y las
causas A a D eran agravantes.

Medido el 2026-09-19 sobre las 1.539 filas: **682 (44%) tienen detalle.** No está repartido parejo, y ahí está el
daño:

| cuenta | filas | con detalle | encabezados distintos | qué hay en el detalle |
|---|---|---|---|---|
| Santander CA/CC ARS | 460 | **374** | 31 | **el comercio** |
| BBVA CA ARS | 286 | 224 | 101 | códigos numéricos (`100`, `104`, `246`) |
| Tarjetas (BBVA, Santander, Amex, Visa) | 488 | 0 | 48–79 | — |
| Banesco / BDV / Citibank | 196 | 0 | 3–85 | — |

Santander CA/CC es la cuenta del día a día del usuario y es justo la peor: 374 filas donde el encabezado es el
instrumento (`Compra con tarjeta de debito`, 108 filas, ratio de 14,8 filas por encabezado) y el negocio está del
lado del detalle. BBVA es sueldo y tesorería, su detalle **no** trae contraparte —se verificó, son códigos— y sus
101 encabezados son ~15 instrumentos contaminados con identificadores (`OPER.FONDO COMUN DE INVERSI <nro>CUT`,
`DEBITO CUOTA 196-96-00091422`), que se resuelven con `startswith` y no necesitan código nuevo.

La corrección es una constante, `_SQL_TEXTO_REGLA`, con el mismo `CONCAT` que arma el import, usada por el SELECT de
candidatos y por el preview; y `_match_pattern()`, un estático único que evalúan los dos. Que el preview y el UPDATE
tuvieran cada uno su copia del criterio es lo que hacía que la pantalla prometiera filas que después no se movían.

**Propagación con alerta previa (2026-09-19).** Pedido textual: *"asegurar que si catalogo un gasto eso pueda
replicarse en toda la Base de datos. me das la alerta de todo lo que cambiaria asi no voy a siegas"*, y después
*"por que no lo pones en una lista -- asi lo puedo revisar antes de confirmar"*. `save_rule_count_retro()` devolvía
un número y el label decía *"clasificará N transacciones sin categoría"* — mentía dos veces: contaba también las que
ya tenían categoría puesta por otra regla, y no decía cuáles. En el caso testigo ocultaba 18 filas. Lo reemplaza
`save_rule_preview()`, que devuelve el desglose por categoría más el detalle fila por fila, y el popup lo muestra
como resumen + **lista completa con scroll** (fecha, monto, movimiento, categoría actual). La fecha y el monto van
porque el usuario reconoce la compra por ahí, no por la línea del resumen: *"claro eso los busco trato de acordarme
que compre"*. Dorado = la regla la mueve; gris = queda como está. Por encima de 10 filas el botón pide un segundo
clic con el número a la vista.

`incluir_manual` es la casilla que permite pisar también lo clasificado a mano, apagada por default y visible solo
cuando hay algo que pisar. Existe porque la causa D dejó filas en categorías que nadie eligió. **Nunca toca
`synthetic`.**

**Medición del caso testigo, ahora con el texto completo:** `La portena` / contains coincide con **37 filas** — 18
puestas por regla en *Otros gastos*, 16 manuales en *Restaurantes*, 2 manuales en *Supermercado*, 1 manual en *Otros
gastos*. Eso es lo que el diagnóstico anterior no veía.

**Defecto abierto, encontrado en el primer uso real.** El usuario guardó con la categoría que traía el combo
—*Otros gastos*, la que había puesto la regla vieja— en lugar de *Restaurantes*: las 37 fueron a *Otros gastos* y las
19 manuales pasaron a `classified_by = 'rule'`. La segunda pasada las corrigió a *Restaurantes*, pero **la marca de
"esto lo decidí yo" no se puede recuperar**: las 2 filas de *Supermercado* quedaron indistinguibles del resto.
Tildar `incluir_manual` es hoy una puerta de un solo sentido. Falta guardar la categoría y el `classified_by`
previos antes del retro para poder deshacer. Mitigación puesta en el momento: el encabezado del preview dice **a qué
categoría** se mueve (`MUEVE 37 fila(s) a Restaurantes`) y se actualiza al cambiar el combo.

**2.1 — el popup propone el comercio, no la línea del resumen** (2026-09-19, cierra el paso).

El default era `raw_desc.strip()`: la línea entera, con número de tarjeta, comprobante, fecha, cuota o TNA adentro.
La regla nacía muerta —ese texto no vuelve a aparecer igual el mes siguiente— y el mismo comercio había que
reclasificarlo mes a mes. Lo propone ahora `_CategoryEditPopup._clave_corta()`, y el campo **sigue editable**: es la
condición del paso 2.1, sin ella haría falta una regla por medio de pago.

Cómo llega al comercio, en orden, y por qué cada paso:

| Paso | Ejemplo | Motivo |
|---|---|---|
| Si hay separador `" — "`, se queda con el **detalle** | `Compra con tarjeta de debito — La portena 20 de julio` → `La portena` | el encabezado es el medio de pago; tomarlo obliga a una regla por instrumento (causa E) |
| Corta en el primer `(` | `CUOTIFICACION 6-06 (TNA 79,00)` | lo que va entre paréntesis es el dato del mes, nunca el comercio |
| Saca el prefijo de billetera | `MERPAGO*MIPM` y `Merpago/mipm 1234` → `MIPM` / `mipm` | el mismo comercio llega en dos formatos; sin el prefijo una sola regla cubre los dos |
| Parte el texto en tramos separados por tokens variables y elige el tramo con **más letras** | `COMPRA 4517*1234 FARMACITY 15/08` → `FARMACITY` | **no el primero**: el comercio no siempre viene al principio, y quedarse con `COMPRA` crea justo el comodín de la regla 269 |
| Tope de 3 tokens, y el tramo queda **contiguo** | | el patrón se evalúa como subcadena: unir tokens salteados daría una clave que no existe en ningún movimiento |
| Si la clave queda con menos de 4 caracteres, cae a los primeros 30 del texto | `ASTERISCO 22` | mejor un patrón largo que uno que arrastre filas ajenas |

Medido sobre 13 líneas reales de los extractos (Santander, BBVA, Visa, Amex): **13 de 13** dieron una clave usable.

*Cambio de comportamiento a la vista:* `VISA PLAN V C.01/06 LOOK SHOPPING` propone `LOOK SHOPPING` —el comercio— y no
`VISA PLAN V`. Es lo correcto: `VISA PLAN V` es el instrumento y ya lo cubre la regla genérica del plan de tarjeta
(paso 2.3).

**La regla comodín 269, que es el motivo de la elección "el tramo con más letras".** Patrón `Compra con tarjeta de
debito` → *Otros gastos*, prioridad 100: el instrumento, no el comercio. Clasifica correctamente 0 movimientos y
manda a *Otros gastos* todo lo que se paga con débito en Santander —la cuenta del día a día—, con la prioridad ganando
a reglas más específicas. Es el patrón exacto que `_clave_corta()` habría propuesto con la versión anterior del
algoritmo, que se quedaba con el primer tramo.

**Ejecutado el 2026-09-19** (`UPDATE fin_import_rules SET priority = 200 WHERE id = 269;`). Bajar la prioridad y no
desactivarla **no es lo mismo**, y conviene tenerlo claro porque son dos efectos distintos:

| | Con `priority = 200` (lo que se hizo) | Con `is_active = 0` |
|---|---|---|
| Compra con débito **que tiene** regla de comercio | gana la regla buena — `ORDER BY priority, id`, menor número primero | gana la regla buena |
| Compra con débito **sin** regla de comercio | sigue cayendo en *Otros gastos*, invisible | queda `Sin categoría`, visible en la pasada del 2.0 y en la fila roja del Flujo |
| Efecto en el número de libertad financiera | se mantiene: *Otros gastos* tiene `expense_class` y suma al costo de vida | **baja** hasta que esas filas se clasifiquen |

Se eligió bajar la prioridad: deja de pisar las reglas específicas —que era el daño— sin tirar el costo de vida hacia
abajo por filas que todavía nadie clasificó. El paso **2.0** es el que las va a sacar de *Otros gastos*, y recién ahí
tiene sentido desactivarla. **Lo que sella el caso no es este UPDATE sino `_clave_corta()`** (paso 2.1, ya en código):
el popup no vuelve a proponer el instrumento como patrón, así que no puede nacer otra regla comodín por esta vía.

Sigue pendiente de correr la desactivación de las reglas con la línea completa del resumen como patrón (paso 2.3).

**La 269 no era la única — hallazgo del 2026-09-19 al verificar el UPDATE.** Listando las reglas activas por
precedencia aparecieron dos más con el mismo defecto de fondo, el instrumento en lugar del comercio:

| Regla | Patrón | Destino | Aciertos |
|---|---|---|---|
| 48 | `startswith MERPAGO` | *Otros gastos* | **140** |
| 49 | `startswith MERCADOPAGO` | *Otros gastos* | 0 |

`MERPAGO` es el prefijo de la billetera: es exactamente lo que `_clave_corta()` **saca** del texto para llegar al
comercio. Con la 269 en 200, la 48 pasó a ser el comodín más dañino que queda activo — 140 coincidencias hacia el
cajón. Mismo tratamiento, **ejecutado el 2026-09-19**:
`UPDATE fin_import_rules SET priority = 200 WHERE id IN (48, 49);`. Con eso los tres comodines de instrumento quedan
detrás de cualquier regla de comercio: 535 coincidencias que antes ganaban por prioridad y ahora pierden.

**Esto cierra el criterio del paso 2.3, que estaba planteado como "limpiar reglas muertas".** De las 74 reglas activas,
**22 tienen 0 aciertos** (CARREFOUR, JUMBO, WALMART, UBER, DIDI, DISNEY, HBO, SUBWAY, MOSTAZA…) y **ninguna es basura**:
el patrón es el nombre del comercio, está bien escrito, y el contador en cero solo dice que ese comercio no apareció en
los extractos cargados. Borrarlas no arregla nada y rompe la clasificación del día que aparezcan. Lo que el 2.3 tiene
que limpiar no se detecta por el contador sino por **la forma del patrón**: el instrumento en lugar del comercio
(269, 48, 49) y la línea completa del resumen con número de tarjeta y fecha adentro. Es la misma conclusión que ya
estaba anotada para `hit_count`, ahora medida sobre el catálogo completo.

**La sospecha sobre `startswith DIA` se midió y quedó descartada — pero el camino dejó una regla.** La hipótesis era que
un prefijo de tres letras arrastraba filas ajenas, y de paso que las 13 reglas `startswith` eran ciegas a los dos
formatos que más usa el usuario: pago por billetera (`MERPAGO*comercio`) y débito Santander
(`Compra con tarjeta de debito — comercio`), donde el comercio nunca está al principio.

Medido el 2026-09-19 contando, para cada regla `startswith` activa, las filas que **contienen** el patrón y **no
empiezan** con él:

| Regla | Filas que no ve | USD | Qué resultó ser |
|---|---|---|---|
| 1 `DIA` | 82 | 10.306 | **falsos positivos de subcadena** |
| 48 `MERPAGO` | 37 | 239 | es el comodín: que no las vea es lo deseado |
| 576, 3, 248, 13, 4 | 8 en total | 447 | ruido |
| las otras 8 | 0 | — | el comercio no apareció, el patrón está bien |

Los 10.306 USD de la regla 1 son `TRANSFERENCIA INMEDIATA` — inme**dia**ta —: 42 filas, ~10.020 USD, más
`QUOTIDIANO`, `LA MUNDIAL` y `Comision custodia`. Las filas reales del supermercado (`MODO*DIA TIENDA 150`,
`MERPAGO*SUPERDIA`, `Merpago/dia`, `Compra con tarjeta de debito Dia tienda 150`) **ya estaban todas en
*Supermercado***, y los reintegros de la promo en *Reintegros y bonificaciones*, donde corresponde. No había nada
perdido y no se tocó nada.

**La regla que queda: `startswith` no es un defecto a corregir en masa, es una protección cuando el patrón es corto.**
Pasar la regla 1 a `contains DIA` habría mandado 42 transferencias —10.020 USD— a *Supermercado*. Un patrón de tres
letras solo es seguro anclado al principio; si hace falta que coincida en el medio, el `match_type` correcto es `regex`
con límite de palabra (`\bDIA\b`), nunca `contains`. Para patrones largos y distintivos (CARREFOUR, JUMBO) la discusión
no existe: la medición dio cero.

**Con esto el paso 2.3 queda cerrado de criterio**, aunque falte correr la desactivación de las reglas con la línea
completa del resumen como patrón. Lo que se aprendió en las tres mediciones: no se borra por `hit_count`, no se
reescribe el `match_type` por hipótesis, y los únicos patrones que hay que bajar son los del instrumento en lugar del
comercio — los tres ya bajados.

------------------------------------------------------------------------

### Fase 3 — registro de lo hecho

**3.2 — comprometido hacia adelante: las cuotas** (2026-09-18).

Pregunta que lo disparó, y que el panel no podía contestar: *"¿cuánto tengo que pagar completo en octubre?"*. El mes
en curso se veía entero, pero lo que ya está comprometido hacia adelante no se veía en ninguna parte: una cuota
existe en la BD solo desde el resumen que la cobra, así que un plan 3/12 tiene nueve cuotas que todavía no son
ninguna fila.

*Cambio de enfoque sobre el plan original:* el paso pedía una tabla `fin_card_installments_due`. **No se creó, y no
debe crearse.** Todo lo que esa tabla guardaría ya está en `fin_transactions`: la última cuota cargada de cada plan
trae el monto, la moneda, el cierre que la cobró y el total de cuotas. Una tabla aparte sería una segunda copia de
un dato derivable, con su propio camino para quedar desactualizada — y el problema de este módulo nunca fue la falta
de tablas sino la confianza en las que hay. La proyección se calcula al leer, en
`FinanceScreen.get_installments_pending()`.

**Cómo identifica un plan.** Por `(cuenta, comprobante, total de cuotas)`, no por la descripción. El texto no sirve
de clave: el mismo plan llega escrito distinto en cada resumen —`Look shopping adrogu` y `Look shopping adrogue`
partieron un plan de 6 cuotas en dos grupos de 5 y 1, y durante días pareció una cuota faltante—. El comprobante es
el mismo en todas las cuotas del plan.

**Qué proyecta.** Las cuotas posteriores a la última cargada, una por mes desde su cobro. No hay doble conteo con
los KPIs: lo ya facturado queda del lado de la última cuota conocida y la proyección arranca recién después.

**El monto futuro es el de la última cuota conocida, y eso sobreestima a propósito.** En un plan en pesos la cuota
es fija en ARS, así que su equivalente en dólares baja con el tipo de cambio: proyectar con el valor de hoy deja el
número del lado caro. Para una pregunta de "¿me alcanza?" equivocarse hacia arriba es lo correcto.

**Una cuota proyectada con fecha anterior al mes en curso no es un error de cálculo** — es un resumen que no llegó,
el mismo hueco que marca `get_coverage()`. Se muestra en su período real en vez de esconderse.

**Dónde se ve.** Tercera vista del panel de detalle (`Comprometido`), junto a `Últimas transacciones` y `Cobertura`,
en bloques por mes y con el total del mes arriba de cada bloque. Las tres vistas pasaron a un solo estado
`self._detail` (`txn` / `cover` / `commit`): con dos banderas booleanas, tres vistas admiten una combinación
inválida. No depende del período elegido —mira hacia adelante—, sí de los chips de cuenta.

**Lo que este paso todavía no cubre:** la cuota del préstamo y los fijos promedio (alquiler, expensas, familia,
servicios, suscripciones). Son de naturaleza distinta: la cuota de tarjeta es un compromiso con fecha y monto
exactos, el fijo es un promedio. Mezclarlos en un solo número borraría esa diferencia, y el número que hay que
tener reservado para que el banco no cuotifique al 69–102% es el de las cuotas.

------------------------------------------------------------------------

### Fase 4 — registro de lo hecho

**4.4 — la vista Flujo: el promedio por categoría y el número de libertad financiera** (2026-09-19).

Pedido textual: *"estoy más interesado en ver cómo se ve el flujo de dinero en las categorías y cómo puedo calcular mi
número para la libertad financiera, ver los promedios de ingresos, gastos e inversiones"*. Y la forma, que es una
decisión de arquitectura, no de UI: *"lo quiero como opciones dentro de la app — es un mecanismo para las
depuraciones"*. Esto fija una regla para lo que viene: **una consulta de diagnóstico que vale la pena mirar dos veces
termina siendo una vista, no un SQL pegado en el editor.** Lo que no está en la app hay que volver a escribirlo cada
vez y nadie lo revisa.

**Cuatro ventanas: 1M, 3M, 6M, 1Y** (*"ok flujo 1m 3m 6m y 1y"*). Una sola no distingue el gasto que subió del gasto
que pasó una vez; el mes contra el año es lo que muestra la deriva. Son suyas: **no dependen del selector Mes/Año**,
sí de los chips de banco y cuenta.

**El divisor de cada ventana es la cantidad de meses con movimiento, no los del calendario.** Con `tracked_since` en
enero, dividir el año por 12 inventa meses baratos que no existieron — es el borde de atrás del paso 1.4 metido en un
promedio. El encabezado de la columna lo declara (`1Y ÷8`), así que el usuario ve sobre cuántos meses se promedió sin
tener que deducirlo. Medido el 2026-09-19: 1M÷1, 3M÷3, 6M÷6, **1Y÷8** — enero a agosto, exactamente el piso
declarado.

**El mes en curso no entra en ninguna ventana.** Está abierto: sus resúmenes todavía no llegaron y bajaría los cuatro
promedios a la vez.

**El signo.** En todo lo que no es `income`, positivo = sale plata, negativo = entra. Una fila `Inversiones` en −508
es rescate neto del mes, no un error. Es la misma convención de `_SQL_NETO` que usan los KPIs.

**El número** = (`fixed` + `variable`) × 12 ÷ 4%, o sea 300 veces el costo de vida mensual. `extraordinary` queda
afuera del costo de vida y se promedia aparte: mudanza, mantenimiento, tecnología y viajes no son gasto de todos los
meses, y meterlos adentro infla el objetivo con plata que no hay que rendir para siempre.

Estado al 2026-09-19 (promedio mensual USD, cuentas todas):

| | 1M | 3M | 6M | 1Y |
|---|---|---|---|---|
| Ingreso | 2.824 | 3.338 | 3.047 | 3.142 |
| Gasto | 1.940 | 2.620 | 2.622 | 2.430 |
| Ahorro | 883 | 718 | 425 | 712 |
| Tasa de ahorro | 31,3% | 21,5% | 13,9% | **22,7%** |
| Costo de vida (`fixed`+`variable`) | 1.940 | 2.540 | 2.454 | 2.260 |
| Extraordinario | 0 | 81 | 167 | 169 |
| **Número** | 582.127 | 761.941 | 736.162 | **678.111** |

La ventana que hay que leer es 1Y: 6M queda alta por la mudanza (122/mes) y 1M es un solo mes. Las tres categorías
que mandan el número son `Familia` (639), `Alquiler` (430) y `Cuotas de préstamo` (314) — 1.383 de los 2.260 del costo
de vida, 61%.

**Sirve de mecanismo de depuración sin agregar nada.** `Sin categoría` y las categorías sin `expense_class` aparecen
como filas propias, y la fila **Gasto sin clase** se pinta en rojo cuando no es cero: mientras haya, el costo de vida
y el número salen bajos. Hoy es 1 USD/mes (300 sobre 678.111, irrelevante, pero visible).

**El progreso — cerrado el mismo día, con el hueco a la vista** (*"puede construir el punto 4.4 así vamos terminando"*).
Unas horas antes este mismo registro decía que el progreso quedaba abierto porque el saldo del préstamo no está en
ninguna tabla `fin_*`. **No se resolvió el dato: se resolvió la forma de mostrarlo.** Un avance que se calla lo que le
falta miente; un avance que lo declara sirve, porque el error tiene signo conocido — el real es **menor**. La última
fila del bloque, en rojo, dice `TECHO — falta restar el saldo del préstamo (paso 1.6)`.

Se lee de arriba hacia abajo como una resta, medido el 2026-09-19 con la ventana 1Y:

| Fila | USD | De dónde sale |
|---|---|---|
| Patrimonio a mercado | 57.896 | `SUM(mrkprice * position)` de `inversion`, `iactiva='Y'` |
| (−) Deuda de margen | 411 | `SUM(deuda)` — comprado con plata prestada no es patrimonio |
| (−) Cuotas de tarjeta pendientes | 1.349 | `get_installments_pending(120)` sobre `usdt` — consumo ya hecho esperando su resumen |
| Patrimonio neto hoy | **56.136** | |
| Número a cubrir (1Y) | 678.111 | costo de vida × 300 |
| Falta | 621.975 | |
| Avance | **8,3%** | barra `████░░░░` de 26 bloques |
| Años al ritmo de ahorro | **26,8** | ahorro 712/mes, 5% real |

**El horizonte de las cuotas es 120 meses, no 6.** La vista `Comprometido` proyecta 6 porque muestra el calendario de
lo que viene; acá "pendiente" tiene que significar *todo* lo pendiente o el neto sale alto. El plan más largo que emite
un banco local es de 24 cuotas, así que 120 meses es holgura, no un límite.

**Los años salen de la anualidad con capital inicial, no de dividir el faltante por el ahorro.** El capital ya
invertido también rinde: `FV = P(1+r)^n + PMT((1+r)^n − 1)/r`, resuelta en `n = ln((FV·r + PMT)/(P·r + PMT)) / ln(1+r)`.
Dividir 621.975 por 712 daría 72 años — ignora que los 56.136 trabajan. El rendimiento real es
`_FREEDOM_RENDIMIENTO_REAL = 0,05`, **un supuesto mío, no una medición de la cartera**, y por eso la fila lo escribe al
lado del número (`Años al ritmo de ahorro (5% real)`).

**La ventana que alimenta el progreso es 1Y, no 1M.** El número de 1M (582.127) y el de 6M (736.162) se separan en
150.000 USD: con la ventana corta, el porcentaje de avance y los años saltarían con cada gasto raro del mes.

*Corregido el mismo día:* la barra de estado seguía mostrando el número de la ventana corta —`totales["numero"][0]`—
y le pegaba al lado el avance y los años, que se calculan con 1Y. Quedaba una línea con **582.127 y 26,8 años juntos**,
dos ventanas distintas leídas como si fueran una sola cuenta. Ahora el status toma `[-1]`, la misma ventana que el
progreso. De paso el texto se acortó: con la tasa de ahorro y `(último mes cerrado)` adentro no entraba en la fila de
las pestañas y **se solapaba con ellas** cuando el panel es angosto. El detalle completo ya está en la tabla, la barra
de estado es el resumen de un vistazo.

**Lo que el número dice y el patrimonio no.** 8,3% cubierto con 26,8 años por delante es un resultado del **ahorro**,
no del patrimonio: 712/mes contra un objetivo de 678.111. Cada 100 USD/mes de gasto menos baja el número 30.000 **y**
sube el ahorro — pega por los dos lados, que es exactamente el objetivo 2 de §0.

**Reparto de responsabilidades.** `get_totales_inversiones()` vive en `PlanInversion` y `FinanceScreen` no hereda de
ella: son dominios distintos y ninguna puede resolver el progreso sola. Coordina el panel
(`FinancePanel._freedom_progress()`), que instancia las dos y le pasa patrimonio y deuda a
`FinanceScreen.get_freedom_progress()`. La alternativa —consultar `inversion` desde `FinanceScreen`— metería la tabla
de cartera en el módulo de finanzas personales. `get_totales_inversiones()` se refactorizó agregando `SUM(deuda)` al
SELECT que ya existía, en lugar de duplicar la query.

**Bug latente encontrado de paso:** `Modulos_Mysql.py` usaba `math` en la línea 5105 sin tenerlo importado —
`NameError` esperando a que alguien pasara por ese camino. Agregado al import de `Modulos_python`.

#### La base del avance pasa a ser el capital invertido (2026-09-20)

Pregunta del usuario: *"Mi plan de inversion esta en base al capital invertido, es posible calcular el numero de
libertad financiera del mismo modo?"*, y después de ver los números: *"ahora estoy en la etapa de acumulacion, ni
pretendo sacar dinero de mis inversiones, por eso siempre hablo de capital invertido"*. Decisión: **un solo valor de
avance, medido sobre lo aportado, con el k a la vista**.

**Esto revierte el criterio con el que se cerró el 4.4 el día anterior** (la tabla de arriba, que mide a mercado). No se
borra: el avance a mercado sigue siendo el número honesto para la pregunta *"cuánta renta paga la cartera"*, y el nuevo
es el número honesto para *"cuánto capital logré apartar"*. El usuario gestiona la segunda porque es la que controla, y
es la misma base que su plan de inversión (`trazaplan.tinversion`).

**Antes de cambiarlo se midió la brecha**, porque la elección tiene un costo y no se puede declarar sin el número
(`inversion`, `iactiva='Y'`, 2026-09-20):

| Vehículo | Capital invertido | Valor de mercado | Brecha | k |
|---|---|---|---|---|
| Stock | 63.892 | 45.584 | −18.309 | 0,713 |
| Crypto | 8.982 | 5.567 | −3.414 | **0,620** |
| BBVA.ARS | 6.839 | 6.727 | −112 | 0,984 |
| **Total** | **79.713** | **57.847** | **−21.866** | **0,727** |

El dato está sano: `SUM(unrealizedpnl)` cuadra con `SUM(mrkprice*position) − SUM(costobase)` al centavo en los tres
vehículos, y ninguna posición tiene `mrkprice` en 0 (que habría fabricado un k bajo sin pérdida real).

**El costo de la elección, con el número del 2026-09-19 (678.111):**

| | A mercado (2026-09-19) | Sobre lo aportado (2026-09-20) |
|---|---|---|
| Base | 57.847 | 79.713 |
| Neto tras deuda y cuotas | 56.136 | 77.953 |
| Avance | 8,3% | **11,5%** |
| Años al ritmo de ahorro | 26,8 | **24,9** |

Son **+3,2 puntos de avance y −1,9 años** (calculado, a confirmar contra el panel abierto). Los años son la parte que
importa: la anualidad compone el 5% real sobre el capital inicial, así que tomar la base en aportes proyecta
crecimiento sobre 21.866 USD que el mercado no reconoce y **adelanta la fecha de libertad casi dos años**. La brecha no
desaparece por no vender; solo se cierra si el mercado sube.

La tabla de arriba compara **solo el efecto de la base** y por eso las dos columnas restan los 411 que el bloque
restaba ese día. Con la deuda corregida en la misma tanda (abajo) y el número que ya incluye el interés de margen
(683.794), el estado vigente del bloque es **10,5% cubierto y 25,9 años** sobre un neto de 71.464.

**Por qué no se traduce el objetivo en vez del avance.** La alternativa era dejar el avance a mercado y llevar el
número a base aportes: `N_capital = N / k` = 678.111 / 0,727 ≈ 933.000. Es matemáticamente coherente y **devuelve
exactamente el mismo 8,3%** —dividir numerador y denominador por el mismo k no mueve el porcentaje—, así que no
responde lo que el usuario pidió: quería un solo valor, no dos lecturas del mismo.

**Cómo se declara en pantalla.** La resta del bloque PROGRESO arranca ahora en `Capital invertido` y termina en
`Capital propio hoy`; debajo de la barra hay una fila nueva:

```
Valor a mercado — k 0,73 (el mercado reconoce 73% de lo aportado)        57.847
```

Va en rojo mientras `k < 1`, por la misma razón que el techo: **son 21.866 USD que la barra de arriba no descuenta**, y
un avance que se calla lo que le falta miente. La fila del techo pasa a nombrar los dos motivos a la vez —
`TECHO — base en lo aportado y falta el saldo del préstamo (paso 1.6)`— y la barra de estado dice
`% cubierto sobre lo aportado`, para que la base no quede implícita en ningún lado.

**La deuda se sigue restando, y en la misma tanda se corrigió de dónde sale** (*"el (−) Deuda de margen lo estás
tomando mal — 6.9K"*). Restarla no se discute: el costo base de lo comprado con margen entra en `costobase` igual que
el resto, así que sin restarla el "capital propio" incluiría plata prestada. Lo que estaba mal era la fuente —
`SUM(inversion.deuda)`, 411 USD contra 6.900 reales. `FinancePanel._deuda_inversion()` lee ahora
`DataHub.manager_GyP["Stock"]["Debit"] + ["Crypto"]["Debit"]`, la **misma fuente que la barra `Deuda Total`** del panel
superior, con el mismo fallback a `kpi_snapshot` cuando el WebSocket de Stock está caído (sin él el margen vendría en 0
y el avance saldría inflado). Para Binance no hay snapshot: si el panel se abre antes del primer ciclo de
`Agente_LtvControl` la deuda queda corta y el avance optimista — consistente con el techo que el bloque ya declara.
La fila pasa a llamarse `(−) Deuda de inversión (margen IB + préstamo)`, porque son dos créditos y no uno.

**Hallazgo lateral, y es el más grave: el plan de inversión no mira el valor de mercado en ningún punto.** Los dos
indicadores de `trazaplan` miden contra el capital aportado, y `extractos.crecimiento` es
`Realized S/T Profit + Realized L/T Profit` (`Class_DataFrame.py:840`) — **solo realizado**:

| Indicador | Fórmula | Qué ve |
|---|---|---|
| `trazaplan.efectividad` | `(tinversion − vision) / vision` | aportes contra meta de aportes |
| `trazaplan.trendimiento` | `beneficios / costobase` | solo ganancia realizada |

Con eso un año puede cerrar `Cumplido` mientras el patrimonio cae, y los −21.866 no aparecen en ninguna fila del plan.
El dato para arreglarlo **ya viaja y se descarta**: `select_extracto("sum*")` trae `SUM(navcierre)` — el valor de cierre
de cada mes — y `update_plan()` usa solo `costobase` (`Class_gestion.py::update_plan()`). Queda en el BACKLOG: mostrar
`navcierre` junto a `tinversion` en el plan, para ver en qué año se abrió la brecha. No se hizo en este commit porque
toca el módulo del plan, que es otro dominio.

#### El interés de margen entra al costo de vida (2026-09-19)

Punto de partida del usuario: *"uso margen cuando veo que hay tendencia alcista, el activo promete. Solo recupero unos
USD. Mi idea es no exceder 7.000 USD"*, y después *"deberíamos manejar los números reales, eso me mantendría a raya
para no abusar del crédito; quiero que me pese en mi objetivo de ser libre financieramente"*.

**El mecanismo ya existía: no hacía falta un vigilante, hacía falta que el interés fuera un gasto.** Mientras haya
deuda, el interés se paga todos los meses, así que es costo de vida `fixed`. Con eso pega por los dos lados a la vez —
sube el número (×300) **y** baja el ahorro—, que es exactamente lo que se pidió. Un tope que avisa al cruzar un umbral
no cambia ningún número; este sí.

**El dato estaba en `extractos.imargen`**, no en el ledger. `Class_DataFrame` ya parsea la línea `Interest / Debit` del
extracto del broker y la guarda por mes y cuenta; es lo que se pagó de verdad, no una estimación sobre la tasa.
(`Class_Analisis._evaluar_apalancamiento()` lee el mismo concepto del ledger IB para los semáforos de riesgo, pero
ese camino es un snapshot MTD del momento en que se abre Análisis, no una serie.)

Medido sobre los 12 meses cerrados al 2026-09-19:

| Ventana | Costo de vida | Número | Ahorro/mes | Años |
|---|---|---|---|---|
| 1Y, antes | 2.260 | 678.111 | 712 | 26,8 |
| 1Y, con margen | 2.279 | **683.794** | 693 | **27,2** |

**El hallazgo no es el impacto, es que el progreso está restando la deuda equivocada.** `inversion.deuda` marca
**411 USD** y el bloque PROGRESO resta eso. Pero 411 al 6,5% daría 2,23 USD/mes de interés y el extracto cobra **~22**:
la deuda real es un orden de magnitud mayor.

*Corrección del mismo día, anotada porque el error estaba en el método.* Se despejó la deuda desde el interés
(~22 ÷ 6,5% → ~4.600) y se registró ese número. **Estaba mal y además sobraba:** el dato real ya estaba en pantalla,
en el KPI `Deuda Total` del panel superior, que marca **6.900 USD**. Sale de
`DataHub.manager_GyP["Stock"]["Debit"] + ["Crypto"]["Debit"]` (`DashMain.py`), el saldo en vivo del broker, con
fallback a `kpi_snapshot` en `tmp/` cuando el WebSocket está caído. Cuadra contra el KPI `Leverage (D/K %)`: 9% sobre
79,7K de capital ≈ 7,2K. La lección es de método: **antes de despejar un número con un parámetro supuesto, buscar si
alguna vista ya lo muestra medido** — es la misma regla que [[feedback_reutilizar_antes_de_calcular]], aplicada a los
datos y no solo a los métodos.

*Segunda corrección, del mismo día y por el mismo motivo.* Se leyó ese 6.900 como **99% del techo de 7.000 USD** del
usuario, y tampoco corresponde: **el KPI suma dos créditos de naturaleza distinta.** `Stock.Debit` es el margen de IB
(`Class_customer.py`: `abs(cash_balance)` cuando el cash es negativo, o `gross_pos − mktvalue`); `Crypto.Debit` es el
préstamo colateralizado de Binance (`Class_AgentManager.py`: `sum(loan_usd)`, lo escribe `Agente_LtvControl`). El techo
de 7.000 es sobre el margen de IB, así que **el consolidado no sirve para vigilarlo** y el reparto de los 6.900 no
está medido. No se despeja: se lee del header de cada pestaña, que ya publica `Debit` por vehículo.

**Para el progreso, en cambio, la suma es lo correcto:** las dos son plata prestada y ninguna es patrimonio. Efecto
sobre el progreso al restar la deuda consolidada en lugar de los 411:

| | Con 411 (lo que resta hoy) | Con 6.900 (real) |
|---|---|---|
| Patrimonio neto | 56.136 | **49.647** |
| Cubierto | 8,3% | **7,3%** |
| Años | 27,2 | **27,8** |

**El progreso ya lee la fuente del KPI** (implementado el 2026-09-20, `FinancePanel._deuda_inversion()`; ver arriba).
**Y ya se sabe por qué `inversion.deuda` sumaba 411** (2026-09-21): es la deuda de **Binance por posición**. El
usuario leyó el reparto del header de cada pestaña —**IB 6.462 + Binance 410**— y los 410 son el 411 de la columna. La
escribe `update_inversion()` desde `keys["deuda"]`, y el único camino que le pone algo distinto de 0 es el de Crypto
(`wallet["debit USDT"]`); el de Stock escribe `0` y FCI también, así que el margen de IB no figura en ninguna fila.
No es "la deuda con otro criterio": es una deuda distinta. Queda registrado en `CLAUDE.md` § "Columnas con semántica
propia" para que ningún consumidor nuevo la lea como saldo de la cartera.

**Por qué el bloque leyó bien los 6.900 con IB caído, y qué se ajustó después.**
`DashMain.actualizar_totales_inversiones()` refresca el KPI cada 30s y persiste el valor en `kpi_snapshot` cuando es
> 0. Ese día solo guardaba `stock_debit`: el margen de IB sobrevivió la desconexión por el snapshot, y los 410 de
Binance llegaron en vivo porque `Agente_LtvControl` no depende de IB. Alcanzaba, pero por dos caminos distintos, y el
de Binance se caía en el único hueco que tiene —los primeros 300s tras arrancar la app, antes del primer ciclo del
agente—. El 2026-09-21, a pedido del usuario y por homogeneidad, el snapshot guarda también `crypto_debit`.

El criterio para creerle a un 0 sigue siendo distinto en cada mitad, y tiene que serlo: en Stock se consulta
`DataHub.ws_stock_connected`, porque con el websocket vivo un 0 es real —no hay margen abierto— y caer al snapshot
mostraría deuda que ya se pagó; en Crypto no hace falta bandera porque `Agente_LtvControl` corta con `return` cuando la
API no devuelve préstamos, así que nunca escribe 0 y un 0 solo puede significar "todavía no midió". El mismo par de
reglas vive en `FinancePanel._deuda_inversion()`, que lee la misma fuente que la barra. *Probable, no confirmado:* que `inversion.deuda` se escriba solo para algunos vehículos o solo
en el ciclo de algunas posiciones.

**Y queda un hueco que el interés de margen dejó al descubierto: el costo del préstamo Crypto no entra al costo de
vida.** `get_interes_margen()` lee `extractos.imargen`, que es el extracto de IB y solo cubre el margen de IB. El
préstamo de Binance no tiene extracto equivalente en ninguna tabla y `Class_ApiBinnace.py` no registra interés en
ningún lado. Así que la mitad del crédito pesa en el objetivo y la otra mitad no. *Probable, no confirmado:* que el
préstamo colateralizado de Binance devengue interés que hoy no se guarda; hay que verificarlo contra el estado de
cuenta del préstamo antes de decidir de dónde leerlo.

**Tres decisiones de implementación:**

| Decisión | Por qué |
|---|---|
| No se escriben filas en `fin_transactions` | El interés no pasó por ninguna cuenta bancaria: el broker lo cobra contra el saldo. Escribirlo como movimiento bancario mentiría sobre su origen y chocaría con `uq_tx` en cada reimportación. Se calcula al vuelo |
| **No suma a `meses_con_dato`** | Los extractos del broker empiezan antes que la carga de Finanzas. Si sus meses contaran como "meses con movimiento", el divisor de 1Y pasaría de 8 a 12 y **diluiría todas las demás categorías**: el número bajaría por un motivo falso. El interés solo suma en meses que ya contaban |
| Categoría propia, no dentro de `Intereses y financiación` | Esa ya trae 91 USD/mes de tarjeta. Separadas se ve cuánto cuesta cada crédito; juntas no se sabe cuál mover |

**Reparto de responsabilidades, igual que con el patrimonio.** `extractos` es de la cartera, así que
`PlanInversion.get_interes_margen(meses=12)` devuelve la serie `{"YYYY-MM": usd}` y el panel se la pasa a
`get_category_flow()` por parámetro. El mes en curso queda afuera: su extracto todavía no cerró y el parcial leería
como un mes barato. Falta `2026-03` en la serie — o no hubo deuda ese mes, o el extracto no está cargado; se resuelve
con la cobertura de extractos, no acá.

**Lo que esto no cubre.** El interés mide la deuda *promedio* del mes, no el pico ni el saldo de hoy, así que sigue sin
haber un número que diga "hoy debés X de los 7.000". Eso es el paso 1.6 otra vez: mientras `fin_accounts.balance` esté
en 0 y el saldo del préstamo no esté en ninguna tabla `fin_*`, el neto del progreso es un techo.

**Dónde se ve.** Cuarta vista del panel de detalle (`Flujo`), junto a `Últimas transacciones`, `Cobertura` y
`Comprometido`. Con la cuarta vista, el `pack_forget` de las demás pasó a `_pack_solo()`: repetirlo en cada `_show_*`
es la forma de que alguna quede visible abajo de otra. El estado sigue siendo un solo `self._detail`
(`txn` / `cover` / `commit` / `flow`). El consultón es `FinanceScreen.get_category_flow()`: **una** query agrupada por
(grupo, clase, categoría, mes) y los cuatro promedios armados en Python — cuatro queries darían cuatro recorridos de
la misma tabla para responder la misma pregunta.

------------------------------------------------------------------------

## 1. Objetivo

Dos proyectos que comparten la misma integración Gmail y trabajan juntos:

**A) Finanzas personales:**
Capturar, clasificar y analizar gastos de forma automatizada. La fuente principal
son los extractos que los bancos mandan por mail — cero fricción para el usuario.
El sistema aprende de cada corrección.

**B) Organización Gmail (co-work):**
Limpiar la bandeja de entrada, eliminar spam recurrente, clasificar por remitente
y crear filtros automáticos. Es el paso previo que además habilita la captura
de extractos bancarios.

**Moneda de referencia:** USDT (dólar cripto) — unifica cuentas en VES, ARS y USD.
Los rates se obtienen de Binance (ya integrado en AppOO). No se usa tipo oficial
ni blue; solo el precio de mercado cripto.

------------------------------------------------------------------------

## 2. Arquitectura

### Submódulos

1. **Gmail integration** ← punto de entrada de todo
2. **Importación de extractos** (desde adjuntos de mail o carpeta vigilada)
3. Clasificación automática (reglas + IA)
4. Registro financiero (manual como complemento)
5. Análisis y métricas / hábitos de consumo
6. Planificación y objetivos
7. Motor de decisiones / alertas

### Cuentas en scope

| País      | Tipo                    | Moneda | Notas                               |
|-----------|-------------------------|--------|-------------------------------------|
| Venezuela | Cuenta corriente        | VES    | Alta volatilidad — rate diario      |
| Argentina | Cuentas de ahorro       | ARS    | Brecha cambiaria — usar rate cripto |
| Argentina | Tarjetas de crédito     | ARS    |                                     |
| EEUU      | Cuenta (tipo por definir)| USD   | Base natural en dólares             |
| Binance   | Spot / DeFi / Préstamos | USDT  | Ya integrado en AppOO               |

### Flujo principal

```
Extracto (CSV/PDF/Excel)
    └── Parser por banco (adapter)
            └── Normalizar → deduplicar
                    └── Clasificar (reglas → IA → manual)
                            └── transactions
                                    └── Convertir a USDT (via exchange_rates ← Binance)
                                            └── Métricas / Hábitos / Alertas
```

### Flujo cripto como puente de transferencia

```
ARS / VES
    └── Compra USDT en Binance
            └── Transfer a Venezuela o EEUU
                    └── Registrar como: expense(origen) + income(destino) + transfer en Binance
```
*(Las operaciones Binance ya están en AppOO — se linkearán en Fase 5)*

------------------------------------------------------------------------

## 3. Fuentes de datos y automatización

### Fuentes por prioridad

| Fuente | Mecanismo | Automatización | Cuándo usar |
|--------|-----------|---------------|-------------|
| **Gmail** | OAuth → leer adjuntos de mails bancarios | Total — agente periódico | Bancos que mandan extracto por mail |
| **Carpeta vigilada** | Drop en `data/extractos/` → app procesa sola | Alta — solo bajar del portal | Bancos sin adjunto en mail |
| **Manual** | Subir archivo directo en UI | Ninguna | Casos puntuales |

### Gmail — flujo de captura automática

```
Agente periódico (cada 24h)
    └── Gmail API (OAuth)
            └── Buscar mails de remitentes bancarios conocidos
                    └── Descargar adjunto (PDF o CSV)
                            └── Pipeline normal: parser → deduplicar → clasificar → transactions
```

Remitentes bancarios se configuran en `banks.gmail_sender` (ej: `noreply@bancolombia.com.co`).

### Gmail — organización (co-work)

```
Script de depuración (se corre una vez, luego mantenimiento liviano)
    ├── Listar remitentes con >N mails → proponer etiquetar o marcar spam
    ├── Detectar newsletters/publicidad → crear filtro "skip inbox + etiquetar"
    ├── Identificar mails bancarios → etiquetar "Bancos" + archivar tras procesar
    └── Resumen: X mails archivados, Y filtros creados, Z spam eliminado
```

------------------------------------------------------------------------

## 4. Seguridad

### Principios

- **Todo local**: MySQL en la misma máquina que AppOO. No hay sincronización cloud.
- **OAuth para Gmail**: Google maneja la autenticación. La app guarda un token
  (archivo `token_gmail.json`) — nunca la contraseña. El token se puede revocar
  desde la cuenta Google en cualquier momento.
- **Sin contraseñas en texto plano**: credenciales en variables de entorno o
  en la tabla `sesiones` ya usada por AppOO (encriptada).
- **Permisos mínimos Gmail**: solo los scopes necesarios.
  - Para leer mails y adjuntos: `gmail.readonly`
  - Para mover/etiquetar: `gmail.modify`
  - NO se pide `gmail.send` ni acceso a otros servicios Google.

### Scopes OAuth requeridos

| Scope | Para qué |
|-------|----------|
| `https://www.googleapis.com/auth/gmail.readonly` | Leer mails y descargar adjuntos |
| `https://www.googleapis.com/auth/gmail.modify` | Etiquetar, archivar, crear filtros |

### Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|-----------|
| `token_gmail.json` expuesto | Agregar a `.gitignore`, permisos de archivo restrictivos |
| Datos financieros en BD local | MySQL solo acepta conexiones localhost; sin puerto abierto |
| Extracto mal parseado genera datos erróneos | Campo `needs_review=True` + deduplicación por hash |
| Spam malicioso con adjunto falso | Solo procesar adjuntos de remitentes en whitelist (`banks.gmail_sender`) |

------------------------------------------------------------------------

## 5. Modelo de Datos

### 5.1 exchange_rates — tipos de cambio históricos a USDT

Se popula automáticamente desde Binance (precio spot del par correspondiente).
Se guarda un rate por moneda por día para poder convertir transacciones pasadas
con el rate correcto de la fecha, no el rate de hoy.

| Columna       | Tipo     | Descripción                                           |
|---------------|----------|-------------------------------------------------------|
| id            | INT PK   |                                                       |
| from_currency | CHAR(3)  | ARS, VES, USD, etc.                                   |
| to_currency   | CHAR(4)  | Siempre USDT (moneda base)                            |
| rate          | DECIMAL  | 1 unidad de from_currency = X USDT                   |
| date          | DATE     | Fecha del rate (un registro por moneda por día)       |
| source        | ENUM     | binance / manual                                      |
| pair          | VARCHAR  | Par Binance usado (ej: ARSUSDT, USDTARS invertido)    |

**Notas:**
- VES: Binance P2P es la referencia (no hay par spot directo) → carga manual o scraping. Las órdenes C2C propias
  ya llegan por API con su precio (§8, corrección 2026-09-13)
- ARS: par USDTARS en Binance P2P → invertir para obtener ARS/USDT
- USD: rate = 1.0 siempre (USD ≈ USDT para este propósito)
- Si no hay rate para una fecha, usar el rate más cercano disponible

---

### 5.2 banks — catálogo de bancos/tarjetas soportados

| Columna        | Tipo    | Descripción                                      |
|----------------|---------|--------------------------------------------------|
| id             | INT PK  |                                                  |
| name           | VARCHAR | Nombre del banco / tarjeta                       |
| country        | CHAR(2) | CO, US, etc.                                     |
| adapter_class  | VARCHAR | Nombre de la clase parser (ej: BancolombiaCsv)   |
| date_format    | VARCHAR | Formato de fecha en el extracto (ej: %d/%m/%Y)   |
| delimiter      | CHAR    | Separador CSV (, o ;)                            |
| encoding       | VARCHAR | utf-8, latin-1, etc.                             |
| currency       | CHAR(3) | Moneda por defecto (COP, USD, etc.)              |
| gmail_sender   | VARCHAR | Remitente autorizado (ej: noreply@banco.com) — whitelist de seguridad |
| notes          | TEXT    | Instrucciones de descarga / columnas esperadas   |
| is_active      | BOOL    |                                                  |

---

### 5.3 accounts — cuentas y tarjetas del usuario

| Columna             | Tipo    | Descripción                                 |
|---------------------|---------|---------------------------------------------|
| id                  | INT PK  |                                             |
| name                | VARCHAR | Alias (ej: "Visa Bancolombia")              |
| type                | ENUM    | checking / credit / savings / investment    |
| currency            | CHAR(3) |                                             |
| balance             | DECIMAL | Saldo actual                                |
| opening_balance     | DECIMAL | Saldo inicial al crear la cuenta            |
| credit_limit        | DECIMAL | Solo tarjetas de crédito                    |
| bank_id             | INT FK  | → banks                                     |
| account_number_last4| CHAR(4) | Últimos 4 dígitos (identificación rápida)   |
| institution         | VARCHAR | Nombre institución (puede diferir del banco)|
| is_active           | BOOL    |                                             |

---

### 5.4 statement_imports — registro de extractos importados

Controla qué archivos ya se procesaron para evitar doble importación.

| Columna          | Tipo     | Descripción                                      |
|------------------|----------|--------------------------------------------------|
| id               | INT PK   |                                                  |
| account_id       | INT FK   | → accounts                                       |
| bank_id          | INT FK   | → banks                                          |
| filename         | VARCHAR  | Nombre original del archivo                      |
| file_hash        | CHAR(64) | SHA-256 del archivo — deduplicación exacta       |
| period_from      | DATE     | Primera fecha del extracto                       |
| period_to        | DATE     | Última fecha del extracto                        |
| imported_at      | DATETIME | Cuándo se importó                                |
| row_count        | INT      | Total de filas en el archivo                     |
| processed_count  | INT      | Filas convertidas a transacciones                |
| skipped_count    | INT      | Filas ignoradas (duplicadas o sin monto)         |
| status           | ENUM     | pending / processed / error                      |
| error_log        | TEXT     | Detalle de errores de parseo                     |

---

### 5.5 categories — categorías de gasto/ingreso

> **Diseño original.** La tabla real `fin_categories` difiere desde 2026-09-13: sin `type` ni jerarquía, con
> `category_type` y `expense_class`. El mapa vigente está en §0 → "Mapa de categorías".

| Columna   | Tipo    | Descripción                                          |
|-----------|---------|------------------------------------------------------|
| id        | INT PK  |                                                      |
| name      | VARCHAR | Ej: Alimentación, Transporte, Entretenimiento        |
| type      | ENUM    | income / expense / transfer                          |
| parent_id | INT FK  | → categories (subcategorías, ej: Restaurantes ⊂ Alimentación) |
| color     | CHAR(7) | Hex para UI (#FF5733)                               |
| icon      | VARCHAR | Nombre de ícono (ej: "fork", "car", "tv")           |
| is_active | BOOL    |                                                      |

Categorías base sugeridas (expense):
- Alimentación > Supermercado, Restaurantes, Delivery
- Transporte > Combustible, Taxi/Uber, Transporte público
- Hogar > Arriendo, Servicios, Mantenimiento
- Salud > Médico, Farmacia, Seguro
- Entretenimiento > Streaming, Eventos, Hobbies
- Educación
- Ropa y calzado
- Tecnología
- Viajes
- Finanzas > Cuotas crédito, Intereses, Comisiones
- Otros

Categorías base (income):
- Salario
- Freelance / Honorarios
- Inversiones / Dividendos
- Arriendos recibidos
- Otros ingresos

---

### 5.6 import_rules — motor de clasificación automática

Cada fila es una regla que mapea un patrón de descripción a una categoría.
El sistema las evalúa en orden de prioridad. Si ninguna aplica, consulta IA.

| Columna      | Tipo    | Descripción                                              |
|--------------|---------|----------------------------------------------------------|
| id           | INT PK  |                                                          |
| pattern      | VARCHAR | Texto a buscar en raw_description (ej: UBER, NETFLIX)    |
| match_type   | ENUM    | exact / contains / startswith / regex                    |
| category_id  | INT FK  | → categories                                             |
| priority     | INT     | Menor número = mayor prioridad. Reglas de usuario > IA   |
| is_active    | BOOL    |                                                          |
| created_by   | ENUM    | user / ai / system                                       |
| hit_count    | INT     | Veces que esta regla clasificó una transacción           |
| last_hit_at  | DATETIME|                                                          |
| created_at   | DATETIME|                                                          |

**Flujo de clasificación:**
1. Evaluar reglas `created_by='user'` por prioridad → si match, asignar categoría
2. Evaluar reglas `created_by='system'` (precargadas)
3. Si no hay match → enviar `raw_description` a IA → IA devuelve categoría sugerida
4. Guardar sugerencia como regla nueva `created_by='ai'` con `is_active=False`
5. Usuario confirma o corrige → la regla queda `is_active=True`

---

### 5.7 transactions — transacciones (fuente central)

| Columna                  | Tipo     | Descripción                                      |
|--------------------------|----------|--------------------------------------------------|
| id                       | INT PK   |                                                  |
| date                     | DATE     | Fecha de la transacción                          |
| type                     | ENUM     | income / expense / transfer                      |
| amount                   | DECIMAL  | Siempre positivo                                 |
| currency                 | CHAR(3)  |                                                  |
| amount_base              | DECIMAL  | Monto convertido a moneda base (para métricas)   |
| category_id              | INT FK   | → categories                                     |
| account_id               | INT FK   | → accounts                                       |
| description              | VARCHAR  | Descripción limpia (editada por usuario)         |
| raw_description          | VARCHAR  | Texto original del extracto — nunca editar       |
| import_id                | INT FK   | → statement_imports (NULL si es manual)          |
| classified_by            | ENUM     | rule / ai / manual                               |
| classification_confidence| FLOAT    | 0.0–1.0 (solo si classified_by='ai')            |
| is_recurring             | BOOL     | Detectado automáticamente o marcado por usuario  |
| recurring_group_id       | INT      | Agrupa pagos recurrentes del mismo origen        |
| tags                     | VARCHAR  | Etiquetas libres separadas por coma              |
| notes                    | TEXT     | Nota libre del usuario                           |
| needs_review             | BOOL     | True si clasificación pendiente de confirmar     |
| created_at               | DATETIME |                                                  |
| updated_at               | DATETIME |                                                  |

**Deduplicación de transacciones:**
Al importar, antes de insertar se verifica:
`(account_id, date, amount, raw_description)` → si ya existe, se marca como `skipped`.

---

### 5.8 budgets — presupuestos por categoría

| Columna         | Tipo    | Descripción                      |
|-----------------|---------|----------------------------------|
| id              | INT PK  |                                  |
| category_id     | INT FK  |                                  |
| monthly_limit   | DECIMAL |                                  |
| alert_threshold | FLOAT   | Ej: 0.80 → alerta al 80% del límite |
| currency        | CHAR(3) |                                  |
| is_active       | BOOL    |                                  |

---

### 5.9 goals — objetivos financieros

| Columna            | Tipo    | Descripción                                  |
|--------------------|---------|----------------------------------------------|
| id                 | INT PK  |                                              |
| name               | VARCHAR | Ej: "Viaje a Europa", "Fondo de emergencia"  |
| target_amount      | DECIMAL |                                              |
| current_amount     | DECIMAL |                                              |
| deadline           | DATE    |                                              |
| priority           | INT     |                                              |
| type               | ENUM    | savings / purchase / emergency / debt_payoff |
| status             | ENUM    | active / completed / paused / cancelled      |
| linked_account_id  | INT FK  | Cuenta donde se acumula el ahorro            |
| monthly_required   | DECIMAL | Calculado: (target - current) / meses_restantes |

---

### 5.10 savings_rules — reglas de ahorro automático

| Columna             | Tipo    | Descripción                                      |
|---------------------|---------|--------------------------------------------------|
| id                  | INT PK  |                                                  |
| rule_type           | ENUM    | percentage / fixed_amount / round_up             |
| value               | DECIMAL | % o monto fijo                                   |
| trigger             | ENUM    | on_income / monthly / on_transaction             |
| source_account_id   | INT FK  |                                                  |
| destination_account_id | INT FK |                                               |
| goal_id             | INT FK  | Objetivo al que aplica (NULL = ahorro general)   |
| is_active           | BOOL    |                                                  |
| frequency           | ENUM    | daily / weekly / monthly (si trigger=monthly)    |

---

### 5.11 daily_financial_state — snapshot diario (histórico)

Se popula automáticamente al final de cada día con un proceso batch.
Base para análisis histórico y predicciones.

| Columna                   | Tipo    | Descripción                         |
|---------------------------|---------|-------------------------------------|
| id                        | INT PK  |                                     |
| date                      | DATE    |                                     |
| account_id                | INT FK  |                                     |
| balance_eod               | DECIMAL | Saldo al cierre del día             |
| daily_income              | DECIMAL |                                     |
| daily_expense             | DECIMAL |                                     |
| cumulative_income_month   | DECIMAL | Acumulado desde inicio de mes       |
| cumulative_expense_month  | DECIMAL |                                     |
| savings_rate_month        | FLOAT   | % ahorro acumulado del mes          |
| top_category_id           | INT FK  | Categoría con más gasto del día     |

------------------------------------------------------------------------

## 4. Pipeline de importación (detalle técnico)

### Paso 1 — Parser por banco (adapter)

Cada banco tiene su clase adapter que hereda de `StatementParser`:

```
StatementParser (base)
    ├── BancolombiaCreditoCsv
    ├── DaviviendaCsv
    ├── NubankCsv
    └── ... (agregar según necesidad)
```

Responsabilidad del adapter:
- Leer el archivo (CSV/PDF/Excel)
- Devolver lista de dicts normalizados:
  `[{date, raw_description, amount, type, currency}, ...]`

### Paso 2 — Normalización

- Limpiar `raw_description`: strip, uppercase, remover caracteres especiales
- Detectar si es gasto o ingreso (por signo del monto o columna específica)
- Convertir fecha al formato estándar
- Calcular `amount_base` si la moneda difiere de la moneda base del usuario

### Paso 3 — Deduplicación

- Calcular SHA-256 del archivo → si `file_hash` ya existe en `statement_imports`, abortar
- Por transacción: verificar `(account_id, date, amount, raw_description)` en `transactions`

### Paso 4 — Clasificación

Ver flujo en sección 3.5 (import_rules)

### Paso 5 — Revisión pendiente

Todas las transacciones con `needs_review=True` aparecen en una cola de revisión
en la UI. El usuario las procesa una vez y el sistema aprende (regla nueva).

------------------------------------------------------------------------

## 5. Métricas y análisis de hábitos

*(Detalle en próxima iteración — primero consolidar captura de datos)*

Áreas previstas:
- Ratio de ahorro mensual
- Gasto por categoría (mes actual vs promedio histórico)
- Detección de gastos recurrentes
- Tendencias: categorías que suben/bajan
- Días/semanas de mayor gasto
- Fondo de emergencia: meses de runway
- Score financiero personal (ahorro 40 / control gasto 30 / estabilidad 20 / objetivos 10)

------------------------------------------------------------------------

## 6. Alertas

*(Detalle en próxima iteración)*

Canales: UI (notificación en app) + Telegram (bot existente en AppOO)

Tipos previstos:
- Presupuesto de categoría al 80% / 100%
- Mes con flujo negativo
- Gasto inusual (>2x promedio histórico en una categoría)
- Transacciones pendientes de clasificar > N
- Objetivo financiero en riesgo (ritmo actual no alcanza el deadline)

------------------------------------------------------------------------

## 7. Planificación de compras

```
ahorro_mensual_necesario = (precio - ahorro_actual) / meses_restantes
```

------------------------------------------------------------------------

## 8. Integración con AppOO (inversiones + Binance)

### Señales de control (Fase 4)
- Si `savings_rate < 10%` → señal en Screener: pausar nuevas posiciones
- Si `balance_total_usdt < fondo_emergencia_minimo` → bloquear compras automáticas BotCrypto
- Dividendos recibidos → registrar como `income` en `transactions` automáticamente

### Binance como fuente de tipos de cambio
- Agente en `Class_DashBot.py` con `@wait_rate(86400)` → descarga precio spot
  de los pares relevantes (ARSUSDT, VESUSDT vía P2P si disponible) y popula `exchange_rates`
- Para VES: ~~P2P de Binance no tiene API pública → carga manual o scraping del portal~~
  **Corregido 2026-09-13:** las órdenes C2C propias sí salen por API firmada
  (`/sapi/v1/c2c/orderMatch/listUserOrderHistory`, con `unitPrice` por orden) y desde esa fecha se cargan solas
  (§0 "Extractos y cargas automáticas"). Lo que sigue sin API documentada es el libro público de anuncios P2P:
  la tasa sale de las operaciones propias, no del mercado.

### Binance DeFi — integración futura (Fase 4)
- Operaciones Spot ya registradas en AppOO → mapear a `transactions` con
  `type='transfer'` cuando sean envíos VE/US
- Earn/DeFi: intereses recibidos → `type='income'`, `category='Inversiones/Dividendos'`
- Préstamos (loan): desembolso → `type='income'`; cuota/repago → `type='expense'`,
  `category='Finanzas/Cuotas crédito'`
- Las operaciones cripto que sean **puente de transferencia** (ARS→USDT→VES o ARS→USDT→USD)
  se registran como transfer entre cuentas, no como gasto

------------------------------------------------------------------------

## 9. Roadmap

> **Filosofía:** proyecto de segundo plano. Una fase a la vez, solo cuando haya
> tiempo disponible. Cada fase debe poder usarse sola sin depender de la siguiente.
> Preferir scripts independientes antes que integrar a AppOO — integrar solo cuando
> la funcionalidad ya esté probada y estable.

---

### Fase 0 — Gmail: organización + acceso *(punto de entrada)*

**Objetivo:** limpiar el mail y dejar la integración Gmail lista para todo lo demás.
Es el paso que habilita la captura automática de extractos.

Entregables:
1. Configurar proyecto en Google Cloud Console → obtener `credentials.json` (OAuth)
2. Script `gmail_setup.py` en `AppTest/`: autorizar app, guardar `token_gmail.json`
3. Script `gmail_cleanup.py`:
   - Listar top remitentes por volumen → proponer etiquetar o marcar spam
   - Detectar newsletters/publicidad → crear filtros "skip inbox + archivar"
   - Identificar mails de bancos conocidos → etiquetar "Bancos"
   - Resumen final: X mails procesados, Y filtros creados
4. Whitelist de remitentes bancarios: primer llenado manual de `banks.gmail_sender`

**Resultado esperado:** bandeja organizada + app con acceso Gmail funcionando.
No se toca la BD de finanzas todavía.

**Pendiente de confirmar:** ¿qué bancos mandan extracto/resumen por mail?
¿El adjunto viene en PDF, CSV o es solo texto en el cuerpo del mail?

---

### Fase 1 — Cimientos: tablas + primer extracto manual

**Objetivo:** poder cargar un extracto y ver los gastos clasificados en la BD.
Sin UI, sin automatización — solo que los datos entren correctos.

Entregables:
1. Crear las tablas en MySQL: `banks`, `accounts`, `exchange_rates`, `categories`,
   `import_rules`, `statement_imports`, `transactions`
2. Reglas base precargadas en `import_rules` (50–100 patrones comunes: UBER,
   NETFLIX, supermercados, etc.)
3. Adapter para el primer banco con extracto disponible (definir en esta fase)
4. Script `load_statement.py` en `AppTest/`: recibe archivo CSV/PDF,
   parsea, deduplica e inserta en `transactions`
5. Script de verificación: últimas N transacciones + cuántas quedaron `needs_review`

**Resultado esperado:** cargar un extracto real y ver los datos en la BD.

---

### Fase 2 — Gmail: captura automática de extractos

**Objetivo:** que los extractos se carguen solos cuando llega el mail del banco.

Entregables:
1. Script `gmail_fetch_statements.py`: busca mails de remitentes en whitelist,
   descarga adjuntos a `data/extractos/`, llama al pipeline de la Fase 1
2. Agente periódico (24h) en `Class_DashBot.py` que lo ejecuta
3. Log de lo procesado: qué mail, qué extracto, cuántas transacciones nuevas

**Resultado esperado:** no tener que hacer nada — los gastos aparecen solos.

---

### Fase 3 — Clasificación asistida

**Objetivo:** reducir a cero las transacciones `needs_review`, con mínimo esfuerzo.

Entregables:
1. Script `review_pending.py`: lista pendientes en consola, usuario escribe
   categoría → se guarda como regla nueva automáticamente
2. Clasificación por IA (Claude): para lo que no matchea reglas, sugiere
   categoría → usuario confirma con una tecla
3. Detección de recurrentes: mismo patrón + monto similar ~30 días → `is_recurring=True`

**Resultado esperado:** después de revisar una vez, los extractos futuros se
clasifican solos en >90%.

---

### Fase 4 — Métricas básicas (primer dashboard)

**Objetivo:** responder "¿en qué gasté este mes?" con una pantalla simple.

Entregables:
1. Tab nuevo en AppOO: "Finanzas" — gastos del mes agrupados por categoría
2. Comparación mes actual vs mes anterior
3. Total ingresos / gastos / % ahorro
4. Highlight: categorías con gasto >20% respecto al mes anterior

**Resultado esperado:** mirar el tab una vez por semana y entender el estado
financiero en 30 segundos.

---

### Fase 5 — Multi-país y tipos de cambio

**Objetivo:** unificar VES + ARS + USD + USDT en una sola vista.

Entregables:
1. Adapters para bancos venezolanos y argentinos
2. Agente que popula `exchange_rates` diariamente desde Binance
3. `amount_base` (USDT) calculado al importar
4. Dashboard muestra totales consolidados en USDT

---

### Fase 6 — Alertas e integración AppOO

**Objetivo:** que la app avise cuando algo está mal, sin tener que ir a buscar.

Entregables:
1. Alertas vía Telegram: flujo negativo, categoría sobre presupuesto
2. Señal a Screener/BotCrypto si `savings_rate` baja del umbral configurado
3. Linkear operaciones Binance existentes como transferencias entre cuentas

---

### Fase 6 — Planificación y objetivos *(long term)*

- Objetivos financieros con seguimiento (`goals`)
- Ahorro automático por reglas (`savings_rules`)
- Predicción de flujo futuro (`daily_financial_state`)
- Motor de decisiones completo
