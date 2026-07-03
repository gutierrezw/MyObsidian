# AppOO — Centro de Control

> Trading platform | Dividendos + Oportunidades | Claude co-work

---

## Navegación rápida

| Área | Descripción |
|------|-------------|
| [[10-Memoria/MEMORY\|Memoria Claude]] | Índice de toda la memoria entre sesiones |
| **Docs de diseño** (abajo) | Todos los docs de arquitectura, spec y diseño |
| [[30-Gestion/BACKLOG\|BACKLOG]] | Tareas pendientes y prioridades |
| [[30-Gestion/Dashboard\|Dashboard Dataview]] | Vistas dinámicas: decisiones, memoria |
| [[30-Gestion/Skills\|Skills y Agentes]] | Slash commands disponibles y cuándo usarlos |

## Docs clave

### Arquitectura central
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/ref-datahub\|DataHub]] | Estructura de procesos y monitoring |
| [[20-Proyecto/ref-cache\|CacheHut]] | Caché de datos de mercado |
| [[20-Proyecto/ref-modelos-ia\|Class IA Modelos]] | Clases del motor IA |
| [[20-Proyecto/PROJECT_INSTRUCTIONS\|Project Instructions]] | Instrucciones generales |

### Modelos de decisión IA
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/design-agente-ia\|Agente IA Autónomo]] | Diseño motor de decisiones |
| [[20-Proyecto/ref-oportunidades\|Oportunidades BUY/SELL]] | Reglas, umbrales, score híbrido TOP10 |
| [[20-Proyecto/ref-dividendos\|Dividendos]] | Lógica de selección por dividendos |
| [[20-Proyecto/design-preservation\|Preservation Claude]] | Stop dinámico con IA |
| [[20-Proyecto/spec-autoremediation\|Auto-Remediación]] | Corrección automática |

### Screener y mercado
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/ref-consenso\|Consenso Score]] | Votos, tags, gate Telegram, columnas Screener — referencia rápida |
| [[20-Proyecto/spec-consenso\|Consenso Score (técnico)]] | Pipeline detallado, fórmulas, decisiones de diseño (archivo histórico) |
| [[20-Proyecto/ref-yahoo-loader\|Yahoo Finance Loader]] | Descarga de datos de mercado |
| [[20-Proyecto/design-sentimiento\|Sentimiento]] | Análisis de sentimiento — flujo técnico |
| [[20-Proyecto/design-gains-capture\|Gains Capture]] | Captura de ganancias |

### Trading y portfolio
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/design-rebalanceo\|Rebalanceo]] | Estrategia + restricciones |
| [[20-Proyecto/spec-botcrypto\|BotCrypto]] | Arquitectura + especificación técnica |
| [[20-Proyecto/spec-ltv-agent\|LTV Agent]] | Control préstamo Binance |
| [[20-Proyecto/spec-apalancamiento-ib\|Apalancamiento IB]] | Módulo Interactive Brokers |

### Infraestructura y MCP
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/design-api-server\|server-api Node]] | Bridge AppOO ↔ TradingView/co-work (3 fases) |
| [[20-Proyecto/ref-cowork-tools\|Co-work Tools]] | Tools MCP disponibles para Claude + gaps actuales |
| [[20-Proyecto/spec-github-mcp\|GitHub MCP]] | Especificación MCP |
| [[20-Proyecto/spec-convergia\|ConvergIA]] | Especificación multi-agente |
| [[20-Proyecto/spec-multi-agent\|Multi-Agent Debate]] | Arquitectura debate IA |
| [[20-Proyecto/ref-mcp-conectores\|Conectores MCP]] | Resumen conectores |

### Finanzas y usuario
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/spec-finanzas\|Finanzas Personales]] | Gmail + finanzas |
| [[20-Proyecto/ref-instalacion\|Instalación AppOO]] | Guía de instalación |
| [[20-Proyecto/ref-instalacion-hijo\|Instalación hijo]] | Versión hijo |

---

## Estado del proyecto

- **Cuenta stock:** U4214563
- **Cuenta crypto:** B0000001 / B0000002
- **DB:** MySQL 8.x — schema `bdinv`
- **Branch docs:** `docs` (fuente de verdad para .md)

---

## Módulos activos

```
Class_DashBot     → Chatbot + Agentes IA + Decisiones Autónomas
Class_BrowserFCI  → Descarga automática BBVA + Santander
Class_Screener    → Screener NASDAQ + Consenso Score
Class_vehiculo    → Binance Prod/Testnet
AgentManager      → Infraestructura lenta (EDGAR 13F, NASDAQ)
```

---

## Configuración co-work Claude

| Archivo | Descripción |
|---------|-------------|
| [CLAUDE.md](file:///C:/Users/InversionesWildaga/Documents/MyPython/AppOO/CLAUDE.md) | Instrucciones del proyecto — convenciones, checklist inicio/cierre |
| [CLAUDE.md global](file:///C:/Users/InversionesWildaga/.claude/CLAUDE.md) | Convenciones Python globales |

## Links útiles

- [[20-Proyecto/ref-consenso|Consenso Score]]
- [[20-Proyecto/design-preservation|Preservation dinámico Claude]]
- [[20-Proyecto/spec-multi-agent|Multi-Agent Debate]]
- [[20-Proyecto/spec-finanzas|Módulo Finanzas Personales]]
- [[10-Memoria/ref-naming-docs|Convención nombres de documentos]]
