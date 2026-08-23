# Agente de Control de LTV — Binance Crypto

## Objetivo

Mantener el LTV de cada préstamo flexible dentro de un rango parametrizado
mediante ajuste de colateral, sin modificar la deuda.

---

## Definición

```
LTV = Deuda (USDT) / Colateral (USD)
Cobertura = Colateral (USD) / Deuda (USDT) = 1 / LTV
```

---

## Parámetros (columna `parameters` en sesion Crypto — BD)

```json
{
  "preservation": {
    "roi_minimo": 0.18,
    "proteccion_base": 0.40,
    "correccion_pct": 0.12,
    "atr_mult": 2.5,
    "revisiones_dia": 2
  },
  "ltv": {
    "target": 0.28,
    "tolerance": 0.05,
    "critical": 0.65,
    "rebalance_step": 0.25
  },
  "loan": {
    "max_deuda_pct": 0.09,
    "delta_minimo": 1.0,
    "repay_pct_venta": 0.05
  }
}
```

> **`sesion.parameters` es BLOB, no JSON.** `JSON_SET` directo falla con
> *Error 3144: Cannot create a JSON value from a string with CHARACTER SET 'binary'*.
> Hay que convertir de ida y volver, siempre sobre el sub-objeto para no reescribir el resto
> del blob (que tiene credenciales):
>
> ```sql
> UPDATE sesion
>    SET parameters = CAST(
>          JSON_SET(CONVERT(parameters USING utf8mb4), '$.loan.repay_pct_venta', 0.05)
>          AS BINARY)
>  WHERE vehiculo = 'Crypto';
> ```
>
> Para leer: `SELECT JSON_EXTRACT(CONVERT(parameters USING utf8mb4), '$.loan') ...`
> En Python es el `raw.decode("utf-8")` de `ServiciosCrypto._loan_config()`.

> `preservation`, `ltv` y `loan` comparten el mismo JSON en BD.
> Todos los agentes leen desde el cache compartido (`_params_cache`) sin releer BD.

### Restricción de deuda máxima (`loan.max_deuda_pct`)

```
capital_earn  = valor USD total de todos los activos en Simple Earn (flexible)
max_deuda     = capital_earn * max_deuda_pct
apalancamiento = total_debt / capital_earn
```

La billetera Earn completa es la garantía real del sistema — no solo los activos
en colateral activo. Antes de cualquier préstamo nuevo:

```
si total_debt_actual + requested > capital_earn * max_deuda_pct → rechazar / reducir
```

**Ejemplo:** capital_earn=$2,000 → max_deuda=$180 USDT. Deuda actual=$147 → disponible=$33.
API: `GET /sapi/v1/simple-earn/flexible/position`
> Ambos agentes los leen desde un cache compartido (`_params_cache`) sin releer BD.

### Repago automático con la venta (`loan.repay_pct_venta`) — 2026-08-23

Cada venta Crypto que se ejecuta paga deuda con un porcentaje de **el importe bruto de la
operación** (no de la ganancia). Alcanza a toda la cuenta Crypto: manual, agentes y Telegram.
`repay_pct_venta = 0` lo desactiva; el valor en producción arrancó en `0.05` para ir buscando el
mejor número.

**Cuánto se paga:**

```
monto = min(importe_venta * repay_pct_venta, deuda_total, usdt_libre_spot)
```

Nunca más que la deuda viva, nunca más que el USDT libre. Si no hay deuda, no paga. Si el monto
queda bajo `loan.delta_minimo` ($1), **no paga y lo loguea** — no acumula para la próxima venta;
si los logs muestran que se saltea seguido, se convierte en acumulador.

**Dónde engancha:** en el fill, no en el envío de la orden. Una LIMIT SELL no es plata hasta que
se ejecuta. El hook es `procesa_execution_report_crypto()` con `X == "FILLED"` y `S == "SELL"`,
usando el campo `Z` del `executionReport` (quote acumulado = los USDT que entraron). `FILLED` es
terminal, así que dispara una sola vez aunque la orden se haya llenado en varios fills.

El repago corre en un hilo daemon aparte: son tres llamadas REST y no pueden frenar el stream del
websocket.

**Cómo se reparte:** con `loan_repay_distribuir()`, el mismo criterio de nivelación del botón
**Pagar** de Análisis Crypto — cada préstamo recibe en proporción a cuánto excede la deuda media
que quedaría tras el pago, o sea paga más donde más deuda hay. La lógica vivía anidada dentro de
la sección Tk (`_ejecutar_pago`), inalcanzable desde un agente; se subió a `ServiciosCrypto` y
ahora la UI y el repago automático entran por la misma puerta. Un solo criterio de reparto en
todo el sistema.

### Selección del target

| Target | Cobertura operativa | Drop máximo tolerable (sin agente) |
|--------|--------------------|------------------------------------|
| 0.28   | 3.4x–3.8x          | **-67%** (recomendado — app on-premise) |
| 0.33   | 2.9x–3.2x          | **-61%** (recomendado — app en nube)    |
| 0.37   | 2.6x–2.8x          | **-57%** (riesgoso para altcoins)       |

**Target actual: 0.28** — conservador mientras la app no esté en la nube.
Migrar a 0.33 cuando haya disponibilidad 24/7 (cambio de un parámetro en BD).

---

## Estados y acciones

| Estado  | Condición                                | Acción     |
|---------|------------------------------------------|------------|
| NORMAL  | `target*(1-tol) ≤ LTV ≤ target*(1+tol)` | Sin ajuste |
| ALTO    | `LTV > target*(1+tol)`                   | ADDITIONAL |
| BAJO    | `LTV < target*(1-tol)`                   | REDUCED    |
| CRITICO | `LTV ≥ critical`                         | ADDITIONAL |

