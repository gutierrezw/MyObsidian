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

## Arquitectura del sistema — lo que cada interfaz ve

> Esta sección existe para que Desktop no tenga que redescubrir cómo funciona el sistema cada vez.

### Memoria y contexto

| Archivo | Quién lo lee | Propósito |
|---------|-------------|-----------|
| `~/.claude/CLAUDE.md` | Solo Code | Convenciones de código Python, reglas de estilo, arquitectura de agentes. **No poner referencias a docs de Obsidian aquí.** |
| `memory/MEMORY.md` | Solo Code | Índice de memoria del proyecto. Se carga automáticamente en cada sesión Code. Es el lugar correcto para referenciar docs de Obsidian. |
| `10-Memoria/00-Home.md` | Solo Desktop | Índice central del vault Obsidian. Desktop arranca siempre leyendo este archivo. |
| `10-Memoria/MEMORY.md` | Solo Desktop | Copia/junction de memory/MEMORY.md — mismo contenido que lee Code. |
| `30-Gestion/handoff-claude.md` | **Ambos** | Este archivo. Canal de comunicación entre interfaces. |
| `30-Gestion/debates/` | **Ambos** | Debates activos. Cada interfaz completa su sección. |

### Regla clave para Desktop
**No agregar referencias a docs en `~/.claude/CLAUDE.md` global** — ese archivo es solo para convenciones de código. Si encontrás un doc relevante que Code debería conocer, agregalo en `memory/MEMORY.md` (o pedile a Code que lo haga). Code lo levantará en la próxima sesión automáticamente.

### Lo que Desktop no ve sin que se lo pasen
- Estado del repo git (commits recientes, ramas, archivos modificados)
- Contenido de archivos fuera de Obsidian (código Python, Node.js)
- Scripts ad-hoc que corrió Code en sesiones anteriores
- Qué archivos son sensibles y no deben commitearse (oauth_tokens.json, credentials, etc.)

### Lo que Code no ve sin que se lo pasen
- Conversaciones anteriores de Desktop (a menos que estén en Obsidian)
- Qué analizó Desktop fuera del vault
- Decisiones tomadas en sesiones Desktop que no se documentaron

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


---

## Ideas completadas

### H-001 — Monitoreo schema MySQL ✅
**Origen:** Code → Desktop | **Fecha:** 2026-07-12
Debate `debate-2026-07-12-schema-monitoring.md` cerrado con consenso. Desktop implementó `get_schema_health` + `get_slow_queries` en `routes/mcp.js` (commit dd44229). `run_schema_fix` pausado hasta caso de uso real.

---

## Notas del patrón

- **Code implementa, Desktop opera** — la regla general. Code construye la infraestructura (tools MCP, agentes, tablas). Desktop usa esa infraestructura para monitoreo y decisiones.
- **Desktop detecta, Code implementa** — cuando Desktop encuentra algo durante análisis que requiere cambio de código → documenta aquí con `Desktop → Code` y lo retoma la próxima sesión en VS Code.
- **No duplicar en BACKLOG** — los prerequisitos de Code sí van al BACKLOG (ítem 68 y derivados). Este archivo captura el *flujo completo* de la idea, no solo la tarea técnica.
