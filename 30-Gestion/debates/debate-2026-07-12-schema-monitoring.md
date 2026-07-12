---
tipo: debate
estado: Abierto — esperando perspectiva Desktop
---

# Debate — Tools MCP para monitoreo y corrección de schema MySQL

**Fecha:** 2026-07-12
**Propuesto por:** usuario (sesión VS Code)
**Estado:** 🟡 Abierto — Code escribió, falta Desktop

---

## Tema / Pregunta

> Tenemos un MCP server propio (`MyNode/server-api/routes/mcp.js`) expuesto vía Cloudflare Tunnel con OAuth.
> La BD MySQL tiene índices críticos documentados y un script de análisis existente (`SchemasSQL/mysql_index_analyzer.py`).
>
> **¿Vale la pena agregar 3 tools MCP (`get_schema_health`, `get_slow_queries`, `run_schema_fix`) para que Claude Desktop pueda monitorear y corregir el schema sin salir del chat?**
>
> O dicho de otro modo: ¿es mejor infraestructura MCP o es suficiente con el script existente?

---

## 🖥️ Perspectiva Code
*Lente: implementación, seguridad, convenciones, esfuerzo técnico, riesgos de ejecución*

**Postura:** Condicional — vale la pena, pero `run_schema_fix` necesita salvaguardas

**Puntos clave:**
- Los 3 tools siguen exactamente el patrón que ya existe en `routes/mcp.js` (query_portfolio, execute_order, etc.) — no hay arquitectura nueva que aprender, es agregar endpoints Node.js a algo que ya funciona
- `get_schema_health` y `get_slow_queries` son read-only — riesgo casi cero, alto valor informativo
- El script `mysql_index_analyzer.py` requiere ejecutar Python local. Los tools MCP los puede invocar Desktop desde cualquier lugar (teléfono, otra máquina) — eso es una ventaja real dado el objetivo de portabilidad del proyecto
- `run_schema_fix` es el único con riesgo: CREATE INDEX en una tabla de 1M+ filas puede bloquear escrituras en MySQL 5.x (en MySQL 8 es online DDL — ya tenemos 8.x, OK). Aun así requiere `confirm=true` como `execute_order`
- El audit log en `mcp_audit.jsonl` ya existe — solo hay que logear las llamadas nuevas, no construir nada

**Riesgos que ve:**
- `run_schema_fix` mal usado podría ejecutar `OPTIMIZE TABLE` en `fund_holdings` (1.16M filas) y bloquear temporalmente la app — mitigación: whitelist de operaciones permitidas + timeout explícito
- Si el Tunnel cae, Desktop no puede acceder — pero eso ya aplica a todos los tools actuales, no es riesgo nuevo
- Costo de desarrollo: estimo 2-3 horas para los 3 tools con tests básicos en `AppTest/`

**Condiciones para proceder:**
- `run_schema_fix` solo acepta operaciones de la whitelist: `CREATE INDEX`, `ANALYZE TABLE`, `OPTIMIZE TABLE` — no DDL libre
- Requiere `confirm=true` para cualquier escritura
- Loguear en `mcp_audit.jsonl` con timestamp, operación, tabla, resultado
- Probar primero `get_schema_health` solo (read-only) antes de agregar `run_schema_fix`

---

## 🖱️ Perspectiva Desktop
*Lente: valor estratégico, experiencia de uso, ROI, simplicidad, visión de largo plazo*

> **[Pendiente — Claude Desktop completa esta sección]**
>
> Sugerencia de qué evaluar:
> - ¿Con qué frecuencia real necesitarías corregir el schema? ¿Justifica la infraestructura?
> - ¿El script existente es realmente un problema o es suficiente?
> - ¿Hay algo que Code no está viendo desde su lente de implementación?
> - ¿Qué pasa si en 6 meses el schema está estable — sigue siendo útil el tool?

---

## 🔄 Réplicas (si aplica)

*[Se completa después de que Desktop escriba su perspectiva]*

---

## 🎯 Síntesis

*[Se completa cuando ambas perspectivas estén escritas]*

---

## Historial
| Fecha | Quién | Acción |
|-------|-------|--------|
| 2026-07-12 | Code | Escribió perspectiva inicial |
| | Desktop | Pendiente |
