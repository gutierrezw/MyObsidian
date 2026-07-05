---
tipo: ref
modulo: cowork-tools
version: 1.1
fecha: 2026-07-03
status: activo
---

# Co-work — Tools disponibles para Claude

> Referencia rápida de lo que puede hacer Claude como co-work.
> Arquitectura y decisiones de diseño → [[design-api-server]]

---

## Disponibilidad por tipo de sesión

| Sesión | MCP appoo disponible | Cómo |
|--------|---------------------|------|
| **Claude Code CLI** | ✅ Ya funciona | `~/.claude.json` via `claude mcp add` |
| **Cowork / Chat / Mobile** | ✅ OAuth implementado | `api-main.wildaga.com/mcp` — registrar en claude.ai Settings → Connectors |

---

## Configuración MCP — Claude Code CLI (activo)

Comando usado para registrar (una sola vez):

```bash
claude mcp add --transport http --scope user appoo http://localhost:8050/mcp \
  --header "X-API-Key: <api_key del config>"
```

Escribe en `~/.claude.json`. Verificar con: `claude mcp list` → debe mostrar `appoo: ✔ Connected`.

**Prerequisito:** `pm2 start server-api` corriendo en puerto 8050.

---

## Configuración MCP — Cowork/Chat (OAuth activo)

Túnel: `https://api-main.wildaga.com` → puerto 8050. OAuth 2.0 + PKCE implementado.

**Registrar conector en claude.ai:**
1. Settings → Connectors → Add MCP
2. MCP Server URL: `https://api-main.wildaga.com/mcp`
3. OAuth Authorization URL: `https://api-main.wildaga.com/oauth/authorize`
4. OAuth Token URL: `https://api-main.wildaga.com/oauth/token`
5. Client ID: `claude-ai`
6. Client Secret: en `oauth_clients.json`
7. Hacer click en "Connect" → se abre popup → "Permitir" → token guardado en `oauth_tokens.json`

**Discovery automático:** `https://api-main.wildaga.com/.well-known/oauth-authorization-server`

---

## Tools disponibles

### `query_portfolio`
Posiciones activas. Incluye qty, avg cost, last price, sector, inst_score, consenso tag.

| Param | Tipo | Descripción |
|-------|------|-------------|
| `account` | string (opcional) | Ej: `U4214563`. Sin valor → todas las cuentas. |

---

### `get_consenso`
Score de consenso de un símbolo: votos por categoría + tag final.

| Param | Tipo | Descripción |
|-------|------|-------------|
| `symbol` | string | Ej: `AAPL` |

Retorna: `consenso_tag`, `consenso_score`, votos (net/opt/flujo/ana/val/cob), `inst_score`, `fh_count`, `inst_ownership_pct`.

---

### `get_market_data`
Datos de mercado de un símbolo: precio, volumen, márgenes, 13F score.

| Param | Tipo | Descripción |
|-------|------|-------------|
| `symbol` | string | Ej: `MSFT` |
| `fields` | string[] (opcional) | Campos específicos. Sin valor → resumen estándar. |

---

### `get_booktrading`
Historial de operaciones.

| Param | Tipo | Descripción |
|-------|------|-------------|
| `account` | string (opcional) | Filtrar por cuenta |
| `symbol` | string (opcional) | Filtrar por símbolo |
| `limit` | int (opcional) | Máx registros. Default: 50, máx: 200. |

---

### `execute_order`
Ejecuta una orden BUY/SELL via AppOO. **Por defecto simula** — requiere `confirm=true` para ejecutar.

| Param | Tipo | Descripción |
|-------|------|-------------|
| `symbol` | string | Ej: `NVDA` |
| `side` | `BUY` \| `SELL` | Dirección |
| `price` | number | Precio límite en USD |
| `qty` | number (opcional) | Lotes. Alternativa a `importe`. |
| `importe` | number (opcional) | Monto total USD. Alternativa a `qty`. |
| `vehiculo` | `Stock` \| `Crypto` | Default: `Stock` |
| `account` | string (opcional) | Cuenta. Default: la activa en app. |
| `confirm` | bool | `false` = simulación (default). `true` = ejecuta. |

---

### `get_agent_status`
Estado actual del servidor y la app.

Sin parámetros. Retorna: símbolo activo, balance USDT libre, última conexión TradingView, lista de símbolos con posición abierta.

---

## Qué NO está disponible (gaps)

| Dato | Dónde existe | Disponible vía MCP |
|------|--------------|--------------------|
| Precio live del símbolo activo | `/tv/price` (TV state) | ❌ |
| Contexto completo que AppOO envía a TV | `/tv/contexto` | ❌ |
| Lista completa de símbolos registrados | `/tv/symbols` | Parcial via `get_agent_status` |
| Extractos YTD | `/db/extractos` | ❌ |
| Diagnóstico DB (tablas, índices) | `/db/diagnostics` | ❌ |

Si alguno de estos gaps se vuelve necesario → agregar tool en `routes/mcp.js` (trivial).

---

## Audit log

Cada tool call queda registrado en `server-api/logs/mcp_audit.jsonl` con timestamp, tool, args y resultado.

## Historial

| Versión | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-07-03 | Creación tras implementar MCP Fase 3 |
| 1.1 | 2026-07-03 | Corrección config: `claude mcp add` (no settings.json). Tabla disponibilidad por sesión. Pasos Tunnel pendiente. |
| 1.2 | 2026-07-05 | Tunnel activo: `api-main.wildaga.com`. Bloqueado por OAuth — claude.ai Connectors no acepta API key. |
| 1.3 | 2026-07-05 | OAuth 2.0 + PKCE implementado en `routes/oauth.js`. Cowork/Chat ya puede conectar. |
