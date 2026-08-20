# Agente IA Autónomo — Diseño del Sistema

> Objetivo: Claude opera alineado a la misión de inversión del usuario.
> Automatizar "Seguimiento y Operaciones" — ítem explícito del Plan de Inversión (Gestión).

---

## Visión

Esto no es un bot de trading. Es un sistema de gestión de inversiones con IA incorporada.

Lo que se fue construyendo tiene una coherencia real:

- Los **datos** ya existen (posiciones, mercado, rebalanceo, riesgo)
- Las **señales** ya votan (consenso, 13F, sentimiento, técnicos)
- El **agente** ya decide y registra
- El **chat** ya conversa con contexto y memoria
- El **teletipo** ya muestra el acontecer autónomo

Lo que le falta para ser un sistema completo de gestión es que el **objetivo sea editable** desde la UI. Porque hoy Claude recibe la misión como texto fijo, pero la misión en realidad evoluciona: antes era acumular capital, ahora el 3% puede escalar, mañana puede cambiar el año o la meta.

Cuando eso esté, tenés un sistema donde:

1. Definís tu plan con sus riesgos y objetivos → desde la UI
2. El agente opera alineado a ese plan → sin intervención
3. El chat te explica por qué tomó cada decisión → con memoria
4. El teletipo muestra el acontecer en tiempo real → en la misma ventana

---

## Vista General — Las 6 Capas

> **Capa 6 — Meta-Monitoreo:** el agente también observa cómo funciona el propio sistema
> y genera un backlog interno de mejoras. Separado del backlog de co-work con el usuario.

