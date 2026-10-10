---
tipo: plan
tags: [gestion, finanzas, viaje]
creado: 2026-09-27
estado: en carga
actualizado: 2026-09-27
---

# Plan de gastos del viaje

Hoja de trabajo para decidir **con qué tarjeta pagar cada cosa y en qué mes cae la plata**, y para comparar lo
estimado contra lo real cuando el viaje termine. Los datos de contexto salen de `bdinv` (tablas `fin_*`)
medidos el **2026-09-27**; lo del viaje se completa a mano.

Documento de trabajo: se edita mientras el viaje se planifica y se cierra con la comparación estimado vs real.
Los números del sistema se refrescan como indica la sección 6.

---

## 1. Lo que falta definir ⬅ completar primero

| Dato | Valor |
|---|---|
| Destino | **Venezuela** |
| Alojamiento | **Casa propia** — no hay gasto |
| Fechas (ida / vuelta) | ‹completar› |
| Personas | ‹completar› |
| Moneda en destino | USD (efectivo) |
| Ya consumido | **Boleto, DESPEGAR 08/09, 1.447.633,02 ARS ≈ 912 USD** — tarjeta BBVA Visa Signature, financiado en 6 cuotas |
| Ya consumido | **Seguro Assist Card, 27/09, 453.612 ARS ≈ 294 USD** — Santander Visa, 1 cuota (20% off promo Santander) |

El boleto ya está gastado: no es una decisión, es un hecho que todavía no se pagó. Lo que sigue abierto es
**cómo** se paga (sección 4) y qué falta gastar.

---

## 2. El terreno: cuándo se paga lo que gasto

**La regla que manda:** un gasto con tarjeta no se paga cuando se hace, se paga en el vencimiento del resumen
que lo factura. Elegir la tarjeta **es** elegir el mes en que sale la plata.

### Ciclos vigentes (`fin_card_cycles`, último resumen cargado de cada tarjeta)

| Tarjeta | Último cierre | Su vencimiento | Próximo cierre | Su vencimiento |
|---|---|---|---|---|
| Santander Visa (TC-0925) | 24/09 | 05/10 | **22/10** | **02/11** |
| Santander AmEx (TC-9541) | 24/09 | 06/10 | **22/10** | **03/11** |
| BBVA Mastercard Black | 27/08 | 07/09 | **01/10** | **09/10** |
| BBVA Visa Signature | 27/08 | 07/09 | **01/10** | **09/10** |

**Consecuencia directa para un gasto hecho hoy (27/09):**

- **Con BBVA** → cierra el 01/10 y se paga el **09/10**. Doce días.
- **Con Santander** → cierra el 22/10 y se paga el **02-03/11**. Treinta y siete días.

Son **25 días de diferencia** por elegir el plástico. Para un gasto grande de viaje esa elección pesa más que
casi cualquier otra optimización de esta hoja.

Las cuatro tarjetas son **cuentas en ARS**, pero tres registran consumo en dólares por movimiento: AmEx 34
movimientos por 363 USD, BBVA Mastercard 19 por 82 USD, BBVA Visa 3 por 75 USD. El consumo en el exterior
**queda registrado en su moneda** y el panel lo convierte; no hay que estimarlo aparte.

### Lo que ya está comprometido antes de gastar un peso del viaje

Cuotas de planes vigentes que todavía no aparecieron en ningún resumen (`get_installments_pending`):

| Mes | Cuotas comprometidas |
|---|---|
| 2026-10 | 216,47 USD |
| 2026-11 | 188,90 USD |
| 2026-12 | 188,90 USD |
| 2027-01 | 158,79 USD |
| 2027-02 | 79,52 USD |
| 2027-03 | 79,52 USD |

Total: **1.082 USD en 27 cuotas**. Si el viaje se paga en cuotas, se apila **arriba** de esto.

### Costo de vida base, medido

Promedio de los cinco meses completos (abril a agosto 2026), gasto clasificado:

