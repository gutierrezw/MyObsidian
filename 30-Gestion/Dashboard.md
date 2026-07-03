# Dashboard — AppOO

## Sesiones recientes

```dataview
TABLE fecha, file.size AS "Tamaño"
FROM "30-Gestion/Sesiones"
SORT fecha DESC
LIMIT 10
```

## Decisiones técnicas abiertas

```dataview
TABLE fecha, file.link AS "Decisión"
FROM "20-Proyecto"
WHERE contains(tags, "decision") AND estado = "pendiente"
SORT fecha DESC
```

## Memoria — por tipo

```dataview
TABLE description AS "Descripción"
FROM "10-Memoria"
WHERE metadata.type = "project"
SORT file.mtime DESC
LIMIT 15
```

## Feedbacks registrados

```dataview
TABLE description AS "Regla"
FROM "10-Memoria"
WHERE metadata.type = "feedback"
SORT file.mtime DESC
```

## Infraestructura — procesos de mantenimiento

```dataview
TABLE estado, fecha AS "Última revisión"
FROM "40-Infraestructura"
SORT fecha DESC
```