```
╔══════════════════════════════════════════════════════════════════════╗
║  MISIÓN  (Gestión)                                                   ║
║  Meta: 1.2M USD en 2030 · Ingresos pasivos 3% anual · Apalancado    ║
╚══════════════════════╦═══════════════════════════════════════════════╝
                       ║ define objetivos y límites
                       ▼
╔══════════════════════════════════════════════════════════════════════╗
║  CAPA 1 — CONFIGURACIÓN  ✅ IMPLEMENTADO                             ║
║                                                                      ║
║  sesion.monto_por_trade = 170   → monto BASE; real = max(170, price) ║
║  sesion.gainInversion = 100     → si gain_usd ≥ 100 → eval captura  ║
║  sesion.llave_privada (JSON)    → agente_ia + preservation           ║
║                                                                      ║
║  Bloque agente_ia en llave_privada (vehiculo=Stock):                 ║
║    activo: false → true para activar                                 ║
║    monto_por_trade: 170 (ajustado al precio si precio > 170)         ║
║    gate_consenso_min: 4                                              ║
║    gate_inst_score_min: 0.5                                          ║
║    leverage_max: 1.8 · risk_real_max: 2.0 · deuda_max_pct: 35       ║
╚══════════════════════╦═══════════════════════════════════════════════╝
                       ║ parametriza
                       ▼
╔══════════════════════════════════════════════════════════════════════╗
║  CAPA 2 — DATOS  ✅ IMPLEMENTADO (2026-06-15)                        ║
║                                                                      ║
║  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐  ║
║  │  PORTFOLIO       │  │  MERCADO          │  │  RIESGO / DEUDA    │  ║
║  │                 │  │                  │  │                    │  ║
║  │ posiciones      │  │ lastPrice        │  │ deuda_total_usd    │  ║
║  │ avgcost         │  │ consenso_tag     │  │ margen_libre_ib    │  ║
║  │ ROI actual      │  │ consenso_suma    │  │ leverage_actual    │  ║
║  │ gain_usd        │  │ inst_score       │  │ risk_real (Lev×β)  │  ║
║  │ dividendos      │  │ sentiment_score  │  │ ltv_binance        │  ║
║  │ encartera Y/N   │  │ dividendYield    │  │ mrg_risk_pct       │  ║
║  │ categoriaActivo │  │ monto_sugerido*  │  │                    │  ║
║  └─────────────────┘  └──────────────────┘  └────────────────────┘  ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────────┐ ║
║  │  REBALANCEO — 4 DIMENSIONES (DataHub.manager_buysell)            │ ║
║  │                                                                  │ ║
║  │  sector   → peso%, objetivo=1/N, gap_pct, gap_valor             │ ║
║  │  region   → peso%, objetivo=1/N, gap_pct, gap_valor             │ ║
║  │  activos  → tipos (Exchange/Oro/Stock/Crypto/Bonos/Cash)        │ ║
║  │  dividends→ ingreso total portafolio (anual)                    │ ║
║  │                                                                  │ ║
║  │  ▼ subponderado · ▲ sobreponderado · ✓ equilibrado (gap<3%)     │ ║
║  └──────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────────┐ ║
║  │  RANKING REBALANCEO — Top 5 (DataHub.rebalanceo["Stock"])        │ ║
║  │                                                                  │ ║
║  │  Generado por RebalanceEngine.rank() — símbolos priorizados      │ ║
║  │  por score estructural: gaps normalizados × dimensión faltante   │ ║
║  │  Incluye: symbol · score · dimension · monto_sugerido            │ ║
║  └──────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────────┐ ║
║  │  OPORTUNIDADES BUY (DataHub.info[sym]["buy" | "dividends"])      │ ║
║  │                                                                  │ ║
║  │  Calculadas por oportunidades_buy() sobre posiciones activas:    │ ║
║  │  · tipo: "buy" (sin dividendo) o "dividends" (con yield>0)      │ ║
║  │  · ganancia_inversión: ROI proyectado si se compra ahora        │ ║
║  │  · cantidad_buy: acciones a adquirir con monto_base             │ ║
║  │  · avgCost_post: nuevo costo promedio tras la compra            │ ║
║  │  · dividendYield: tasa anual proyectada post compra             │ ║
║  │  Ordenadas por gain_inversion ascendente (más atractivas primero)│ ║
║  └──────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────────┐ ║
║  │  OPORTUNIDADES SELL (DataHub.info[sym]["sell"])                  │ ║
║  │                                                                  │ ║
║  │  Calculadas por oportunidades_sell() — lotes ganadores:          │ ║
║  │  · profit: ganancia total USD captuable                         │ ║
║  │  · roi: % de ganancia sobre costo acumulado de lotes            │ ║
║  │  · cantidad_sell: acciones con ganancia disponibles             │ ║
║  │  · lotes: número de compras individuales en ganancia            │ ║
║  │  Ordenadas por profit descendente (más ganancia primero)         │ ║
║  └──────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────────┐ ║
║  │  CANDIDATOS EXTERNOS (IaTrace.select_candidatos_ia)              │ ║
║  │                                                                  │ ║
║  │  Activos del Screener NO en cartera con consenso ≥ gate_min:    │ ║
║  │  · consenso_suma · inst_score · dividendYield                   │ ║
║  │  · monto_sugerido = max(monto_base, lastPrice)                  │ ║
║  └──────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  * Claude cruza las 3 fuentes BUY (oport_buy + candidatos externos  ║
║    + ranking rebalanceo) y prioriza lo que aparece en más de una.   ║
║                                                                      ║
║  Fuentes: IBroks_Client · MarketScreen · Agente_LtvControl           ║
║           Agente_ConsensoCache · Agente_13FScores · RebalanceEngine  ║
║           oportunidades_buy() · oportunidades_sell() en TickerInfo  ║
╚══════════════════════╦═══════════════════════════════════════════════╝
                       ║ alimenta
                       ▼
╔══════════════════════════════════════════════════════════════════════╗
║  CAPA 3 — SEÑALES  ✅ EXISTE                                         ║
║                                                                      ║
║  Voto Net    · ¿flujo institucional neto positivo?                   ║
║  Voto Opt    · ¿opciones Call dominan?                               ║
║  Voto Flujo  · ¿entradas > salidas institucionales?                  ║
║  Voto Ana    · ¿análisis técnico alinea?                             ║
║  Voto Val    · ¿precio está por debajo del valor justo?              ║
║  Voto Cob    · ¿cobertura de analistas es positiva?                  ║
║  Voto Sent   · ¿sentimiento de noticias acompaña? (🧪 en prueba)    ║
║                                                                      ║
║  Resultado → consenso_tag: UNANIME · CONSENSO · TENDENCIA           ║
║                             NEUTRAL · ALERTA · SALIDA                ║
╚══════════════════════╦═══════════════════════════════════════════════╝
                       ║ informa
                       ▼
╔══════════════════════════════════════════════════════════════════════╗
║  CAPA 4 — DECISIÓN  ✅ IMPLEMENTADO — Agente_ClaudeIA                ║
║                                                                      ║
║  @wait_rate(86400, persist=True) — corre 1 vez por día               ║
║  Modelo: claude-haiku-4-5-20251001                                   ║
║                                                                      ║
║  Contexto que recibe Claude:                                         ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │  Portfolio: symbol · ROI% · mkt$ · gain_usd · [GAINS?]     │     ║
║  │  Posiciones con ganancia ≥ min_ganancia (gains_candidate)   │     ║
║  │  Rebalanceo: sector/región/tipos con peso% y ▼▲✓            │     ║
║  │  Dividendos: ingreso total portafolio                       │     ║
║  │  Candidatos: consenso_suma · inst_score · yield             │     ║
║  │              monto_sugerido = max(monto_base, lastPrice)    │     ║
║  │  Límites: monto_base · leverage_max · inst_score_min        │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                      ║
║  Instrucciones al modelo:                                            ║
║  · BUY → priorizar candidatos alineados con dimensiones ▼            ║
║  · BUY → usar monto_sugerido del candidato elegido                   ║
║  · [GAINS?] → evaluar si el contexto justifica captura (SELL) u HOLD ║
║                                                                      ║
║  Decisiones posibles:                                                ║
║  → BUY    símbolo + monto + justificación                            ║
║  → HOLD   motivo registrado                                          ║
║  → SELL   símbolo (gains capture o salida)                           ║
║  → ALERTA condición de riesgo detectada                              ║
╚══════════════════════╦═══════════════════════════════════════════════╝
                       ║ registra / ejecuta
                       ▼
╔══════════════════════════════════════════════════════════════════════╗
║  CAPA 5 — TRAZABILIDAD + UI  ✅ IMPLEMENTADO (Fase 1)                ║
║                                                                      ║
║  ┌──────────────────────┐  ┌───────────────────────────────────────┐ ║
║  │  Panel IA Trace       │  │  Asistente de Inversión (chat)        │ ║
║  │                      │  │                                       │ ║
║  │  Tab System → IA Trace│  │  Ventana flotante — Claude API        │ ║
║  │  Auto-refresh 60s    │  │  Memoria persistente entre sesiones   │ ║
║  │  Timer próximo ciclo │  │  Multi-turno (historial completo)     │ ║
║  │  Colores por decisión│  │  Contexto: portfolio actual           │ ║
║  │  BUY=verde SELL=rojo │  │  Guardado en chatbot_history.json     │ ║
║  └──────────────────────┘  └───────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Capa 6 — Meta-Monitoreo  (backlog propio del agente)

El agente no solo opera — también observa cómo funciona el sistema y registra
oportunidades de mejora en su **propio backlog**, separado del backlog de co-work.

```
╔══════════════════════════════════════════════════════════════════════╗
║  CAPA 6 — META-MONITOREO  [pendiente Fase 3+]                        ║
║                                                                      ║
║  El agente observa:                                                  ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐    ║
║  │  Agentes del sistema                                         │    ║
║  │  · ¿Algún agente falló repetidamente?                        │    ║
║  │  · ¿Hay datos que siempre llegan vacíos?                     │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║  ┌──────────────────────────────────────────────────────────────┐    ║
║  │  Decisiones propias                                          │    ║
║  │  · ¿Qué gates rechazaron oportunidades que resultaron buenas?│    ║
║  │  · ¿El prompt produce decisiones consistentes?               │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  → Registra en tabla `ia_mejoras` (backlog propio del agente)        ║
║  → Se mostrará en Panel IA Trace (sub-tab "Mejoras")                 ║
║  → NO se mezcla con el BACKLOG.md de co-work                         ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Prompt real enviado a Claude (Fase 1)

