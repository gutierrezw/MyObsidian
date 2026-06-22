---
fecha: 2026-06-22
tipo: referencia
tags: [claude, skills, agentes]
---

# Skills y Agentes — Claude Code

## Skills (slash commands)

Invocás escribiendo `/nombre` en el chat de Claude Code.

| Skill | Comando | Cuándo usarlo |
|-------|---------|---------------|
| Claude API | `/claude-api` | Cuando querés integrar Claude en el código de AppOO — genera llamadas al SDK de Anthropic en Python. Soporta streaming, tool use, structured output |

### `/claude-api` — detalle

Genera código Python usando el SDK oficial de Anthropic. Útil para:
- Agregar llamadas a Claude en `Class_DashBot.py` (chatbot, decisiones IA)
- Integrar Claude en preservation dinámico (`preservation_claude_dynamic_design`)
- Implementar el debate multi-agente (`multi_agent_debate_spec`)

Modelo por defecto que usa: `claude-opus-4-8`  
Soporta: streaming · tool use · structured output · extended thinking

---

## Agentes especializados (Agent tool)

Claude los usa internamente. Podés pedirle a Claude que los use explícitamente.

| Agente | Cuándo pedirlo |
|--------|----------------|
| **Explore** | "Buscá en el código dónde está X" — búsqueda rápida sin modificar nada. Ideal para localizar símbolos, archivos, referencias |
| **Plan** | "Diseñá el plan de implementación para X" — devuelve un plan paso a paso con trade-offs antes de escribir código |
| **claude-code-guide** | "¿Cómo funciona X en Claude Code?" — preguntas sobre la herramienta, hooks, MCP, settings |
| **statusline-setup** | Configurar la barra de estado de Claude Code en VS Code |
| **general-purpose** | Investigación amplia, búsquedas en múltiples archivos, tareas multi-step complejas |

---

## Cómo crear una skill propia

Skills del proyecto van en: `AppOO/.claude/skills/`  
Skills globales van en: `C:\Users\InversionesWildaga\.claude\skills\`

Cada skill es una carpeta con un archivo `skill.md` que define el comportamiento.  
Se invoca con `/<nombre-carpeta>`.

> Por ahora no hay skills custom definidas — oportunidad para automatizar flujos repetitivos del proyecto.

---

## Ideas de skills a crear

| Skill | Utilidad |
|-------|----------|
| `/nueva-sesion` | Ejecuta el checklist de inicio: lee última sesión + Home |
| `/cierre-sesion` | Genera la nota de sesión automáticamente |
| `/backlog` | Muestra el BACKLOG desde rama `docs` formateado |
| `/decision` | Crea una nota de decisión técnica con el template |
