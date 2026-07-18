---
tipo: spec
modulo: FCI/BrowserFCI
version: 0.1-borrador
fecha: 2026-07-18
status: borrador
---

# Spec — BrowserFCI: Ejecución de Operaciones FCI (Santander + BBVA)

## Visión

Primera automatización de ejecución real de operaciones FCI desde la app.
El canal Telegram actúa como aprobador: propuesta → usuario aprueba → BrowserFCI ejecuta.

Mismo modelo de captura por escalones (25/33/100%) ya usado en Stock y Crypto,
ahora aplicado a FCI. Los FCI de Renta Fija funcionan como "cash remunerado" —
el destino de rotación cuando se captura ganancia en Renta Variable.

---

## Flujo general

```
Agente_ClaudeIA detecta oportunidad SELL en FCI RV
        ↓
Propuesta Telegram (fondo origen, monto, fondo destino sugerido)
        ↓
Usuario aprueba (/ok)
        ↓
BrowserFCI ejecuta la operación en el banco
        ↓
Resultado notificado por Telegram + registrado en booktrading
```

---

## Lógica de candidato destino

`select_fci_rf_candidato(cuenta)` ya implementado en `DiariaCNV` (Modulos_Mysql.py):
- Excluye fondos "Acciones" y "Renta Variable"
- Ordena por `variacion30dias ASC` → el más deprimido primero
- `LIMIT 1` → devuelve el candidato

### Santander — candidatos RF posibles
- Supergestión MIX VI
- Super Ahorro Plus (Pesos)

### BBVA — candidatos RF posibles
- FBA Horizonte (ya es la banda de referencia / piso)
- TBD — confirmar con guide & capture

---

## Dos lógicas de señal (pendiente separar en código)

| Señal | Tipo | Acción |
|-------|------|--------|
| RV con ganancia | Captura | Vender RV → rotar a RF más deprimida |
| RF deprimida vs banda | Acumulación | Vender RF menos deprimida → comprar RV en el piso |

Hoy ambas se mezclan en `oport_sell`. Hay que diferenciarlas en el prompt Claude
y en el mensaje Telegram para que la decisión sea correcta.

---

## Santander — traspaso directo

### Ventaja
Una sola operación dentro del banco. El banco mueve internamente entre fondos.
No hay brecha de tiempo entre venta y compra.

### Navegación (PENDIENTE guide & capture)
- [ ] URL sección traspasos
- [ ] Pasos desde dashboard hasta formulario
- [ ] Campos: fondo origen, fondo destino, monto o cuotapartes
- [ ] Botones de confirmación (nombres exactos)
- [ ] Pantalla de confirmación / comprobante

### Código a agregar en Class_BrowserFCI.py
```python
async def _sant_ejecutar_traspaso(self, page, fondo_origen, fondo_destino, monto):
    # TODO: completar con guide & capture
    pass

def ejecutar_traspaso_santander(self, fondo_origen, fondo_destino, monto) -> dict | None:
    """Ejecuta traspaso entre fondos Santander vía Playwright."""
    if self._check_blocked():
        return None
    creds = FinanceScreen().get_bank_credentials("Santander")
    if not creds:
        _logger.error("ejecutar_traspaso_santander: credenciales no configuradas")
        return None
    profile_dir = os.path.join(os.environ.get("APPOO_TMP", "tmp"), "santander_profile")
    return asyncio.run(
        self._sant_async_traspaso(profile_dir, creds["login_user"], creds["login_pass"],
                                   fondo_origen, fondo_destino, monto)
    )
```

---

## BBVA — venta + compra separadas

### Diferencia con Santander
Dos operaciones distintas. Posible brecha de tiempo entre rescate y suscripción.
Riesgo: el dinero queda sin invertir entre operaciones.

### Navegación (PENDIENTE guide & capture)
- [ ] URL sección rescate
- [ ] Pasos y campos para rescate (fondo, monto/cuotapartes)
- [ ] Confirmación rescate
- [ ] URL sección suscripción
- [ ] Pasos y campos para suscripción (fondo destino, monto)
- [ ] Confirmación suscripción

### Código a agregar en Class_BrowserFCI.py
```python
async def _bbva_ejecutar_venta(self, page, fondo, monto):
    # TODO: completar con guide & capture
    pass

async def _bbva_ejecutar_compra(self, page, fondo, monto):
    # TODO: completar con guide & capture
    pass

def ejecutar_venta_bbva(self, fondo, monto) -> dict | None:
    pass

def ejecutar_compra_bbva(self, fondo, monto) -> dict | None:
    pass
```

---

## Integración Telegram

Modelo similar a GainsCapture / propuesta supervisada:

```
🔄 *Propuesta FCI — Santander*
Origen: Superfondo Renta Variable
Destino: Super Ahorro Plus
Importe: $45,000 ARS (ganancia: $3,200 | ROI: +7.1%)

/fci_ok_SANT | /fci_no_SANT
```

Callback en `Class_DashBot.py`:
- `fci_ejecutar_sant|{params}` → `BrowserFCI().ejecutar_traspaso_santander(...)`
- `fci_ejecutar_bbva|{params}` → `BrowserFCI().ejecutar_venta_bbva(...)` + esperar + `ejecutar_compra_bbva(...)`

---

## Arquitectura — principios a respetar

- Todo en `Class_BrowserFCI.py` — no crear módulos nuevos
- Reutilizar perfiles Chrome persistentes ya configurados
- Reutilizar `_check_blocked` / `_set_blocked` / credenciales
- Un solo punto de acceso por banco — no duplicar login
- Resultado de ejecución → log + `system_alerts` Telegram + insertar en booktrading

---

## Pendientes antes de implementar

- [ ] Guide & capture Santander (traspaso)
- [ ] Guide & capture BBVA (rescate + suscripción)
- [ ] Definir separación señal RV-captura vs RF-acumulación en `oport_sell`
- [ ] Definir formato mensaje Telegram propuesta FCI
- [ ] Definir cómo registrar el traspaso en booktrading (1 fila o 2)

---

## Historial
- v0.1 — 2026-07-18: borrador inicial, pendiente guide & capture