| Clase | Promedio mensual |
|---|---|
| `fixed` | 2.062 USD |
| `variable` | 604 USD |
| **Base** | **2.666 USD/mes** |

Septiembre da 1.216 USD porque **faltan los resúmenes de Santander** — no es un mes barato, es un mes a medio
cargar. `extraordinary` queda afuera del promedio a propósito: es la clase donde cae el viaje (categoría
`Viajes`, con 57 USD en lo que va de 2026).

---

## 3. Presupuesto del viaje

Una fila por rubro. **`Mes de pago` no se escribe a mano**: sale de cruzar la fecha del gasto con el ciclo de
la tarjeta de la sección 2.

| # | Rubro | Estimado USD | Estado | Medio de pago | Cuándo sale la plata |
|---|---|---|---|---|---|
| 1 | **Boleto** (DESPEGAR, 1.447.633,02 ARS) | **912** | ✅ gastado 08/09 | BBVA Visa Signature | **6 cuotas de 276.399 ARS** (≈174 USD) — sección 4.1 |
| 2 | **Alquiler de vehículo** | **700** | pendiente | ‹a definir› | en destino |
| 3 | **Varios** (comida, nafta, imprevistos) | **500** | pendiente | ‹a definir› | en destino |
| 4 | **Seguro de viajero** (Assist Card, 453.612 ARS) | **294** | ✅ comprado 27/09 | Santander Visa, 1 cuota | **02/11** (cierre 22/10) |
| 5 | Alojamiento | **0** | — | — | casa propia |
| | **Total del viaje** | **2.406** | | | |

De ese total, **1.206 USD ya están gastados** (boleto + seguro) y 1.200 todavía no. A eso se suman
≈133 USD de intereses nominales por financiar el boleto (sección 4.1).

Medios de pago posibles: `Santander Visa`, `Santander AmEx`, `BBVA MC Black`, `BBVA Visa`, `débito`,
`efectivo USD`, `transferencia`.

### El resumen BBVA que cierra el 01/10 y vence el 09/10

Leído de las capturas del homebanking del 27/09 (todavía **no** está en la base: el último movimiento BBVA
cargado es del 27/08, así que estos números no salen de `fin_transactions`).

| Tarjeta | Consumos del ciclo | ≈ USD | De eso, el viaje |
|---|---|---|---|
| BBVA Visa Signature | 1.790.868 ARS | **1.128** | 912 (el boleto) |
| BBVA Mastercard Black | 479.504 ARS | **302** | 0 |
| **Total a pagar el 09/10** | **2.270.372 ARS** | **1.430** | |

**Ese vencimiento es el problema del plan, no el viaje.** 1.430 USD en un mes contra un costo de vida base de
2.666 USD/mes: el boleto solo se lleva el 64% de ese resumen. Lo que se decida en la sección 4 decide si
octubre es un mes duro o uno normal.

### Impacto por mes

| Mes | Base | Cuotas ya comprometidas | Viaje | Total |
|---|---|---|---|---|
| 2026-10 | 2.666 | 216 | 174 (cuota 1 boleto)* | 3.056 |
| 2026-11 | 2.666 | 189 | 174 + 294 (seguro) | 3.323 |
| 2026-12 | 2.666 | 189 | 174 | 3.029 |
| 2027-01 | 2.666 | 159 | 174 | 2.999 |
| 2027-02 | 2.666 | 80 | 174 | 2.920 |
| 2027-03 | 2.666 | 80 | 174 | 2.920 |

\* Confirmar en el resumen si la cuota 1 cae en octubre o en noviembre; si cae en noviembre, el plan corre
de noviembre a abril. Alquiler y varios (1.200 USD) van en destino y no están en esta tabla.

---

## 4. Decisiones abiertas

### 4.1 — El boleto: un pago el 09/10, o 12 cuotas

Todo depende de la tasa que ofrezca BBVA, y **ese dato todavía no lo tengo**. Los dos escenarios, sobre
1.447.633,02 ARS:

