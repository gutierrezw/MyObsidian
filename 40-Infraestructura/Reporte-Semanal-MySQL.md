---
fecha: 2026-06-28
tipo: infraestructura
estado: activo
tags: [infraestructura, mysql, monitoreo, task-scheduler]
---

# Reporte Semanal MySQL — salud del schema

## Qué es

Reporte semanal de salud del schema MySQL (`bdinv`): tamaño de tablas, índices sin uso, queries con full scan y uso del buffer pool. Se genera como documento HTML local — **no envía email**, queda para revisión manual.

## Por qué corre local (no Cowork cloud)

El sandbox de Cowork es un contenedor Linux aislado en la nube, sin ruta de red hacia este PC — confirmado con `curl` devolviendo "Connection refused" contra `localhost:8050`. Por eso la tarea programada original de Cowork (`mysql-weekly-report`) fue desactivada (`enabled: false`) y reemplazada por Task Scheduler local, igual patrón que [[Backup-Diario-MySQL|Wildaga Backup]].

## Ubicación del script

`C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report\report.js`

## Qué hace

1. Conecta **directo** a MySQL (`mysql2/promise`) usando credenciales de `Documents\Claude-Cowork-Scripts\mysql_config.json` (bloque `db`) — no depende de que `server-api` esté corriendo.
2. Construye HTML con: tablas por tamaño, índices sin uso, queries con full scan, buffer pool.
3. Guarda la salida en `.\output\reporte-YYYYMMDD.html` y `.\output\reporte_ultimo.html`.
4. Si algo falla, escribe el error en `.\output\` y registra log en `.\logs\`.

## Endpoint relacionado en server-api

`GET /db/diagnostics` en `routes/db.js` — mismas queries de solo lectura, no usa `ALLOWED_TABLES`. Queda disponible para consultas remotas vía HTTP si se necesita en el futuro, pero `report.js` **no lo usa** (va directo a MySQL porque corren en el mismo equipo).

## Programación (Task Scheduler)

| Campo | Valor |
|---|---|
| Nombre tarea | `Reporte Semanal MySQL` |
| Frecuencia | Semanal, lunes |
| Hora | 08:00 |
| Comando | `node C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report\report.js` |
| Logon type | Solo interactivo |

## Comandos útiles

```powershell
# Probar manualmente
cd C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report
node report.js

# Ver logs
type C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report\logs\report-YYYYMMDD.log

# Ver estado de la tarea
schtasks /query /tn "Reporte Semanal MySQL" /fo LIST /v
```

## Historial

- 2026-06-28: reemplazo de la tarea Cowork (rota por aislamiento de red) por script local + Task Scheduler. Verificado: `REPORTE_OK — full_scans:10 sin_uso:21 buffer:0.48GB/2.00GB`.

## Referencias

- [[Backup-Diario-MySQL]]
- [[ref-instalacion]]