```
Sos el agente de inversión autónomo. Misión: acumular capital hacia 1.2M USD en 2030
generando ingresos pasivos ≥3%/año. Foco en dividendos, uso moderado de apalancamiento IB.
En crisis → Hold o sumar posiciones, nunca vender por pánico.

Fecha: YYYY-MM-DD HH:MM

Portfolio actual (Stock):
  BP:   ROI=+12.3% | mkt=$830  | gain=$91
  PBR:  ROI=+18.1% | mkt=$5200 | gain=$390 [GAINS?]
  ABBV: ROI=+22.4% | mkt=$9800 | gain=$210 [GAINS?]

Posiciones con ganancia ≥$100: PBR, ABBV

Candidatos de entrada (consenso ≥ 4):
  MO  (Altria Group Inc      ): consenso_suma=6 inst_score=0.72 yield=8.2% monto_sugerido=$170
  T   (AT&T Inc              ): consenso_suma=5 inst_score=0.61 yield=6.1% monto_sugerido=$170
  VZ  (Verizon Comm          ): consenso_suma=5 inst_score=0.58 yield=6.8% monto_sugerido=$170
  NVDA(NVIDIA Corporation    ): consenso_suma=4 inst_score=0.81 yield=0.1% monto_sugerido=$135

Límites: monto_base=$170 (ajusta al precio del activo) | leverage_max=1.8x | inst_score_min=0.5

Para BUY: usá el monto_sugerido del candidato elegido.
Para [GAINS?]: evaluá si el contexto justifica captura parcial (SELL) u HOLD.

Producí UNA decisión con formato JSON exacto:
{"decision": "BUY|SELL|HOLD|ALERTA", "simbolo": "TICKER_O_VACIO", "monto": 0, "motivo": "max 150 chars"}
```