| Escenario | Cuota | En USD de hoy | Total pagado | Sobre el capital |
|---|---|---|---|---|
| **Un pago el 09/10** | 1.447.633 | 912 USD | 1.447.633 ARS | 1,00× |
| **12 cuotas sin interés** (Cuota Simple) | 120.636 | 76 USD/mes | 1.447.633 ARS | 1,00× |
| **12 cuotas TNA 79%** | 178.230 | 112 USD/mes | 2.138.760 ARS | **1,48×** |

La TNA 79% no es un supuesto suelto: es la que BBVA le cobró al `VISA PLAN V 6-06` que ya está en curso.

**Cómo se decide:**

- **Sin interés → financiar, sin dudarlo.** La cuota queda fija en pesos mientras el peso se devalúa: cada
  cuota vale menos en dólares que la anterior, y los 912 USD nominales terminan costando bastante menos.
- **TNA 79% → hay que comparar contra la devaluación esperada.** Se paga 1,48× el capital en pesos; solo
  conviene si el dólar sube más que eso en el año. Es una apuesta, no una cuenta.
- **El punto que no aparece en la tabla:** pagar los 912 USD de una vez el 09/10 significa sacarlos de algún
  lado en octubre. Si la alternativa es vender cartera o tocar el margen de IB —que está en 6.462 USD contra
  un techo autoimpuesto de 7.000—, financiar es más barato que eso aunque la tasa no sea regalada.

### 4.1 — Resultado (27/09)

Oferta real de BBVA sobre 1.447.633,02 ARS — **los tres plazos tienen la misma tasa: 4,03% mensual,
TNA 48,3%, TEA 60,6%**:

| Plan | Cuota | Total | Interés nominal | Costo real (desc. inflación REM ~1,75%/mes) |
|---|---|---|---|---|
| **6 cuotas** ✅ | 276.399 | 1.658.394 | 210.761 | ≈113.700 ARS ≈ **72 USD** |
| 9 cuotas | 194.939 | 1.754.451 | 306.818 | ≈162.700 ARS ≈ 102 USD |
| 12 cuotas | 154.493 | 1.853.916 | 406.283 | ≈211.600 ARS ≈ 133 USD |

La tasa (4%/mes) duplica la inflación esperada: las cuotas **no** se licúan, y a igual tasa el plazo más
corto es el más barato. Se eligió **6 cuotas**, que baja el vencimiento del 09/10 de 2.270.372 a
≈1.098.700 ARS. Hay **10 días hábiles para ajustar** (≈ hasta el 09/10): si aparece la liquidez sin tocar
cartera ni el margen de IB, cancelar el plan y pagar el boleto completo.

### 4.2 — El seguro de viajero (320 USD): va con Santander ✅ hecho

**Recomendación.** Con BBVA cae en el resumen que cierra el 01/10 y se apila sobre los 1.430 USD del 09/10.
Con Santander cierra el 22/10 y se paga el **02-03/11**: son 25 días más y, sobre todo, cae en **otro mes**.

No es optimizar centavos: es no juntar el boleto y el seguro en el mismo vencimiento.

### 4.3 — Alquiler del vehículo y varios (1.200 USD): a definir

En Venezuela, en efectivo USD o con tarjeta. Si va con tarjeta argentina hay que contar la percepción
(la `RG 5617 30%` aparece en los dos resúmenes de septiembre). Si va en efectivo, sale de la caja en USD y
no toca ningún resumen — pero tampoco queda registrado en Finanzas salvo que se cargue a mano.

### Decisiones tomadas

| Fecha | Decisión | Motivo |
|---|---|---|
| 2026-09-27 | Seguro Assist Card con Santander Visa, 1 cuota, 453.612 ARS (294 USD) | 20% off pago en 1 cuota (promo vence 28/09); vence 02/11, fuera del resumen BBVA del 09/10 |
| 2026-09-27 | Boleto financiado en 6 cuotas de 276.399 ARS (TEA 60,6%) | Mismo costo por mes que 9/12 cuotas, menor interés total; revisable 10 días hábiles |

