---
tipo: design
modulo: api-server
version: 1.0
fecha: 2026-06-28
status: planificado
---

# AppOO API Server — Diseño por Fases

## Objetivo

Servidor Node.js standalone que actúa como capa unificada de acceso externo a la plataforma AppOO: base de datos, estado del portfolio, órdenes, y herramientas MCP para el agente autónomo. Corre independientemente de la app Tkinter.

---

## Arquitectura Final (visión completa)

```
Tu PC
├── MySQL 3306              ← nunca expuesto directamente
├── AppOO Tkinter           ← sigue siendo local
│     └── POST /internal/*  → push de datos al servidor Node
├── IB Gateway 5501         ← siempre local (requisito IB)
└── appoo-api (Node.js)     ← nuevo, standalone, siempre activo
      ├── /db/*             → MySQL proxy autenticado
      ├── /tv/*             → TradingView bridge (reemplaza BrowserBridge)
      ├── /internal/*       → endpoints privados app→servidor
      └── /mcp              → MCP server para el autónomo
            Cloudflare Tunnel / ngrok
                    ↓
         Co-working / Autónomo / MCPs
```

**PM2** gestiona el proceso: auto-start con Windows, auto-restart si cae.

**Panel en System tab** de AppOO: estado, uptime, restart/stop sin tocar consola.

---

## Fase 1 — MySQL API + Co-working (PRÓXIMA)

**Objetivo**: Co-working puede consultar y escribir en DB desde fuera.

### Stack
- Runtime: Node.js
- Framework: Express
- DB: `mysql2` (mismas credenciales que AppOO vía `config.json`)
- Proceso: PM2
- Túnel: Cloudflare Tunnel (dominio fijo, gratuito) o ngrok

### Estructura de archivos
```
server-api/
├── server.js           ← Express + rutas
├── db.js               ← pool de conexiones MySQL
├── auth.js             ← validación API key
├── routes/
│   ├── query.js        ← endpoints de lectura
│   └── execute.js      ← endpoints de escritura (restringidos)
├── config.json         ← API key + DB credentials (.gitignore)
├── package.json
└── ecosystem.config.js ← configuración PM2
```

### Endpoints Fase 1

| Método | Path | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/health` | público | Estado del servidor |
| GET | `/db/portfolio` | API key | Posiciones activas consolidadas |
| GET | `/db/market?symbol=X` | API key | Datos de mercado de un símbolo |
| GET | `/db/consenso?symbol=X` | API key | Score consenso del símbolo |
| GET | `/db/booktrading?account=X&symbol=Y` | API key | Historial operaciones |
| GET | `/db/extractos?account=X` | API key | Extractos YTD |
| POST | `/db/query` | API key | SELECT libre sobre tablas permitidas |

**Tablas permitidas en /db/query**: `market`, `inversion`, `booktrading`, `fund_holdings`, `diaria_cnv`, `market_sentiment`, `extractos`

### Seguridad Fase 1
- API key en header `X-API-Key`
- Rate limiting: 60 req/min
- Solo GET + /db/query restringido a SELECT
- No exposición de tablas de credenciales/sesiones

### PM2 + Control en AppOO
```bash
# Setup único
npm install -g pm2
cd api-server && npm install
pm2 start ecosystem.config.js
pm2 startup && pm2 save
```

Panel en System tab: indicador ●Online/●Offline + botones Restart/Stop + últimas 10 líneas de log.

---

## Fase 2 — Migración BrowserBridge (FUTURO)

**Objetivo**: Node reemplaza el servidor Python interno (localhost:5050). AppOO empuja datos via HTTP en lugar de compartir memoria.

### Cambio en AppOO
Cada lugar que hoy escribe en `_tv_data` / `_tv_prices` pasa a hacer:
```python
requests.post("http://localhost:NODE_PORT/internal/update", json={...})
```

### Endpoints nuevos en Fase 2

| Método | Path | Descripción |
|--------|------|-------------|
| POST | `/internal/update` | App empuja precios/posiciones al servidor |
| POST | `/internal/order-result` | App notifica resultado de una orden |
| GET | `/tv/position?symbol=X` | Tampermonkey lee posición (reemplaza :5050/position) |
| GET | `/tv/current` | Símbolo activo actual |
| GET | `/tv/price?symbol=X` | Precio live |
| GET | `/tv/ping` | Heartbeat Tampermonkey |
| POST | `/tv/order` | Orden desde TradingView |
| POST | `/tv/current` | Cambia símbolo activo en la app |

**`Class_BrowserBridge.py` queda deprecado** — se elimina en esta fase.

---

## Fase 3 — MCP Server (FUTURO)

**Objetivo**: El mismo servidor expone herramientas MCP que el autónomo usa directamente via protocolo MCP (HTTP/SSE).

Node.js es el runtime nativo del SDK oficial de MCP de Anthropic — integración natural.

### Herramientas MCP planificadas

| Tool | Descripción |
|------|-------------|
| `query_portfolio` | Lee posiciones activas consolidadas |
| `get_consenso` | Score y votos de un símbolo |
| `get_market_data` | Datos mercado + 13F score |
| `execute_order` | Ejecuta orden via IB/Binance (requiere confirmación) |
| `get_booktrading` | Historial de operaciones |
| `get_agent_status` | Estado de los agentes programados |

### Seguridad Fase 3
- MCP sobre HTTP con API key
- `execute_order` requiere doble confirmación (flag en config)
- Audit log completo de cada tool call

---

## Decisiones de arquitectura

| Decisión | Elección | Razón |
|----------|----------|-------|
| Runtime | Node.js | SDK MCP nativo, Express mínimo, fácil en Windows |
| Framework | Express | 30 líneas para 5 endpoints, zero overhead |
| DB client | mysql2 | Pool de conexiones, Promise API |
| Proceso | PM2 | Auto-restart, auto-start Windows, logs integrados |
| Túnel | Cloudflare Tunnel | Dominio fijo gratuito, más estable que ngrok free |
| Auth | API key (header) | Suficiente para cliente único (autónomo) |
| Puerto | 8050 | No conflicta con 5050 (BrowserBridge actual) |

---

## Lo que NO cambia

- MySQL sigue en localhost — nunca expuesto directo
- IB Gateway sigue local — requisito de Interactive Brokers
- AppOO Tkinter sigue siendo app de escritorio
- Puerto 5050 sigue activo en Fase 1 (BrowserBridge Python intacto)

---

## Historial
- 2026-06-28 v1.0 — diseño inicial 3 fases, definición de arquitectura y stack