**Nota NVDA:** precio $135 > monto_base $170 → monto_sugerido=$135 (1 acción). Así ningún activo queda
excluido por precio — siempre se puede comprar al menos 1 acción.

---

## Flujo de una Decisión BUY

```
Agente_ClaudeIA corre (@wait_rate 86400 — 1 vez/día)
        │
        ├─ 1. Lee parámetros del vehículo (sesion.llave_privada)
        │        monto_por_trade=170, gate_consenso_min=4, leverage_max=1.8
        │
        ├─ 2. Lee portfolio de DataHub (posiciones + gain_usd por posición)
        │        Marca [GAINS?] donde gain_usd ≥ min_ganancia (100)
        │
        ├─ 3. Obtiene candidatos de mercado (consenso_suma ≥ 4, no en cartera)
        │        Calcula monto_sugerido = max(monto_por_trade, lastPrice)
        │
        ├─ 4. Llama Claude API (Haiku) con contexto completo
        │        → Claude decide: BUY candidato | SELL gains | HOLD | ALERTA
        │
        ├─ 5. Registra en ia_trace
        │        timestamp · símbolo · decisión · monto · motivo · PENDIENTE
        │
        └─ 6. [Fase 2] Envía propuesta Telegram con botones ✅/❌
```

---

## Asistente de Inversión (Chat)

Ventana flotante siempre disponible. Claude responde con contexto del portfolio.

| Característica | Detalle |
|---|---|
| Backend | Claude Haiku · sesion ClaudeAPIC |
| Memoria | Persiste entre sesiones → `tmp/chatbot_history.json` |
| Multi-turno | Historial completo enviado a Claude (últimos 40 mensajes) |
| Contexto | Portfolio actual (símbolo + precio) incluido en system prompt |
| Archivo | `Class_DashBot.py` → `Chatbot._enviar()` + `_consultar_claude()` |

---

## Panel IA Trace

Sub-tab en `System` → `IA Trace`.

