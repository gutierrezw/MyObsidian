# Spec: Guía de estilo UI — AppOO

## Objetivo

Reglas de estilo visual obligatorias para **cualquier ventana/popup nuevo** que se agregue a AppOO
(Tkinter). No es un rediseño de la app existente — es el estándar a partir del cual se construye
todo lo nuevo, y el criterio para ir migrando lo viejo cuando se toque por otro motivo.

**Origen:** 2026-08-15, a partir de comparar 3 pantallas ya existentes (Setup-Inversionista/Editar
Vehículo, Diversificación vs pago Dividendos, Editar Plan) que usaban 3 fuentes y 3 estilos de botón
distintos. Ver [[ref-datahub]] para la estructura completa de `DataHub.colors`/`DataHub.cchart`
(paleta de colores, cargada desde BD, sesión `DataHub`).

## Regla 1 — Fuente

- **Formularios/labels/popups:** `("Segoe UI", 9)` — labels; `("Segoe UI", 9, "bold")` para headers
  de sección (barras tipo "Visión Deseada"/"Misión IA" en Editar Plan).
- **Grids/Treeview:** se mantiene `("Courier", 8)` (definido en `style_app()`,
  `Modulos_Utilitarios.py`) — el monoespaciado es necesario para alinear columnas numéricas.
- Eliminadas: `("Arial", 8)` (Diversificación) y el font default de Tk sin especificar
  (Editar Vehículo) — ninguna ventana nueva debe omitir `font=`.

## Regla 2 — Botón: `Flat.TButton`

Nueva clase en `style_app()` (`Modulos_Utilitarios.py`), tomada del botón "Cancel" del popup
Diversificación (el que le gustó al usuario como referencia):

```python
style.configure("Flat.TButton", background="#444444", foreground="white",
                font=("Segoe UI", 8), relief="flat", padding=4)
style.map("Flat.TButton", background=[("active", "#606060"), ("disabled", "#2a2a2a")])
```

Uso: `ttk.Button(parent, text="Guardar", style="Flat.TButton", command=...)`. Reemplaza:
- `tk.Button(bg="gray", fg="white", ...)` (Editar Plan)
- `tk.Button(...)` sin estilo, gris de sistema (Editar Vehículo)
- el `tk.Button(bg="#444444", ...)` hardcodeado repetido en cada popup (Diversificación)

## Regla 3 — Entry / Text

`bg=<color de fondo del contenedor>, fg="white", insertbackground="white"` — siempre explícito,
nunca dejar el Entry en blanco/negro default de Tk (pasaba en Editar Vehículo). El color de fondo
del Entry sigue el que ya usa cada ventana para sus campos (`self.bgcolor`/`self.colors["bgcolor"]`
= DarkCyan en Editar Plan y Editar Vehículo) — no se fuerza un color nuevo, solo que esté siempre
declarado.

## Regla 4 — Colores: una sola fuente, `DataHub.colors`/`cchart`

No crear una paleta nueva. `bgcolor`/`cgcolor`/`cchart` (ver [[ref-datahub]]) ya son la fuente de
verdad y ya se usan en las 3 pantallas — el problema no era la falta de un sistema de color, era que
fuente/botón/entry se resolvían a mano en cada popup en vez de apoyarse en `style_app()`.

## Regla 5 — "Sesión" (barra de sección de ancho completo): `ui_section_bar()`

Toda ventana que abre desde **Setup - Inversionista** organiza su contenido en "sesiones" — bloques
temáticos con un título propio (ej. "Restricciones de Cartera", "Parámetros de Trading"). Cada
título de sesión debe usar la misma barra de ancho completo, extraída del patrón que ya existía en
Editar Plan y que le pareció sobrio al usuario:

```python
def ui_section_bar(parent, text, bg="#37474f", row=0, column=0, columnspan=2,
                    font=("Segoe UI", 9, "bold"), pady=(8, 4)):
    ...
```

Definida en `Modulos_Utilitarios.py`, se importa con `from Modulos_Utilitarios import ui_section_bar`.