> **BAJO → REDUCED**: retirar exceso de colateral para subir LTV hacia target.
> Agregar colateral cuando BAJO crea un loop divergente (LTV baja más cada ciclo).

---

## Fórmulas

```
rango_normal  = [target*(1-tolerance), target*(1+tolerance)]
colateral_obj = deuda / target
delta_usd     = |colateral_actual - colateral_obj| * rebalance_step
precio_col    = collateral_usd / collateral_amount
ajuste_coin   = delta_usd / precio_col
```

`rebalance_step = 0.25` — cubre 25% del gap por iteración.
Convergencia al rango en ~4–8 iteraciones (20–40 min a ciclos de 5 min).

---

## Reglas

- No tocar la deuda (solo ajuste de colateral)
- Si `currentLTV == 0`: skip (sin préstamo activo)
- Mantener buffer vs margin call (85%) y liquidación (91%)
- El agente solo actúa si la sesión Crypto está activa

---

## Arquitectura

```
Class_DashBot.py
  └── Agente_LtvControl()          ← coordinador puro (@wait_rate(300))

DashMain.py
  └── procesa_execution_report_crypto()
        └── repay_deuda_venta()    ← coordinador puro, hilo daemon

Class_ServiciosCrypto.py
  └── ServiciosCrypto
        ├── ltv_check_and_adjust()
        ├── repay_venta()          ← cuánto pagar (% del importe, topes)
        ├── loan_repay_distribuir()← cómo repartirlo — lo usa tambien el boton Pagar
        ├── loan_ongoing()         ← prestamos vivos normalizados
        └── loan_balance()         ← PENDIENTE (ver sección abajo)

Class_Analisis.py
  └── boton Pagar → ServiciosCrypto().loan_repay_distribuir()

Class_ApiBinnace.py
  └── BinanceSpot (API wrappers puros)
        ├── get_flexible_loan_ongoing_orders()
        ├── get_flexible_adjust_ltv()
        ├── get_flexible_loan_repay()
        └── get_flexible_loan_borrow()   ← PENDIENTE AGREGAR
```

---

## Servicio pendiente — loan_distribute

### Objetivo
Distribuidor de liquidez inteligente: el usuario solicita X USDT y el sistema
los reparte entre todos los activos colaterales de forma que todos terminen
con el mismo LTV, sin forzar exactitud (parámetro `delta` de tolerancia).

### Lógica
```
total_debt_new  = Σ deuda_actual_i + requested
total_col       = Σ collateral_usd_i
ltv_nuevo       = total_debt_new / total_col

para cada activo i:
    deuda_objetivo_i = collateral_usd_i * ltv_nuevo
    borrow_i         = deuda_objetivo_i - deuda_actual_i
    si borrow_i < delta  → skip (evita micro-préstamos)
    si borrow_i > 0      → get_flexible_loan_borrow(loanCoin, collateralCoin, borrow_i)
```

### Ejemplo — solicitud de 100 USDT
```
total_debt_new = 147.35 + 100 = 247.35 USDT
total_col      = 543 USD
ltv_nuevo      = 247.35 / 543 = 45.5%
```

| Activo | Col. USD | LTV hoy | Pedir  | LTV final |
|--------|----------|---------|--------|-----------|
| ADA    | 135.98   | 24.8%   | +28.19 | 45.5%     |
| ICP    | 131.30   | 29.2%   | +21.36 | 45.5%     |
| FIL    | 122.69   | 27.2%   | +22.47 | 45.5%     |
| VET    | 91.39    | 27.2%   | +16.69 | 45.5%     |
| SOL    | 61.66    | 27.7%   | +11.01 | 45.5%     |
| **Total** |       |         | **≈100 USDT** |    |

Todos terminan con el mismo LTV — carga distribuida proporcionalmente
al colateral de cada activo.

### Parámetro delta
Monto mínimo para ejecutar un préstamo individual. Si `borrow_i < delta`
se omite ese activo y el resto absorbe la diferencia.
Sugerido: `delta = 1.0 USDT`.

### API requerida
- `borrow_i > delta` → `get_flexible_loan_borrow(loanCoin, collateralCoin, amount)`

### Estado
Pendiente de implementación — requiere crear `Class_ServiciosCrypto.py`
y agregar `get_flexible_loan_borrow()` en `Class_ApiBinnace.py`.

---

## Historial de decisiones

| Fecha      | Decisión |
|------------|----------|
| 2026-03-24 | Target=0.28 conservador mientras app es on-premise; migrar a 0.33 en nube |
| 2026-03-24 | BAJO→REDUCED — agregar colateral en BAJO crea loop divergente |
| 2026-03-24 | Skip LTV=0 — sin préstamo activo, nada que gestionar |
| 2026-03-24 | tolerance relativa: `target*(1±tol)` — rango 26.6%–29.4% para target=0.28 |
| 2026-03-24 | `preservation` y `ltv` comparten cache `_params_cache` en ClassAgenteIA |
| 2026-03-24 | Agente activo en producción — ejecuta ajustes reales cada 5 min |
| 2026-08-23 | Repago sobre el **importe** de la venta, no sobre la ganancia — y solo si hay deuda |
| 2026-08-23 | El hook es el fill (`FILLED`/`Z`), no el envío de la orden: antes del fill no hay plata |
| 2026-08-23 | Reparto = el de `_ejecutar_pago` (nivelación), subido a `ServiciosCrypto`. Se descartó "mayor LTV primero" para no tener dos criterios |
| 2026-08-23 | Bajo `delta_minimo` se saltea y se loguea — sin acumulador, no se agrega estado nuevo sin evidencia |
