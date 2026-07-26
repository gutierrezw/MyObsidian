---
tipo: ref
modulo: instalacion-hijo
version: 1.0
fecha: 2026-07-25
status: activo
---

# Instalación AppOO — Versión Hijo

Guía paso a paso para instalar la versión hijo.
Requisitos mínimos: MySQL 8.x + ejecutable AppOO + Obsidian.
No requiere IB Gateway, Node/server-api ni Binance testnet.

---

## PASO 1 — Descargar el release

1. Ir a `https://github.com/gutierrezw/MyPython/releases`
2. Descargar el release más reciente (ej: `v10.5.0`)
3. Extraer en una carpeta local, por ejemplo `C:\AppOO\`
4. La estructura debe quedar:
   ```
   C:\AppOO\
     AppOO.exe
     profiles\main.json      ← ya viene configurado como hijo
     tmp\                    ← carpeta vacía (se crea automáticamente)
   ```

---

## PASO 2 — Instalar MySQL

1. Descargar MySQL Community Server 8.0.x desde `https://dev.mysql.com/downloads/`
2. Instalar con usuario `root` y una contraseña segura
3. Crear la base de datos:
   ```sql
   CREATE DATABASE bdinv CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

---

## PASO 3 — Importar estructura de tablas

Desde la carpeta `setup_hijo\` incluida en el release:

```bash
# Tablas vacías (estructura)
mysql -u root -p bdinv < hijo_estructura.sql

# Datos de referencia y estadísticas
mysql -u root -p bdinv < hijo_datos.sql
```

---

## PASO 4 — Crear tablas adicionales

Estas tablas se crearon después del release base y deben aplicarse manualmente:

```bash
mysql -u root -p bdinv < SchemasSQL\incidencias.sql
mysql -u root -p bdinv < SchemasSQL\ib_flex_trades.sql
mysql -u root -p bdinv < SchemasSQL\reportes_historial.sql
mysql -u root -p bdinv < SchemasSQL\market_sentiment.sql
mysql -u root -p bdinv < SchemasSQL\youtube_canales.sql
mysql -u root -p bdinv < SchemasSQL\youtube_candidatos.sql
```

> Todos los scripts usan `CREATE TABLE IF NOT EXISTS` — son idempotentes, se pueden correr de nuevo sin riesgo.

---

## PASO 5 — Crear índices críticos

Verificar si ya existen antes de correr (el dump puede incluirlos). Si no:

```sql
CREATE INDEX idx_hash_id              ON booktrading (hash_id);
CREATE INDEX idx_hash_id              ON oportunidadesbuysell (hash_id);
CREATE INDEX idx_idcuenta             ON trazaplan (idcuenta);
CREATE INDEX idx_symbol               ON market (symbol);
CREATE INDEX idx_cusip                ON market (cusip);
CREATE INDEX idx_cik                  ON funds (cik);
CREATE INDEX idx_fecha_cod            ON diaria_cnv (fecha, codCAFCI);
CREATE INDEX idx_idcuenta_vehiculo    ON performa_inversion (idcuenta, vehiculo);
CREATE INDEX idx_account_symbol       ON order_trader (account, symbol);
```

---

## PASO 6 — Configurar `profiles\main.json`

Ya viene preconfigurado para el hijo. Solo editar:

```json
{
  "db": {
    "host": "localhost",
    "user": "root",
    "password": "TU_CONTRASEÑA",
    "database": "bdinv"
  },
  "tmp_path": "C:\\AppOO\\tmp\\"
}
```

---

## PASO 7 — Configurar credenciales desde la app

Abrir `AppOO.exe` y desde el menú de configuración de cada vehículo:

| Vehículo | Qué configurar |
|----------|---------------|
| Chatbot | Bot Token y Chat ID de Telegram |
| Crypto | API Key y Secret Key de Binance |
| BBVA / Santander | Usuario y clave bancaria (desde menú FCI) |
| ClaudeAPIC | API Key para el Chatbot IA |
| ClaudeAPIP | API Key para Preservation y Agente IA |
| ClaudeAPIS | API Key para Sentimiento (opcional) |
| ClaudeAPIE | API Key para Clasificador ETF (opcional) |

---

## PASO 7b — Crear API Key de Binance

El hijo necesita su propia API Key — cada usuario de Binance tiene las suyas.

### Paso 1 — Crear la API Key en Binance

1. Ir a `https://www.binance.com` → Login → menú de usuario → **API Management**
2. Click **Create API** → elegir tipo **System generated** (HMAC)
3. Darle un nombre descriptivo (ej: `AppOO`)
4. Pasar el 2FA que pida
5. Binance genera dos valores — **guardarlos inmediatamente** (la Secret Key no se vuelve a mostrar):
   - **API Key** (empieza con letras/números, ~64 chars)
   - **Secret Key** (similar longitud)

### Paso 2 — Permisos mínimos necesarios

En la configuración de la API Key activar solo lo necesario:

| Permiso | Activar |
|---------|---------|
| Enable Reading | ✅ Sí |
| Enable Spot & Margin Trading | ✅ Sí (si opera spot) |
| Enable Withdrawals | ❌ No |
| Enable Margin Loans | ❌ No |

