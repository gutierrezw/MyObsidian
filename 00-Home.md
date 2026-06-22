# AppOO — Centro de Control

> Trading platform | Dividendos + Oportunidades | Claude co-work

---

## Navegación rápida

| Área | Descripción |
|------|-------------|
| [[10-Memoria/MEMORY\|Memoria Claude]] | Índice de toda la memoria entre sesiones |
| [[20-Proyecto/INDEX\|Índice Proyecto]] | Todos los docs de diseño y arquitectura |
| [[30-Gestion/BACKLOG\|BACKLOG]] | Tareas pendientes y prioridades |
| [[30-Gestion/Sesiones\|Sesiones]] | Notas por sesión de trabajo |
| [[30-Gestion/Dashboard\|Dashboard Dataview]] | Vistas dinámicas: sesiones, decisiones, memoria |
| [[30-Gestion/Skills\|Skills y Agentes]] | Slash commands disponibles y cuándo usarlos |

## Docs clave

### Arquitectura central
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/DataHub_estructura\|DataHub]] | Estructura de procesos y monitoring |
| [[20-Proyecto/CacheHut_estructura\|CacheHut]] | Caché de datos de mercado |
| [[20-Proyecto/Class_IA_modelos\|Class IA Modelos]] | Clases del motor IA |
| [[20-Proyecto/PROJECT_INSTRUCTIONS\|Project Instructions]] | Instrucciones generales |

### Modelos de decisión IA
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/agente_ia_autonomo_design\|Agente IA Autónomo]] | Diseño motor de decisiones |
| [[20-Proyecto/modelo_buyv01\|Modelo BUY v01]] | Lógica de compra |
| [[20-Proyecto/modelo_sellv01\|Modelo SELL v01]] | Lógica de venta |
| [[20-Proyecto/documento_regla_buy_vs_dividends_y_estructura_de_datos\|Regla BUY vs Dividendos]] | Criterio de selección |
| [[20-Proyecto/preservation_claude_dynamic_design\|Preservation Claude]] | Stop dinámico con IA |
| [[20-Proyecto/modulo_autoremediation\|Auto-Remediación]] | Corrección automática |

### Screener y mercado
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/screener_consenso_modelo\|Screener + Consenso]] | Modelo de scoring institucional |
| [[20-Proyecto/consenso_expansion_design\|Consenso Score — expansión]] | Votos y señales |
| [[20-Proyecto/YahooFinance_REST_Market_Loader\|Yahoo Finance Loader]] | Descarga de datos de mercado |
| [[20-Proyecto/sentimiento_design\|Sentimiento]] | Análisis de sentimiento |
| [[20-Proyecto/gains_capture_design\|Gains Capture]] | Captura de ganancias |

### Trading y portfolio
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/rebalanceo_diseno_unificado_y_estado_actual_V3\|Rebalanceo V3]] | Estrategia + restricciones |
| [[20-Proyecto/BotCrypto_especificacion_tecnica\|BotCrypto]] | Trading automático crypto |
| [[20-Proyecto/ltv_agent\|LTV Agent]] | Control préstamo Binance |
| [[20-Proyecto/Analisis Stock - modulo_apalancamiento IB\|Apalancamiento IB]] | Módulo Interactive Brokers |

### Infraestructura y MCP
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/AppOO_GitHub_MCP_spec\|GitHub MCP]] | Especificación MCP |
| [[20-Proyecto/ConvergIA_especificacion\|ConvergIA]] | Especificación multi-agente |
| [[20-Proyecto/multi_agent_debate_spec\|Multi-Agent Debate]] | Arquitectura debate IA |
| [[20-Proyecto/Conectore MCP_Conversation_Summary\|Conectores MCP]] | Resumen conectores |

### Finanzas y usuario
| Doc | Descripción |
|-----|-------------|
| [[20-Proyecto/modulo_finanzas\|Finanzas Personales]] | Gmail + finanzas |
| [[20-Proyecto/Instalacion APP\|Instalación AppOO]] | Guía de instalación |
| [[20-Proyecto/instalacion_hijo\|Instalación hijo]] | Versión hijo |

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
| [[30-Gestion/Sesiones\|Sesiones]] | Historial de sesiones de co-work |

## Links útiles

- [[20-Proyecto/consenso_expansion_design|Consenso Score — diseño]]
- [[20-Proyecto/preservation_claude_dynamic_design|Preservation dinámico Claude]]
- [[20-Proyecto/multi_agent_debate_spec|Multi-Agent Debate]]
- [[20-Proyecto/modulo_finanzas|Módulo Finanzas Personales]]
