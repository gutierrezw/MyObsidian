---
fecha: 2026-06-28
tipo: infraestructura
estado: activo
tags: [infraestructura, mysql, backup, task-scheduler]
---

# Backup Diario MySQL — "Wildaga Backup"

## Qué es

Respaldo diario automático de la base MySQL local (`bdinv`), más carpetas de ebooks e inversiones, hacia Google Drive.

## Ubicación del script

`C:\Users\InversionesWildaga\Documents\backup_script\backup_diario.bat`

## Qué respalda

- **Base de datos:** MySQL local, base `bdinv`, usuario `root`, dump completo (schema + data) → `dumps\bdinv_YYYYMMDD.sql`
- **Ebooks:** carpeta de libros electrónicos
- **Inversiones:** carpeta de inversiones

## Destino

Google Drive: `G:\Mi unidad\Backup\`
- `dumps\YYYYMMDD\`
- `ebook\YYYYMMDD\`
- `Inversiones\YYYYMMDD\`

## Retención

15 días — rotación automática de carpetas antiguas.

## Programación (Task Scheduler)

| Campo | Valor |
|---|---|
| Nombre tarea | `Wildaga Backup` |
| Frecuencia | Diaria |
| Hora | 07:00 |
| Usuario | InversionesWildaga |
| Logon type | Solo interactivo |

## Comandos útiles

```powershell
# Ver estado de la tarea
schtasks /query /tn "Wildaga Backup" /fo LIST /v

# Forzar ejecución manual
schtasks /run /tn "Wildaga Backup"

# Ver log del día
type C:\Users\InversionesWildaga\Documents\backup_YYYYMMDD.log
```

## Pendiente

Activar sección MCP del script cuando se configuren MCPs en VS Code:
- Descomentar sección MCP en `backup_diario.bat`
- Ruta config: `%APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings`
- Destino: `G:\Mi unidad\Backup\VSCode-MCP\YYYYMMDD\`

## Referencias

- [[Reporte-Semanal-MySQL]]
- [[ref-instalacion]]