| Feature | Detalle |
|---|---|
| Auto-refresh | Cada 60 segundos automático |
| Timer agente | Muestra "Agente_ClaudeIA: próximo en Xh YYm" |
| Columnas | ID · Fecha · Vehículo · Símbolo · Decisión · Monto · Motivo · Estado |
| Colores | BUY=verde · SELL=rojo · HOLD=gris · ALERTA=naranja |
| Fuente | `IaTraceScreen.select_trace(limit=100)` |
| Archivo | `Class_SystemStatus.py` → `ia_trace_system()` |

---

## Estado de implementación

| Componente | Estado | Archivo |
|---|---|---|
| DataHub — portfolio, riesgo, deuda | ✅ Existe | Class_customer.py |
| Consenso Score (7 votos) | ✅ Existe | Class_Screener.py |
| inst_score v2 | ✅ Existe | pipeline 13F |
| Sentiment Score | 🧪 En prueba | Agente_Sentimiento |
| Telegram propuesta + botones | ✅ Existe (parcial) | Class_DashBot.py |
| IB place_order | ✅ Existe | IBroks_Client |
| Binance create_order | ✅ Existe | Class_vehiculo.py |
| Claude API key configurada | ✅ Existe | sesion ClaudeAPIP / ClaudeAPIC |
| Agente_LtvControl (crypto) | ✅ Existe | Class_AgentManager.py |
| Preservation (stops automáticos) | ✅ Existe | Class_DashBot.py |
| `agente_ia` en llave_privada | ✅ Configurado | sesion vehiculo=Stock |
| `Agente_ClaudeIA` @24h | ✅ Implementado | Class_DashBot.py |
| `_armar_contexto_ia()` (gain_usd, monto_sugerido) | ✅ Implementado | Class_DashBot.py |
| `_claude_ia_eval()` (prompt diferenciado BUY/GAINS) | ✅ Implementado | Class_DashBot.py |
| `ia_trace` tabla BD | ✅ Creada | MySQL bdinv |
| `IaTraceScreen` (insert/select/trace/mejoras) | ✅ Implementado | Modulos_Mysql.py |
| Panel IA Trace (auto-refresh + timer) | ✅ Implementado | Class_SystemStatus.py |
| Chat Asistente con memoria persistente | ✅ Implementado | Class_DashBot.py → Chatbot |
| Script forzar ejecución manual | ✅ Creado | AppTest/run_agente_claudeia.py |
| `ia_mejoras` tabla BD | ✅ Creada | MySQL bdinv |
| Sub-tab "Mejoras" en IA Trace | ❌ Pendiente Fase 3 | Class_SystemStatus.py |
| Telegram propuesta BUY/SELL del agente | ❌ Pendiente Fase 2 | Class_DashBot.py |
| Ejecución automática de órdenes | ❌ Pendiente Fase 3 | Class_DashBot.py |

---

## Configuración por Vehículo (sesion.parameters JSON)

> Corrección 2026-08-19: la columna real es `sesion.parameters` (no `llave_privada`, nombre
> desactualizado en versiones previas de este doc). `BDsystem.get_sesion_by_vehiculo()` /
> `update_sesion_parameters()` (Modulos_Mysql.py) son el único punto de lectura/escritura.

### Stock (U4214563)
```json
{
  "preservation": {
    "atr_mult": 2.0,
    "roi_minimo": 0.1,
    "correccion_pct": 0.08
  },
  "gains_capture": {
    "min_roi": 0.20,
    "min_ganancia": 100.0
  },
  "agente_ia": {
    "activo": false,
    "modo": "autorizado",
    "monto_por_trade": 170,
    "deuda_max_pct": 35,
    "leverage_max": 1.8,
    "risk_real_max": 2.0,
    "gate_consenso_min": 4,
    "gate_inst_score_min": 0.5,
    "concentracion_sector_max": 30,
    "concentracion_region_max": 40,
    "plan": {
      "meta_capital": "20.0M USD",
      "meta_año": "2035",
      "ingreso_pasivo_pct": "≥4%",
      "perdida_max_pct": "20%",
      "mision": "En crisis → Hold o sumar posiciones, nunca vender por pánico."
    }
  }
}
```

