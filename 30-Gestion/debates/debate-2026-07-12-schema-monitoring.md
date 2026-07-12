---
tipo: debate
estado: Abierto — ambas perspectivas escritas, falta síntesis
---

# Debate — Tools MCP para monitoreo y corrección de schema MySQL

**Fecha:** 2026-07-12
**Propuesto por:** usuario (sesión VS Code)
**Estado:** 🟡 Abierto — ambas perspectivas escritas, falta síntesis

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

**Postura:** A favor de Fase 1 (solo lectura) ya mismo — `run_schema_fix` mejor diferido hasta que haya una necesidad real confirmada

**Puntos clave:**
- La visión del proyecto es 24x7 + portabilidad (posible escenario sin laptop a mano, cortes de luz/internet, retorno a Venezuela). Poder chequear salud del schema desde el teléfono sin abrir VS Code es exactamente el tipo de resiliencia que se busca — eso solo ya justifica `get_schema_health` y `get_slow_queries`, independiente de cuán seguido se usen.
- El costo es bajo (2-3h, mismo patrón ya probado en `routes/mcp.js`) y el valor de monitoreo no caduca: `fund_holdings` ya tiene 1.16M filas y sigue creciendo con el pipeline 13F diario/semanal — más motivo para tener visibilidad remota, no menos, aunque el schema hoy esté estable.
- Dudo del ROI de `run_schema_fix` en particular: un `CREATE INDEX` casi siempre nace junto con un cambio de código (se agrega una tabla o columna nueva → se agrega su índice) — eso pasa en sesión Code, no en un chat desde el celular. No veo todavía el escenario real donde hace falta corregir schema sin tocar código al mismo tiempo.
- Mismo criterio que propuso Code para el otro debate (umbral N=2, no construir especulativamente): si en las próximas semanas se repite de verdad la necesidad de corregir algo desde Desktop sin pasar por Code, ahí se justifica construirlo.

**Riesgos que veo:**
- Construir `run_schema_fix` ahora es superficie de ataque + mantenimiento (whitelist, audit) para un caso de uso todavía no confirmado — si no se usa, queda código con permisos de escritura sobre la BD real sin necesidad real detrás.
- Lo inverso también aplica: si NO se construyen los tools de solo lectura, se sigue dependiendo de abrir laptop/VS Code para diagnosticar un problema de performance — eso choca directo con la visión de reducir intervención manual.

**Conclusión estratégica:**
Construir ya `get_schema_health` y `get_slow_queries` (Fase 1 de Code). Pausar `run_schema_fix` — no por desconfianza en la salvaguarda técnica (la whitelist + `confirm=true` que propone Code está bien pensada), sino porque falta evidencia de que el caso de uso ocurra en la práctica. Retomarlo cuando aparezca la primera vez real que se necesite.

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
| 2026-07-12 | Desktop | Escribió perspectiva estratégica — a favor de Fase 1 (solo lectura), pausar `run_schema_fix` |
