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

## PASO 7 — Configurar credenciales en BD

### Binance (vehículo Crypto)
```sql
UPDATE sesion SET login_user='API_KEY', login_pass='SECRET_KEY'
WHERE vehiculo='Crypto';
```

### FCI Bancos
```sql
UPDATE fin_banks SET login_user='USUARIO', login_pass='CLAVE'
WHERE bank_name='BBVA';

UPDATE fin_banks SET login_user='USUARIO', login_pass='CLAVE'
WHERE bank_name='Santander';
```

### Telegram (notificaciones)
```sql
UPDATE sesion SET login_user='BOT_TOKEN', login_pass='CHAT_ID'
WHERE vehiculo='Telegram';
```

### Servicios Claude — ver inventario completo en PASO 8

---

## PASO 8 — Inventario de servicios Claude (API Keys)

Todas las keys se configuran en la tabla `sesion` como un vehículo:

```sql
INSERT INTO sesion (vehiculo, login_user, login_pass)
VALUES ('ClaudeAPIP', 'sk-ant-...', '');
-- repetir para cada servicio
```

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

## PASO 9 — Verificar que la app arranca

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
