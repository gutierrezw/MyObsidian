---
tipo: debate
estado: Abierto — esperando perspectiva Desktop
---

# Debate — Auto-detección de queries recurrentes → endpoint MCP automático

**Fecha:** 2026-07-12
**Propuesto por:** usuario (sesión VS Code)
**Estado:** 🟡 Abierto — Code escribió, falta Desktop

---

## Tema / Pregunta

> Hoy cuando Claude (Code o Desktop) necesita consultar la BD para algo que no está en el MCP,
> tiene que hacer un script externo ad-hoc. Eso es fricción.
>
> La idea: **detectar automáticamente cuando un patrón de consulta se repite** en sesiones de
> co-work y, al superar un umbral de recurrencia, sugerir o crear el endpoint MCP correspondiente.
>
> **¿Cómo implementar esto? ¿Vale la pena la complejidad vs. simplemente hacerlo manual?**
> ¿Dónde vive la lógica de detección — en el servidor, en el audit log, en Obsidian?

---

## 🖥️ Perspectiva Code
*Lente: implementación, seguridad, convenciones, esfuerzo técnico, riesgos de ejecución*

**Postura:** A favor — pero la detección debe ser pasiva, no intrusiva

**Puntos clave:**
- La materia prima ya existe: `logs/mcp_audit.jsonl` loguea cada tool call con timestamp, tool name y parámetros. Si además loguiéramos los scripts externos que Claude corre (vía hook o convención), tendríamos el dataset completo de fricción
- El mecanismo más simple: un tool MCP `get_friction_report` que analiza el audit log y devuelve patrones — queries que Claude hizo fuera del MCP más de N veces en los últimos 30 días. Desktop lo corre periódicamente y propone al usuario qué endpoints crear
- No hace falta auto-crear el endpoint (riesgo de endpoints basura) — con **detectar + proponer** alcanza. La creación sigue siendo manual (Code la implementa en la siguiente sesión VS Code)
- El umbral N=2 es razonable: si lo pediste dos veces, lo vas a pedir una tercera

**Riesgos que ve:**
- Si el logging de scripts externos no es sistemático, la detección es incompleta — solo ve la mitad del problema (las calls MCP, no los scripts Python)
- Auto-crear endpoints sin revisión puede llenar `routes/mcp.js` de tools de baja calidad
- Requiere disciplina: Claude Code tiene que loguear sus scripts ad-hoc en algún lugar estándar para que el sistema los detecte

**Condiciones para proceder:**
- Fase 1: solo detección pasiva sobre `mcp_audit.jsonl` existente — costo casi cero
- Fase 2: convención para que Claude logúe scripts ad-hoc (ej: comentario estándar al inicio del script que queda en el historial de sesión)
- Fase 3 (si Fase 1 y 2 funcionan bien): auto-sugerencia vía Telegram cuando se supera el umbral

---

## 🖱️ Perspectiva Desktop
*Lente: valor estratégico, experiencia de uso, ROI, simplicidad, visión de largo plazo*

> **[Pendiente — Claude Desktop completa esta sección]**
>
> Sugerencia de qué evaluar:
> - ¿Esto resuelve un dolor real o es sobre-ingeniería?
> - ¿La fricción actual (scripts ad-hoc) es tan frecuente como para justificar un sistema de detección?
> - ¿Hay una solución más simple que no requiera infraestructura nueva?
> - Visión de largo plazo: si la app escala a N usuarios, ¿este mecanismo sigue siendo útil?

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
