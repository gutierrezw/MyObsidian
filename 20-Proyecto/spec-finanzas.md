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
| 4 | Encontrar el número de libertad financiera y trabajar en función de él: reducir gastos, potenciar inversiones | Gasto anual recurrente ÷ tasa de retiro (4% → ×25, configurable). Progreso = patrimonio invertido ÷ número. Años estimados según la tasa de ahorro. Variante a evaluar: ingreso pasivo ≥ gasto recurrente | `get_totales_inversiones()` ya da el patrimonio invertido en USD de todos los vehículos |
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

### Orden de trabajo

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

**Mapa vigente (31 categorías):**

| Tipo | Clase | Categorías |
|------|-------|-----------|
| expense | fixed | Alquiler, Servicios, Suscripciones, Expensas, Familia, Cuotas de préstamo, Intereses y financiación, Banco e impuestos |
| expense | variable | Supermercado, Restaurantes, Transporte, Salud, Médico, Hogar, Ropa y compras personales, Otros gastos |
| expense | extraordinary | Tecnología, Viajes, Mantenimiento, Cancelación de préstamo |
| income | — | Sueldo, Reintegros y bonificaciones, Intereses ganados, Otros ingresos |
| transfer | — | Pago tarjeta crédito, Entre cuentas propias, Cambio de moneda, Préstamo recibido |
| investment | — | Inversiones, Aportes a IB, Retención USDT |

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
- Debe ganar la regla más específica: hoy #269 y #320 quedan tapadas por otras.
- La aplicación retroactiva solo debe completar filas sin categoría, sin pisar las clasificadas.
- Reglas genéricas para los planes de tarjeta.
- El popup de categorías no muestra ni edita `expense_class`.
- Las cuotas de tarjeta se importan con la fecha de la compra original: CUOTIFICACION 1-06…5-06 viene de cinco
  extractos distintos y las cinco filas quedan el 2026-02-07.
- Re-aplicar las reglas una vez corregidas (escritura en BD, con autorización).

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