> **Restricción IP recomendada:** agregar la IP de la máquina donde corre AppOO — reduce el riesgo si la key queda expuesta.

### Paso 3 — Configurar en la app

En `AppOO.exe` → menú **Configuración → Sesiones** → vehículo `Crypto` → editar:

| Campo | Valor |
|-------|-------|
| `userapi` | API Key de Binance |
| `userpass` | Secret Key de Binance |

> **Nota:** Si usa lending/colateral (Flexible Loan), la API Key también necesita permiso **Enable Flexible Loan**. Agregar si aplica.

---

## PASO 7c — Crear el bot de Telegram

El hijo necesita su propio bot de Telegram — no puede compartir el del padre (cada bot tiene un token único).

### Paso 1 — Crear el bot con BotFather

1. Abrir Telegram y buscar `@BotFather`
2. Enviarle el comando `/newbot`
3. BotFather pide un nombre para el bot (ej: `AppOO Inversiones`)
4. Pide un username (debe terminar en `bot`, ej: `AppOO_MiNombre_bot`)
5. BotFather devuelve el **Token** — guardarlo, se ve así:
   ```
   7412345678:AAFxyz_abcdefghijklmnopqrstuvwxyz123
   ```

### Paso 2 — Obtener el Chat ID

El Chat ID es el identificador numérico del chat donde el bot enviará mensajes.

1. Buscar el bot recién creado en Telegram (por el username que elegiste)
2. Enviarle cualquier mensaje (ej: `/start`)
3. En el browser ir a:
   ```
   https://api.telegram.org/bot<TOKEN>/getUpdates
   ```
   (reemplazar `<TOKEN>` por el token del paso anterior)
4. En el JSON de respuesta buscar `"chat":{"id":XXXXXXX}` — ese número es el **Chat ID**

> **Alternativa rápida:** buscar `@userinfobot` en Telegram y enviarle cualquier mensaje — te responde con tu Chat ID directamente.

### Paso 3 — Configurar en la app

En `AppOO.exe` → menú **Configuración → Sesiones** → buscar el vehículo `Chatbot` → doble click para editar:

| Campo en la app | Valor |
|-----------------|-------|
| `userapi` | Token que dio BotFather |
| `iduser` | Chat ID numérico |

Guardar y reiniciar el módulo Telegram desde la app.

### Servicios Claude — ver inventario completo en PASO 8

---

## PASO 8 — Inventario de servicios Claude (API Keys)

Configurar desde el menú de la app (vehículo `ClaudeAPIC`, `ClaudeAPIP`, etc.). Cada vehículo pide la API Key al abrirse por primera vez. Obtener keys en `console.anthropic.com`.

| Clave en BD | Módulo que la usa | Para qué | ¿El hijo lo necesita? |
|-------------|-------------------|----------|-----------------------|
| `ClaudeAPIC` | Chatbot (`Class_DashBot`) | Chat conversacional con Claude en la app | ✅ Sí |
| `ClaudeAPIP` | Preservation + GainsCapture + Agente IA | Evalúa stops, ganancias y contexto de mercado | ✅ Sí |
| `ClaudeAPIS` | Agente Sentimiento + YouTube Scanner | Clasificación de noticias y análisis de sentimiento | ⚠️ Opcional (requiere Screener activo) |
| `ClaudeAPIE` | Agente ClasificadorETF | Clasifica fondos ETF con IA | ⚠️ Opcional |
| `ClaudeAPIA` | ApiCostTracker (solo lectura de costos) | Monitoreo de consumo API (solo admin) | ❌ No aplica |

> **Nota:** El hijo puede compartir la misma API key en varios vehículos. Las keys separadas sirven para monitorear el consumo por módulo en `console.anthropic.com`.

> **Modelo por defecto:** `claude-haiku-4-5-20251001` para Sentimiento/ETF/GainsCapture; `claude-opus-4-8` para Preservation y Chatbot (configurable en cada vehículo).

---

## Tablas — Qué llega con datos y qué empieza vacío

### Con datos de referencia / estadísticas (importadas desde padre)

| Tabla | Contenido |
|-------|-----------|
| `sesion` | Configuración de vehículos — **editar credenciales post-importación** |
| `estrategia` | Estrategias de inversión definidas |
| `plan` | Parámetros del plan de inversión |
| `variablesplan` | Variables del plan |
| `modelos_ia` | Modelos BUY/SELL entrenados (se reusan del padre) |
| `diaria_cnv` | Histórico de precios FCI (CNV) — crítico para TWR |
| `split` | Histórico de splits bursátiles |
| `sys_objeto` | Catálogo de objetos del sistema |
| `fin_banks` | Credenciales bancos — **editar con las del hijo** |
| `fin_categories` | Categorías de gastos finanzas personales |
| `fin_import_rules` | Reglas de importación finanzas |

### Vacías al instalar (el hijo las puebla con sus propios datos)

