---
tipo: gestion
tags: [cowork, handoff, flujo]
fuente_de_verdad: true
---

# Handoff Claude — Ideas entre interfaces

Registro de ideas que nacen en una interfaz de Claude y se ejecutan en otra.

**Interfaces:**
- **Code** = Claude Code en VS Code — implementación, git, archivos, BD directa
- **Desktop** = Claude Desktop (Chat/Cowork) — análisis, monitoreo, MCP vía Tunnel, acciones correctivas

---

## Patrón de entrada

Cada idea sigue esta estructura:

```
## [ID] Título corto

**Origen:** Code → Desktop  |  Desktop → Code
**Estado:** Pendiente | En progreso | Listo

**Contexto:**
Qué se analizó, qué problema resuelve, qué ya existe.

**Prerequisitos:**
Qué debe estar listo antes de ejecutar (ej: tools MCP a construir en Code).

**Tarea para [interfaz destino]:**
Instrucción concreta de lo que debe hacer.

**Resultado esperado:**
Qué se considera hecho.
```

---

## Ideas pendientes

---

### H-001 — Monitoreo y corrección de schema MySQL

**Origen:** Code → Desktop
**Estado:** Pendiente (prerequisitos en Code)

**Contexto:**
En sesión 2026-07-12 se identificó la necesidad de pasar de reportar problemas de schema (ya cubierto por `SchemasSQL/mysql_index_analyzer.py`) a tomar acciones correctivas. La BD tiene 12+ índices críticos documentados en CLAUDE.md. Claude Desktop es el agente ideal para monitoreo recurrente + correcciones puntuales vía MCP.

**Prerequisitos (a implementar en Code):**
Agregar 3 tools en `MyNode/server-api/routes/mcp.js`:

| Tool | Descripción |
|------|-------------|
| `get_schema_health` | Tablas sin índices, full scans, config InnoDB vs recomendado, tamaño tablas críticas |
| `get_slow_queries` | Lee slow_query_log — filtra por tabla, tiempo, frecuencia |
| `run_schema_fix` | Crea índice o ANALYZE/OPTIMIZE tabla. Requiere `confirm=true` para ejecutar |

Todas deben loguearse en `logs/mcp_audit.jsonl`.

**Tarea para Desktop:**
Una vez implementados los tools:
1. Llamar `get_schema_health` → identificar problemas actuales
2. Para cada problema → `run_schema_fix(confirm=false)` primero (simular)
3. Proponer al usuario las acciones y ejecutar con `confirm=true` las aprobadas
4. Recurrencia sugerida: lunes 8am (ya hay slot de reporte HTML en Calendar)

**Resultado esperado:**
Schema sin full scans en tablas críticas. Correcciones auditadas en `mcp_audit.jsonl`.

---

## Ideas completadas

*(vacío por ahora)*

---

## Notas del patrón

- **Code implementa, Desktop opera** — la regla general. Code construye la infraestructura (tools MCP, agentes, tablas). Desktop usa esa infraestructura para monitoreo y decisiones.
- **Desktop detecta, Code implementa** — cuando Desktop encuentra algo durante análisis que requiere cambio de código → documenta aquí con `Desktop → Code` y lo retoma la próxima sesión en VS Code.
- **No duplicar en BACKLOG** — los prerequisitos de Code sí van al BACKLOG (ítem 68 y derivados). Este archivo captura el *flujo completo* de la idea, no solo la tarea técnica.