`agente_ia.plan` (Misión IA) y las 5 `Restricciones de cartera` (`leverage_max`, `deuda_max_pct`,
`risk_real_max`, `concentracion_sector_max`, `concentracion_region_max`) se editan desde
**Gestión → Editar Plan** (`Class_gestion.py::edit_plan()`) — es el único punto de escritura de
este blob desde la UI. `_load_ia_plan()` es el único punto de lectura para poblar ese formulario.

**Un único plan por cuenta (2026-08-19):** `plan` y `restricciones` son compartidos por los 3
vehículos de trading — no tiene sentido que Stock y Crypto persigan metas distintas. Al guardar,
`edit_plan()` escribe en Stock (fuente canónica leída por `_load_ia_plan()`) y
`_sync_plan_restricciones()` replica `agente_ia.plan` + las 5 restricciones a Crypto y BotCrypto,
dejando intacto el resto de cada `agente_ia` (`activo`, `modo`, `monto_por_trade`, `gate_*`, que sí
son propios de cada vehículo — ritmo y gates de entrada pueden diferir aunque la meta sea la misma).

**Activar agente:**
```sql
UPDATE sesion
SET parameters = JSON_SET(parameters, '$.agente_ia.activo', true)
WHERE vehiculo = 'Stock';
```

---

## Plan de Inversión — tablas `plan` y `variablesplan`

Persistencia relacional (MySQL, fuera del blob JSON de `sesion.parameters`) para el resto del
panel Gestión → Plan. Editable desde `Class_gestion.py`: `edit_plan()` (objetivo/visión) y
`_edit_riesgos()` (riesgos y variables cualitativas).

### `plan`
Una fila por `idcuenta` × `vision`. `vision` toma valores como `deseada` (capital objetivo,
estilo de vida, contribución) y las demás visiones narrativas mostradas en "Objetivo del Plan"
(columna `proyecto`, texto libre con soporte de placeholder `{ingreso_pasivo}` resuelto en runtime
por `_resolver_objetivo_texto()` contra `agente_ia.plan.ingreso_pasivo_pct`).

| Acceso | Método (Modulos_Mysql.py) |
|---|---|
| Leer | `select_plan(idcuenta)` |
| Actualizar texto narrativo | `update_plan_proyecto(id, proyecto)` |
| Actualizar visión "deseada" (capital/estilo de vida/contribución) | `update_plan_inversion(idcuenta, vision, values)` — dispara recálculo de `trazaplan` (tasa adaptativa, ver [[sesion-2026-08-15-trazaplan-horizonte-fix]] en memoria) |

### `variablesplan`
Filas cualitativas por `idcuenta` × `tipo`, editadas en el panel "Riesgos" (`_edit_riesgos()`):

| `tipo` | Sección UI | Campos usados |
|---|---|---|
| `riesgos` | Riesgos / Solución Potencial | `ditem` (riesgo), `observaciones` (mitigación) |
| `esfuerzo` | Tiempo / Energía | `ditem` |
| `economico` | Económicos (costo del plan: recorte de gastos, % ingresos invertido) | `ditem` |
| `personal` | Personales (qué se sacrifica: viajes, salidas) | `ditem` |
| `salida_emergencia` | Criterios de salida de emergencia | `ditem` (usado en `edit_plan()`, no en `_edit_riesgos()`) |

| Acceso | Método |
|---|---|
| Leer todo el vehículo | `select_variablesplan(idcuenta)` |
| Insertar item | `insert_variablesplan_item(idcuenta, tipo, ditem, observaciones)` |
| Editar item | `update_variablesplan_item(id, ditem, observaciones)` |
| Borrar item | `delete_variablesplan_item(id)` |

**Hoy son puramente informativas** — ningún agente las lee. Ver idea de integración con modo
autónomo más abajo.

---

## Observación en vivo — DataHub como nodo de integración (2026-08-19)

Screenshot del panel `DataHub - Símbolos` (System → DataHub) mostrando un símbolo (BNBUSDT) con
`activos`, `datos_tecnicos`, `costobase`, `_vehiculo`, `account` cargados en vivo — el usuario
señaló que este es el punto donde confluyen las pestañas `BuySell`, `Rebalanceo`, `Sell IA`,
`Buy IA`, `Symbol Events` y `Alertas`: todas leen del mismo `DataHub.info[symbol]`.