---

## 5. Cierre — estimado vs real

Se completa cuando entren los resúmenes que facturan el viaje.

| Rubro | Estimado | Real | Desvío | Por qué |
|---|---|---|---|---|
| | | | | |

**Regla del cierre:** el desvío se explica rubro por rubro. Un total que cuadra tapando un rubro que se fue al
doble contra otro que no se gastó no enseña nada para el próximo viaje.

---

## 6. Cómo refrescar los números del sistema

Los datos de la sección 2 salen de la app y envejecen. Para actualizarlos:

- **Ciclos de tarjeta y cuotas** → tab **Finanzas**, vista **Compromisos** (`get_installments_pending`)
- **Qué resumen falta cargar** → tab Finanzas, vista **Cobertura**
- **Gasto por clase** → el bloque de KPIs del panel

Para recalcular el gasto por mes y clase a mano:

```sql
SELECT SUBSTRING(COALESCE(t.billing_date, t.date), 1, 7) AS mes,
       COALESCE(c.expense_class, '(sin clase)') AS clase,
       ROUND(SUM(t.amount_usdt), 2) AS usd, COUNT(*) AS n
  FROM fin_transactions t
  LEFT JOIN fin_categories c ON c.id = t.category_id
 WHERE c.category_type = 'expense'
   AND COALESCE(t.billing_date, t.date) >= '2026-04-01'
 GROUP BY mes, clase
 ORDER BY mes, clase;
```

---

## 7. Limitaciones conocidas de los datos

Están acá para que ninguna conclusión de esta hoja se apoye en un dato que no existe:

1. **El gasto del ciclo en curso no está en la base.** Los movimientos entran al importar el resumen, y el
   resumen llega después del cierre. Medido hoy: `fin_transactions` con `billing_date` futura = **0 filas**.
   Lo gastado desde el 24/09 con Santander no está en ningún lado hasta que llegue ese resumen. Es
   exactamente el hueco que taparía el módulo de la sección 8.
2. **Septiembre está a medio cargar** (faltan los resúmenes de Santander). Cualquier promedio que lo incluya
   sale bajo.
3. **El saldo del préstamo bancario no está en ninguna tabla `fin_*`** — solo se ven las cuotas ya cobradas.
4. El `Saldo anterior` de las tarjetas Santander se guarda en NULL por una limitación del parser
   (BACKLOG 95). No afecta a este plan: el cierre sí se lee.

---

## 8. Módulo propuesto: "lo que viene"

Idea del 2026-09-27 — **no implementado**. Una vista que responda *cuánto voy a pagar el mes que viene*,
sumando tres cosas que hoy están en tres lugares distintos:

| Componente | Estado hoy |
|---|---|
| Cuotas de planes vigentes | ✅ existe — `get_installments_pending()`, vista Compromisos |
| Próximo cierre y vencimiento por tarjeta | ✅ existe — `fin_card_cycles` |
| Gasto del ciclo en curso (todavía sin resumen) | ❌ **no existe** — es el hueco del punto 7.1 |
| Estimado del viaje | ❌ no existe — hoy es esta hoja |

Falta un solo dato: **el consumo del ciclo abierto**. Dos caminos, y hay que elegir uno:

- **A — carga manual.** Una tabla de gastos comprometidos que el usuario alimenta. Barato, inmediato, dato
  propio, y sirve igual para el estimado del viaje: los dos son "plata comprometida que ningún resumen
  muestra todavía".
- **B — scraping del homebanking.** Exacto y automático, pero `Class_BrowserFCI` está bloqueado desde el
  2026-09-07 y el login de BBVA rebota al login (BACKLOG 94). Depende de arreglar eso primero.

**A no depende de nada y cubre el viaje además del ciclo.** Queda como propuesta hasta que se decida.
