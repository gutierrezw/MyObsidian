---
name: ref-dividendos
description: Campos Yahoo Finance para dividendos — referencia rápida de campos, validaciones y pendiente migración IB.
metadata:
  tipo: ref
  modulo: dividendos
  version: 1.0
  fecha: 2026-06-29
  status: activo
---

# Referencia: Campos de Dividendos (Yahoo Finance)

> Extraído de ref-regla-buy (v1). Ver contexto de negocio en [[ref-oportunidades]].

---

## Campos yfinance

| Campo | Descripción | Usar |
|-------|-------------|------|
| `dividendRate` | Pago individual (NO anual) — ej: trimestral $0.50 = $2/año | ❌ No usar para cálculo anual |
| `trailingAnnualDividendRate` | TTM — suma últimos 12 meses | ✅ Dividendo anual real |
| `dividendYield` | Rendimiento % | ✅ Para filtros de yield |
| `trailingAnnualDividendYield` | Yield basado en TTM | ✅ Más confiable que `dividendYield` |
| `exDividendDate` | Fecha ex-dividendo | ✅ Para validar pagos próximos |
| `dividendDate` | Fecha de pago | ℹ️ Informativo |
| `payoutRatio` | Dividendos / Ganancias | ✅ Sostenibilidad |
| `fiveYearAvgDividendYield` | Promedio 5 años | ✅ Comparar vs actual |

---

## Validaciones en `dividends_en_market_stock()`

- **Frescura**: último pago > 18 meses atrás → advertencia (yfinance puede retener históricos de activos que dejaron de pagar)
- **Período**: últimos 12 meses móviles (TTM), no año calendario
- **Frecuencia detectada**: mensual / trimestral / semestral / anual según conteo de pagos TTM

---

## Pendiente — migración a IB API

Campos disponibles vía WebSocket IB:

| Campo IB | Descripción |
|----------|-------------|
| `"7286"` | Dividendo actual |
| `"7287"` | Yield % |
| `"7288"` | Fecha ex-dividendo |
| `"7672"` | TTM |
| `"7671"` | Próximo pago |