| Tabla | Se puebla con... |
|-------|-----------------|
| `booktrading` | Operaciones del hijo |
| `inversion` | Posiciones actuales del hijo |
| `oportunidadesbuysell` | Señales generadas por el screener |
| `order_trader` | Órdenes enviadas al broker |
| `diaria_performance` | Performance diaria (se construye con el tiempo) |
| `performa_inversion` | Performance por vehículo |
| `otros_activos` | FCI, ARS y otros activos del hijo |
| `extractos` | Extractos FCI mensuales |
| `trazaplan` | Trazabilidad del plan |
| `fin_accounts` | Cuentas finanzas personales del hijo |
| `fin_exchange_rates` | Tipos de cambio (auto) |
| `fin_statement_imports` | Importaciones de extractos |
| `fin_transactions` | Transacciones finanzas personales |
| `market` | Universo de símbolos (screener) |
| `funds` / `fund_filings` / `fund_holdings` | Pipeline 13F (solo padre) |
| `incidencias` | Incidencias detectadas por agentes |
| `market_sentiment` / `market_sentiment_analysis` | Sentimiento (agente automático) |
| `youtube_canales` / `youtube_candidatos` | Scanner YouTube |
| `reportes_historial` | Report Center |
| `ib_flex_trades` | Trades IB Flex (no aplica si no usa IB) |

---

## Obsidian — Documentación

### Qué llevar

| Carpeta / Archivo | Llevar | Motivo |
|-------------------|--------|--------|
| `20-Proyecto/spec-*.md` | ✅ Sí | Specs de módulos activos |
| `20-Proyecto/design-*.md` | ✅ Sí | Diseño de features |
| `20-Proyecto/ref-instalacion-hijo.md` | ✅ Sí | Este documento |
| `10-Memoria/ref-*.md` | ✅ Solo `ref-*` | Referencias técnicas |
| `10-Memoria/state-botcrypto.md` | ✅ Sí | Estado BotCrypto activo en hijo |
| `10-Memoria/MEMORY.md` | ❌ No | Es memoria del padre — el hijo crea la suya |
| `10-Memoria/feedback-*.md` | ❌ No | Contexto del padre |
| `30-Gestion/` | ❌ No | Backlog y sesiones del padre |

### Cómo mantener la doc actualizada

1. El padre publica nuevos specs/designs en `20-Proyecto/`
2. El hijo puede clonar el mismo vault Obsidian (Git) y hacer `pull` para recibir actualizaciones
3. La memoria del hijo (`10-Memoria/MEMORY.md`) es **independiente** — Claude en el hijo construye la suya
4. Si hay cambios de schema (nuevas tablas) → el padre publica los `.sql` en `SchemasSQL/` del repo

---

## PASO 9 — TradingView + Panel AppOO

### 9.1 Crear cuenta TradingView

Registrarse en `https://www.tradingview.com` — el plan gratuito es suficiente.

### 9.2 Agregar indicadores del autor

En TradingView → abrir un gráfico → **Indicators** → buscar autor: **GutierrezW**

Agregar y marcar como favoritos:
| Indicador | Descripción |
|-----------|-------------|
| `EMA/MACD cross {dual 4 EMA (V2.0)}` | Medias móviles en el gráfico |
| `RSI Cross + VIX + Volume (v5.1)` | Panel RSI + VIX + volumen debajo del gráfico |

### 9.3 Instalar el panel Tampermonkey (tv_panel.js)

El panel conecta TradingView con AppOO para ver la cartera directamente en el gráfico.

**Paso 1 — Instalar Tampermonkey** (extensión del browser):
- Chrome: `https://chrome.google.com/webstore/detail/tampermonkey`
- Firefox: `https://addons.mozilla.org/firefox/addon/tampermonkey`

**Paso 2 — Instalar el script:**
1. Abrir Tampermonkey → Dashboard → icono `+` (nuevo script)
2. Borrar todo el contenido por defecto
3. Pegar el contenido de `setup_hijo\tv_panel.js`
4. Guardar con Ctrl+S

**Paso 3 — Verificar:**
- Abrir TradingView en el browser
- Debe aparecer un panel flotante con datos de la cartera
- Si no aparece → verificar que Tampermonkey esté activado

> **Requisito:** AppOO debe estar corriendo en la misma máquina. El panel se comunica con la app a través de `localhost:5050`.

---

## PASO 10 — Verificar que la app arranca

1. Ejecutar `AppOO.exe`
2. Verificar en la pantalla de Sistema → APIs que aparezcan al menos: **MySQL ✅**, **Telegram ✅**, **Binance ✅**
3. Si alguna falla → revisar credenciales en `sesion` BD

---

## Actualización del ejecutable

Cuando el padre libera un nuevo release:

1. Descargar el nuevo release de GitHub
2. Reemplazar `AppOO.exe` — **no tocar** `profiles\main.json` ni `tmp\`
3. Si el release menciona nuevas tablas en `SchemasSQL/` → aplicarlas (son idempotentes)
4. Si el release menciona cambios en `sesion` → verificar que los nuevos vehículos existan en BD

---

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-25 | Documento inicial — paso a paso + inventario servicios Claude |