**El color NO se hardcodea en cada call** (viola Regla 4). Se toma de `DataHub.colors["session"]`,
un grupo nuevo dentro de la misma estructura `colors` que ya carga `bgcolor`/`cgcolor`/`cchart`
desde la sesión BD `DataHub` (`Class_customer.py`, ver [[ref-datahub]]):

```python
session_colors = {"neutral": "#37474f", "ia": "#1565c0", "danger": "#c0392b", "nested": "#546e7a"}
session_colors.update(envs_config.get("session_colors", {}))  # override opcional desde BD
colors["session"] = session_colors
```

Uso: `ui_section_bar(parent, "Parámetros de Trading", bg=self.colors["session"]["neutral"], row=row, columnspan=2)`.
`load_from_database()` también sincroniza `DataHub.colors["session"]` en caliente si `envs_config`
trae la clave `session_colors` (mismo patrón que `bgcolor`/`cgcolor`/`cchart`).

**Paleta semántica de color por tipo de sesión** (default; editable a futuro desde BD igual que el resto de `DataHub.colors`):

| Key | Hex default | Uso |
|---|---|---|
| `neutral` | `#37474f` (teal oscuro) | Sesión neutral / técnica (default) |
| `ia` | `#1565c0` (azul) | Sesión relacionada a IA |
| `danger` | `#c0392b` (rojo) | Sesión de peligro / emergencia |
| `nested` | `#546e7a` (gris-azulado claro) | Subsección anidada (un nivel debajo de una sesión) |

## Estado de la migración

| Ventana | Fuente | Botón | Entry | Sesiones (`ui_section_bar`) | Estado |
|---|---|---|---|---|---|
| Editar Plan (`Class_gestion.py::edit_plan`) | Segoe UI 9 | `Flat.TButton` | bg=bgcolor | ✅ 4 sesiones | ✅ migrado 2026-08-15 |
| Editar Vehículo (`DashMain.py`) | Segoe UI 9 | `Flat.TButton` | bg=bgcolor | n/a (sin sesiones) | ✅ migrado 2026-08-15 |
| Diversificación vs Dividendos (`DashMain.py::detalle_graph`) | Segoe UI 8 (botón) | `Flat.TButton` | n/a | n/a | ✅ botón migrado 2026-08-15 |
| Variables de Entorno / "Envs" (`DashMain.py`) | Segoe UI 9 | n/a (sin botones a migrar) | bg=bgcolor | ✅ 4 sesiones + 1 subsección | ✅ migrado 2026-08-15 |
| Resto de la app (Screener, Consenso, popups de órdenes, Setup general, etc.) | mixto | mixto | mixto | — | ⏳ backlog, ver `BACKLOG.md` |

## Actualización 2026-08-15 (tarde)

Usuario pidió explícitamente extender el alcance del día: *"si quiero que todas las ventanas que
nacen desde Setup Inversionista queden con el mismo estilo... cada título de 2da pantalla es una
sesión de la ventana... algo así se puede replicar para las sesiones de la ventana Variables de
Entorno"*. Se agregó Regla 5 (`ui_section_bar()`) y se migró la 4ta ventana (Variables de Entorno)
completa, incluyendo Entry/Label a Segoe UI 9 + bg/fg explícitos (16 Entries).

## Pendiente (fuera de alcance de hoy)

- Migrar el resto de ventanas/popups de la app a `Flat.TButton` + Segoe UI 9 + Entry con bg/fg
  explícitos, una por una, cuando se toquen por otro motivo (no un sprint de refactor dedicado).
- Evaluar si conviene mover `font_base`/`btn`/`entry` al JSON de `DataHub.colors` en BD (hoy son
  constantes en `style_app()`) para que sea 100% configurable sin tocar código — no se hizo hoy
  porque implica un `UPDATE` sobre la sesión `DataHub` en producción, fuera del alcance decidido
  ("hoy no vamos a cambiar toda la app").