Primera vez que se confirma en vivo (no solo en el diagrama de capas de este doc) que el dato fluye
Capa 2 (Datos) → Capa 3 (Señales) → Capa 4 (Decisión BUY/SELL IA) → Capa 5 (Symbol Events/Alertas
como trazabilidad) sobre el mismo objeto, para el mismo símbolo, en tiempo real.

**Abre pregunta para próxima sesión** (usuario: "lanzando ideas para discusión") — ver
[[debate-2026-08-19-integracion-tabs-datahub]] en `30-Gestion/debates/`: si Symbol
Events/Alertas es el lugar natural para enganchar el gate de `salida_emergencia` (idea de la
sección siguiente) y qué tan lejos está hoy `Sell IA`/`Buy IA` de una decisión autónoma real vs.
solo mostrar candidatos.

**Primer caso concreto ya resuelto (2026-08-19, mismo día):** al revisar el panel superior de KPIs
(`Total dGyP`, `Leverage (D/K %)`, `Deuda Total`, `UnP&L vs Capital`, `Beta Portafolio`,
`Dividendos YTD` en `DashMain.py`), el usuario notó que esos topes de barra podían estar en duro
sin relación con el plan. Confirmado: `Total dGyP`/`Total Inversión` ya leían su tope de
`trazaplan` (correcto), pero `Leverage (D/K %)` (tope `30.0` fijo) y `Deuda Total` (tope =
`total_debitmax` del broker, no la restricción del usuario) ignoraban `agente_ia.deuda_max_pct` ya
definido en Restricciones de cartera. Se conectaron ambas barras a `deuda_max_pct` vía
`DashMain._get_deuda_max_pct()` (lee `sesion.parameters` de Stock, default 35).
`leverage_max` (múltiplo, ej. "1.8x", logueado en `Class_DashBot.py`) y `risk_real_max` (sin
consumidor en todo el código hoy) quedaron sin conectar — unidades ambiguas/incompatibles con las
barras existentes, no se fuerza una conversión sin confirmar el significado real. `UnP&L vs
Capital` se dejó en su ±20% fijo actual por la misma razón.

---

## Idea — uso de `variablesplan` en modo autónomo (sin fecha, sesión 2026-08-19)

Hoy `riesgos` / `esfuerzo` / `economico` / `personal` / `salida_emergencia` son texto libre que el
usuario llena para sí mismo — ningún agente los consulta. Encajan naturalmente en **Fase 4 —
"Autonomía supervisada"** (arriba) como la capa de **límites declarados por el usuario** que el
agente debe respetar antes de operar sin supervisión:

- **`salida_emergencia`** es el candidato más directo: son criterios explícitos ("qué me haría
  vender todo") que hoy solo existen como texto para humanos. Podrían mapearse a un chequeo del
  agente antes de cada ciclo — si alguna condición matchea (ej. vía un segundo prompt a Claude que
  evalúe "¿aplica alguno de estos criterios al estado actual del portfolio/mercado?"), fuerza
  `HOLD`/`ALERTA` y bloquea nuevas compras hasta revisión humana. Es el mismo espíritu que el gate
  BEAR de BotCrypto o el invariante de Preservation ("stop final ≥ stop calculado") — una regla
  dura que ni el agente puede saltarse.
- **`riesgos` + `observaciones` (Solución Potencial)** podrían alimentar el contexto del prompt
  diario de `Agente_ClaudeIA` como recordatorio de qué escenarios ya fueron pensados y cuál es la
  respuesta planeada — reduce que el agente improvise una reacción distinta a la que el usuario ya
  decidió en frío.
- **`economico`/`personal`** son más restricciones de *capital disponible* que de *decisión de
  mercado* — encajan mejor como input al "Budget diario de capital" de Fase 3 que como gate de
  decisión (ej. "reducir 50% gastos mensuales" ⇒ ajusta `monto_por_trade` o el ritmo de aportes,
  no la elección de símbolo).
- **`esfuerzo`** (tiempo/energía dedicado) es la variable que más directamente justifica la
  automatización en sí — no es un gate, es el motivo del proyecto: cuanto más automatizado el
  seguimiento, menos horas/semana reclama.

No es una tarea para implementar ahora — es la pieza que le falta a Fase 4 para que "Autonomía
supervisada" tenga límites declarados por el usuario y no solo gates técnicos (leverage, inst_score).
Retomar cuando se aborde Fase 3/4 del roadmap de este documento.

---

## Tablas BD

### `ia_trace`
```sql
CREATE TABLE ia_trace (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    timestamp   DATETIME         DEFAULT CURRENT_TIMESTAMP,
    vehiculo    VARCHAR(20)      NOT NULL,
    simbolo     VARCHAR(20),
    decision    ENUM('BUY','SELL','HOLD','ALERTA') NOT NULL,
    monto       DECIMAL(12,2)    DEFAULT 0,
    motivo      TEXT,
    gates_ok    JSON,
    estado      ENUM('PENDIENTE','APROBADO','IGNORADO','EJECUTADO','FALLIDO') DEFAULT 'PENDIENTE',
    telegram_id VARCHAR(50)
);
```

### `ia_mejoras`
```sql
CREATE TABLE ia_mejoras (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    timestamp   DATETIME         DEFAULT CURRENT_TIMESTAMP,
    categoria   ENUM('agente','datos','proceso','decision','ui') NOT NULL,
    titulo      VARCHAR(200)     NOT NULL,
    descripcion TEXT,
    impacto     ENUM('alto','medio','bajo') DEFAULT 'medio',
    estado      ENUM('pendiente','en_revision','adoptado','descartado') DEFAULT 'pendiente',
    origen      VARCHAR(100)
);
```

---

## Fases de Implementación

```
FASE 1 — "Ojos"  ✅ COMPLETA (2026-06-15)
  ├── Agente_ClaudeIA @24h — observa y registra
  ├── Contexto: portfolio (gain_usd, [GAINS?]) + candidatos (monto_sugerido)
  ├── Panel IA Trace — auto-refresh 60s + timer próximo ciclo
  ├── Chat Asistente — memoria persistente entre sesiones
  └── Script: AppTest/run_agente_claudeia.py — forzar primera ejecución

FASE 2 — "Voz"  ❌ pendiente
  ├── Telegram: propuesta BUY/SELL con botones ✅/❌
  ├── handle_callback: APROBADO → insert order_trader + ia_trace=APROBADO
  └── handle_callback: IGNORADO → ia_trace=IGNORADO

FASE 3 — "Manos"  ❌ pendiente
  ├── Gate extra: consenso=UNANIME + inst_score > 0.7
  ├── modo=automatico → ejecuta directo + notifica
  ├── Budget diario de capital
  └── Sub-tab "Mejoras IA" en panel

FASE 4 — "Autonomía supervisada"  ❌ sin fecha
  ├── Horario operativo L-V 10:00-15:00
  ├── Reporte diario Telegram al cierre
  └── Re-entry Scanner integrado (Backlog #56)
```

---

## Plan de trabajo — Fase 2 (próxima)

| Paso | Qué hacer | Archivo | Riesgo |
|---|---|---|---|
| 1 | `_enviar_propuesta_telegram()` — mensaje con monto + motivo + botones | Class_DashBot.py | Medio |
| 2 | `handle_callback("ia_aprobar_XXX")` — busca en ia_trace, llama place_order | Class_DashBot.py | Alto |
| 3 | `handle_callback("ia_ignorar_XXX")` — marca IGNORADO en ia_trace | Class_DashBot.py | Bajo |
| 4 | Actualiza ia_trace estado → APROBADO / EJECUTADO / FALLIDO | Class_DashBot.py | Bajo |
| 5 | Panel IA Trace muestra estado actualizado tras aprobación | Class_SystemStatus.py | Bajo |

**Prerequisito Fase 2:** validar 5-10 días de decisiones en ia_trace.
¿Las decisiones de Claude tienen sentido con la misión?
