# Documentación: Class_IA_modelos.py

## Descripción General
Módulo que implementa modelos de Machine Learning (RandomForestClassifier) para identificar oportunidades de inversión. Contiene dos clases principales:
- `ModeloOportunidadesSell`: Detecta oportunidades de venta
- `ModeloOportunidadesBuy`: Detecta oportunidades de compra

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                    Class_IA_modelos.py                          │
├─────────────────────────────────────────────────────────────────┤
│  DEFAULT_PARAMS_SELL          DEFAULT_PARAMS_BUY                │
│         │                            │                          │
│         ▼                            ▼                          │
│  ┌──────────────────┐      ┌──────────────────┐                │
│  │ModeloOportunida- │      │ModeloOportunida- │                │
│  │   desSell        │      │   desBuy         │                │
│  ├──────────────────┤      ├──────────────────┤                │
│  │ - modelo         │      │ - modelo         │                │
│  │ - metrics        │      │ - metrics        │                │
│  │ - params         │      │ - params         │                │
│  │ - ReOportunidades│      │ - ReOportunidades│                │
│  └──────────────────┘      └──────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Parámetros por Defecto

Los parámetros del algoritmo son idénticos para ambos modelos. La única diferencia es el nombre del umbral de acción (`umbral_sell` vs `umbral_buy`) que refleja la decisión de cada modelo.

### Parámetros comunes (Sell y Buy)
```python
{
    "n_estimators": 100,        # Número de árboles en el bosque
    "max_depth": 10,            # Profundidad máxima de cada árbol
    "min_samples_split": 5,     # Mínimo de muestras para dividir nodo
    "random_state": 42,         # Semilla para reproducibilidad
    "n_folds": 5,               # Folds para validación cruzada
    "test_size": 0.3,           # Proporción de datos para test
    "umbral_observacion": 0.35  # Por debajo se descarta
}
```

### Umbral específico por modelo
| Modelo | Clave BD | Valor defecto | Significado |
|--------|----------|---------------|-------------|
| Sell | `umbral_sell` | 0.65 | Confianza mínima para recomendar venta |
| Buy  | `umbral_buy`  | 0.65 | Confianza mínima para recomendar compra |

> **Nota:** Los umbrales se leen desde la tabla `modelos_ia` en BD. Si no existen, se usan los valores por defecto del código. Cada modelo debe leer su propia clave — Buy lee `umbral_buy`, Sell lee `umbral_sell`.

---

## Clase: ModeloOportunidadesSell

### Propósito
Identificar activos que deben venderse basándose en indicadores técnicos y métricas de rentabilidad.

### Atributos
| Atributo | Tipo | Descripción |
|----------|------|-------------|
| `modelo` | RandomForestClassifier | Modelo entrenado |
| `metrics` | dict | Métricas de rendimiento |
| `modelo_name` | str | "modelo_sellv01" |
| `params` | dict | Parámetros de configuración |
| `ReOportunidades` | RepositorioOportunidadesBuySell | Acceso a BD |

### Métodos

#### `__init__(self)`
Inicializa el modelo, carga parámetros desde BD si existen.

#### `_cargar_params_bd(self)`
Carga parámetros personalizados desde tabla `modelos_ia`.

#### `convertir_dataset_entrenamiento(self, datos) -> pd.DataFrame`
Convierte datos de BD a DataFrame para entrenamiento.
- Extrae `indicadores`, `profit`, `roi` del campo `json_detalle`
- Elimina columnas innecesarias (id, tipo, timestamp, etc.)

#### `aplanar_datos_tecnicos(self, df) -> pd.DataFrame`
Extrae indicadores de la columna `datos_tecnicos` creando columnas individuales:
- Por timeframe: `_d` (diario), `_w` (semanal), `_m` (mensual)
- Indicadores: RSI, MACD, Close, EMAs, Fibonacci, Máximos/Mínimos

#### `cargar_datos(self, df, usar_timeframes=["d"], umbral_columnas=0.5) -> (X, y)`
Prepara features y etiquetas para entrenamiento.
- Filtra columnas con demasiados NaN
- Genera etiquetas: `sell` (1) o `hold` (-1)

#### `entrenar_modelo(self, df)`
Entrena usando validación cruzada estratificada (StratifiedKFold).
- Calcula métricas: accuracy, precision, recall, F1, ROC-AUC
- Usa `class_weight="balanced"` para clases desbalanceadas

#### `metricas_de_clases(self, X, y, y_test, X_test)`
Calcula métricas adicionales en conjunto de test.

#### `calcular_feature_importance(self)`
Extrae importancia de las top 10 features.

#### `predecir_modelo(self, df) -> pd.DataFrame`
Realiza predicciones y clasifica:
- `sell`: confianza >= umbral_sell
- `observacion`: confianza >= umbral_observacion
- `descartar`: confianza < umbral_observacion

#### `save_modelo(self, file="modelo_ia")`
Guarda modelo y métricas como `.pkl` usando `define_FileCache`.

#### `load_modelo(self, file="modelo_ia")`
Carga modelo y métricas desde archivos `.pkl`.

#### `run_entraminetoSell(self)`
Ejecuta pipeline completo de entrenamiento para Sell.

---

## Clase: ModeloOportunidadesBuy

### Propósito
Identificar activos con precio deprimido que representan oportunidades de compra.

### Atributos
| Atributo | Tipo | Descripción |
|----------|------|-------------|
| `modelo` | RandomForestClassifier | Modelo entrenado |
| `metrics` | dict | Métricas de rendimiento |
| `modelo_name` | str | "modelo_buyv01" |
| `params` | dict | Parámetros de configuración |
| `ReOportunidades` | RepositorioOportunidadesBuySell | Acceso a BD |

### Métodos
Misma estructura que `ModeloOportunidadesSell` con diferencias en:

#### `convertir_dataset_entrenamiento(self, datos)`
Extrae campos específicos de Buy:
- `ganancia_precio`: % diferencia entre precio actual y objetivo
- `ganancia_inversion`: Ganancia potencial
- `dividend_yield`: Rendimiento por dividendo
- `score`: Score de rebalanceo

#### `cargar_datos(self, df, usar_timeframes=["d"], umbral_columnas=0.5)`
Features específicas para Buy:
```python
scalar_features = ["ganancia_precio", "ganancia_inversion", "dividend_yield", "score"]
```
Etiquetas: `buy` (1) o `wait` (-1)

#### `predecir_modelo(self, df)`
Clasifica usando umbrales:
- `buy`: confianza >= umbral_buy
- `observacion`: confianza >= umbral_observacion
- `descartar`: confianza < umbral_observacion

#### `run_entrenamientoBuy(self)`
Ejecuta pipeline completo de entrenamiento para Buy.

---

## Features Utilizadas

### Indicadores Técnicos (por timeframe)
| Feature | Descripción | Sufijos |
|---------|-------------|---------|
| `rsi` | Relative Strength Index (0-100) | `_d`, `_w`, `_m` |
| `macd` | Moving Average Convergence Divergence | `_d`, `_w`, `_m` |
| `Close` | Precio de cierre | `_d`, `_w`, `_m` |

### EMAs (Medias Móviles Exponenciales)
| EMAs Largos | EMAs Cortos |
|-------------|-------------|
| EMA020 | EMA009 |
| EMA050 | EMA021 |
| EMA100 | EMA055 |
| EMA200 | EMA144 |

### Otros Indicadores
| Feature | Descripción |
|---------|-------------|
| `fibo_longico` | Nivel de retroceso Fibonacci |
| `13_semanas_max/min` | Máx/Mín últimas 13 semanas (diario) |
| `26_semanas_max/min` | Máx/Mín últimas 26 semanas (semanal) |
| `52_semanas_max/min` | Máx/Mín últimas 52 semanas (mensual) |

### Features Escalares (Solo Buy)
| Feature | Descripción |
|---------|-------------|
| `ganancia_precio` | % diferencia precio actual vs objetivo |
| `ganancia_inversion` | Ganancia potencial de inversión |
| `dividend_yield` | Rendimiento por dividendo |
| `score` | Score de rebalanceo calculado |

### Features Escalares (Solo Sell)
| Feature | Descripción |
|---------|-------------|
| `roi` | Retorno sobre inversión (%) |
| `profit` | Ganancia/Pérdida actual |

---

## Métricas Almacenadas

```python
metrics = {
    # Validación Cruzada
    "precision": float,      # Media de precisión
    "precision_std": float,  # Desviación estándar
    "recall": float,
    "recall_std": float,
    "f1_score": float,
    "f1_std": float,
    "accuracy": float,
    "accuracy_std": float,
    "roc_auc": float,
    "roc_auc_std": float,

    # Información del Dataset
    "n_folds": int,
    "n_samples": int,
    "n_aprobadas": int,      # sell/buy
    "n_rechazadas": int,     # hold/wait
    "balance_ratio": float,

    # Parámetros Usados
    "params": dict,

    # Métricas de Test
    "test_precision": float,
    "test_recall": float,
    "test_f1": float,
    "test_accuracy": float,
    "f1_macro": float,
    "test_roc_auc": float,

    # Feature Importance
    "feature_importance": [
        {"feature": str, "importance": float},
        ...
    ]
}
```

---

## Ciclo de Entrenamiento

```
┌─────────────────────────────────────────────────────────────────┐
│                    CICLO DE ENTRENAMIENTO                       │
└─────────────────────────────────────────────────────────────────┘

    BD: tabla oportunidades (tipo='buy' o 'sell')
                          │
                          ▼
         RepositorioOportunidadesBuySell.obtener_por_tipo()
                          │
                          ▼
         convertir_dataset_entrenamiento()
                          │
                          ▼
         aplanar_datos_tecnicos()
                          │
                          ▼
         ┌─────────────────────────────────────────┐
         │           entrenar_modelo()             │
         ├─────────────────────────────────────────┤
         │  1. cargar_datos() → X, y               │
         │  2. StratifiedKFold (5 folds)           │
         │  3. cross_val_score (métricas)          │
         │  4. train_test_split (70/30)            │
         │  5. RandomForestClassifier.fit()        │
         │  6. metricas_de_clases()                │
         │  7. calcular_feature_importance()       │
         └─────────────────────────────────────────┘
                          │
                          ▼
         save_modelo() → modelo_*.pkl + metrics.pkl
```

---

## Ciclo de Predicción

```
    Nuevos datos (DataFrame)
              │
              ▼
    load_modelo() ← modelo_*.pkl
              │
              ├── Sin .pkl → MODO ETIQUETADO
              │       │
              │       └── Envía TODAS las oportunidades con origen="system"
              │           para que el usuario apruebe/rechace por Telegram
              │           y acumule muestras de entrenamiento
              │
              └── Con .pkl → MODO PREDICCIÓN
                      │
                      ▼
              predecir_modelo()
                      │
                      ├── predict() → etiqueta
                      └── predict_proba() → confianza
                      │
                      ▼
              Clasificación por umbral:
              ┌─────────────────────────────────────┐
              │ confianza >= umbral (0.65) → acción │
              │ confianza >= 0.35      → observar   │
              │ confianza <  0.35      → descartar  │
              └─────────────────────────────────────┘
```

> **Modo etiquetado:** ver sección [Modo Etiquetado (Buy y Sell)](#modo-etiquetado-buy-y-sell) para el mecanismo completo (activación automática vs. manual, persistencia).

---

## Modo Etiquetado (Buy y Sell)

Mecanismo para acumular muestras de entrenamiento cuando el dataset es insuficiente (< ~80-100, ver Notas Técnicas #6). Mientras está activo, **todas** las oportunidades se envían a Telegram con botones ✅ Aprobar / ❌ Rechazar (`origen="system"`), sin pasar por el modelo — el resultado de cada aprobación/rechazo alimenta `recomendado=1`/`-1` en la tabla `oportunidades`, que es el dataset de entrenamiento.

### Activación automática
Si `load_modelo()` no encuentra el `.pkl` del modelo (`self.IAbuy.modelo is None` / equivalente Sell), `evaluar_oportunidades_*_con_IA()` cae al modo etiquetado sin intervención manual (`Class_DashBot.py`).

### Activación/desactivación manual — botón "Modelo ON"
Tabs **"Buy IA"** / **"Sell IA"** → botón junto a "Modelo"/"Entrenar":
- **"Modelo ON"** (gris) → modo etiquetado inactivo, el modelo predice normalmente.
- **"Etiquetando"** (rojo) → modo etiquetado forzado.

Click llama a `chatbot._set_modo_etiquetado(tipo, valor)` (`Class_DashBot.py`), que:
1. Actualiza `self.modo_etiquetado_buy` / `self.modo_etiquetado_sell` en memoria.
2. Persiste `modo_etiquetado: true/false` en el JSON `paramts` de la tabla `modelos_ia` (clave `modelo_buyv01` / `modelo_sellv01`) — **independiente de si existe el `.pkl`**, sobrevive reinicios.

Con el flag en `true`, `evaluar_oportunidades_*_con_IA()` ni siquiera llama `load_modelo()` — evita el `joblib.load()` innecesario y envía directo a etiquetado.

> **Importante:** el flag manual tiene prioridad y persiste aunque el `.pkl` exista y sea válido. Si quedó en `true` (p. ej. tras forzar etiquetado para juntar muestras), el modelo **no vuelve a predecir automáticamente** aunque se reentrene — hay que apagarlo a mano ("Etiquetando" → click → "Modelo ON") una vez reunidas suficientes muestras nuevas.

---

## Archivos Generados

| Archivo | Descripción |
|---------|-------------|
| `modelo_sellv01.pkl` | Modelo RandomForest para Sell |
| `modelo_sellv01_metrics.pkl` | Métricas del modelo Sell |
| `modelo_buyv01.pkl` | Modelo RandomForest para Buy |
| `modelo_buyv01_metrics.pkl` | Métricas del modelo Buy |

Ubicación: Directorio definido por `define_FileCache()`

---

## Dependencias

```python
import json
import warnings
import numpy as np
import pandas as pd
import joblib
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import (
    cross_val_score,
    StratifiedKFold,
    train_test_split,
)
from sklearn.exceptions import UndefinedMetricWarning
from sklearn.metrics import (
    classification_report,
    f1_score,
    roc_auc_score,
)
from Modulos_Mysql import RepositorioOportunidadesBuySell, BDsystem
from Modulos_Utilitarios import define_FileCache
```

---

## Uso desde CLI

```python
if __name__ == "__main__":
    AiSell = ModeloOportunidadesSell()
    AiSell.run_entraminetoSell()
```

---

## Integración con Sistema

### Fuente de Datos
- Tabla BD: `bdinv.oportunidades`
- Campos: `tipo='buy'` o `tipo='sell'`
- Campo JSON: `json_detalle` contiene indicadores técnicos

### Monitor IA (Class_SystemStatus.py)
- Botón "Entrenar" ejecuta `run_entrenamientoBuy()` / `run_entraminetoSell()`
- Muestra métricas del modelo
- Botón **"Modelo"** abre el panel de configuración directamente desde la UI
- Botón **"Modelo ON" / "Etiquetando"** fuerza modo etiquetado manual — ver [Modo Etiquetado (Buy y Sell)](#modo-etiquetado-buy-y-sell)

### Cómo actualizar parámetros del modelo (sin tocar código ni BD manualmente)

```
Tab "Buy IA"  →  botón "Modelo"  →  campo "Parámetros (JSON)"  →  editar  →  "Guardar"
```

El panel expone todos los parámetros del modelo editables en tiempo real:

| Campo | Descripción |
|-------|-------------|
| Modelo | Identificador en BD (`modelo_buyv01`) |
| Nombre | Nombre descriptivo |
| Tipo Modelo | Algoritmo (`RandomForest`) |
| Define Modelo | Tipo de clasificador (`buy_classifier`) |
| Parámetros (JSON) | `n_folds`, `max_depth`, `umbral_buy`, `umbral_observacion`, etc. |
| Documentación | Texto libre — descripción del modelo |

> **Importante:** después de cambiar parámetros y guardar, hay que reiniciar la app y presionar **"Entrenar"** para que el modelo se reentrene con los nuevos valores.

![[graphics/config-modelo-buy.png]]

### Evaluación de Oportunidades
- `Class_DashBot.py`: `evaluar_oportunidades_*_con_IA()`
- Sin modelo: envía todas las oportunidades con `origen="system"`
- Con modelo: filtra por umbral y envía con `origen="ia"`

---

## Notas Técnicas

1. **Balance de Clases**: Usa `class_weight="balanced"` para manejar datasets desbalanceados
2. **Validación Cruzada**: StratifiedKFold mantiene proporción de clases en cada fold
3. **Umbrales Ajustables**: Se leen desde BD tabla `modelos_ia`. Buy lee `umbral_buy`, Sell lee `umbral_sell` — claves distintas, no intercambiables
4. **Timeframes**: Por defecto solo usa diario `["d"]` para reducir features
5. **NaN Handling**: Elimina columnas con más del 50% de NaN; indicadores con `NaN`/`Infinity` se sanitizan a `null` antes de insertar en BD
6. **Dataset mínimo**: Se necesitan ~80-100 muestras etiquetadas para que el modelo generalice correctamente. Con menos muestras las métricas CV son poco confiables (alta varianza ±std)
7. **Modo etiquetado**: ver [Modo Etiquetado (Buy y Sell)](#modo-etiquetado-buy-y-sell) — automático (sin `.pkl`) o forzado por botón, persistido en BD

---

## Features del modelo Buy — medición y decisión (2026-11-06)

### Lo que tenía el modelo vivo

Leído de `deploy/tmp/modelo_buyv01.pkl` (entrenado 2026-11-05 05:37): **16 features**.

| Grupo | Nº | Importancia acumulada |
|---|---|---|
| Precio (`rsi_d`, `macd_d`, `Close_d`, 9 EMAs, `fibo_longico_d`) | 12 | **100%** |
| Sentimiento (`sentiment_score`, `_3d_avg`, `_7d_avg`, `_patron`) | 4 | **0.000000** |

`rsi_d` 26,6% · `macd_d` 14,2%. Métricas: 111 muestras (68 buy / 43 wait),
precisión 0,758 · recall 0,757 · F1 0,756 · ROC-AUC 0,821.

**No es que RSI/MACD sean los mejores predictores: son lo único con dato.** Tres causas
distintas, ninguna de modelado:

1. **Las 4 de sentimiento estaban constantes en 0** desde el 2026-09-28 (ver
   [[design-sentimiento]] § "Stock — la caída de 39 días"). Una feature constante recibe
   importancia **exactamente** 0 en un árbol: no aporta poco, no existe.
2. **Las 4 escalares de negocio no llegaron al modelo.** `scalar_features` declara
   `score`, `ganancia_precio`, `ganancia_inversion`, `dividend_yield`, y las 170 filas
   `tipo='buy'` de `oportunidadesbuysell` las traen con valor en el `json_detalle`
   (medido: 0 ausentes, 0 nulas). Pero el binario de `deploy/` es del **2026-08-29** y el
   modelo se entrenó el 05-11 **con ese binario** — el código que entrena no es el del repo.
3. **Los 6 rangos no pueden entrar nunca.** `aplanar_datos_tecnicos` los nombra
   `13_semanas_max__d` (doble `_`: el sufijo ya trae el suyo) y `cargar_datos` los busca
   `13_semanas_max_d`. Bug de nombre, no de datos.

### Sentimiento y Señal no son dos features: son la misma, dos veces

El sentimiento es **uno de los 8 votos** de la Señal (el voto `Sent`, ver [[ref-consenso]]).
Mandar los dos al modelo lo mete dos veces: crudo en 4 columnas y diluido en 1/8 de la suma.
Pesaría **5 veces más** que los otros 7 conceptos del voto.

**Decisión: en el modelo queda la Señal. El sentimiento crudo sale.** El módulo de sentimiento
no se toca — sigue alimentando el voto `Sent` y lo consume Preservation.

`consenso_suma` sola colapsa 8 conceptos en un número, así que entra acompañada de los dos
componentes que informan por su cuenta, para que el árbol pueda cruzar
*"institucional fuerte + RSI bajo"*:

| | Antes | Después |
|---|---|---|
| Precio | 12 | 12 |
| Sentimiento crudo | 4 | **0** |
| Señal + institucional (`consenso_suma`, `inst_score`, `fh_buy_ratio`) | 0 | 3 |
| **Total** | **16** | **15** |

**El modelo no crece: sustituye.** Con 111 muestras no se podían agregar features sin
empeorar el sobreajuste — la nota técnica 6 de este documento pide 80-100 mínimo y estamos
apenas encima.

### Cobertura — por qué la Señal sí tiene dato y el sentimiento no tenía

Medido sobre las 170 oportunidades `tipo='buy'`:

| Dato | Cobertura |
|---|---|
| `consenso_suma` | **154 / 170 (91%)**, rango real −1 a 6 |
| `inst_score` | 150 / 170 |
| Sin fila en `market` | 16 — los pares cripto (`ADAUSDT`, `ZILUSDT`, `CGPTUSDT`, `ICPUSDT`, `FILUSDT`) y 3 ETF (`TLT`, `VGSH`, `BIL`) |

**`load_consenso_features` no filtra por `encartera`**, a diferencia de
`load_sentiment_features`: las oportunidades de compra incluyen símbolos fuera de cartera
(PFE, BP, ABX, CVS tienen `encartera='N'` y Señal calculada). Filtrar por cartera los perdería.

**Punto de atención.** El fallback es `0.0` para los 16 sin fila, y `consenso_suma = 0` es un
valor **legítimo** (hay símbolos con suma 0). El modelo no puede distinguir *"Señal neutra"*
de *"sin Señal"* — el mismo borde que `sentiment_patron` ya tiene documentado en
[[design-sentimiento]].

**Los 16 son la medición que justifica [[plan-market-crypto]]** (corregido el 2026-11-06: antes
apuntaba a [[plan-crypto-gainscapture]], que está cerrado y resolvió su propio caso **sin** tocar
`market`). Los 5 pares cripto de esta lista son la prueba de que el CSV de entrada del modelo Buy
**ya trae filas cripto** — `csv_OptionBuy_write` (`Class_customer.py:697`) itera
`DataHub.rebalanceo.items()` y toma `ranking` para todo lo que no es Stock — y de que sus tres
features de Señal entran en `0.0`. Es el mismo defecto que el de las 4 features de sentimiento,
en otra columna.

Y antes de cargar esas filas hay que corregir el sesgo del Consenso: una cripto sin ETF al contado
no tiene 13F, opciones ni analistas, y los votos `Net`/`Flujo`/`Cob` **no abstienen** aunque la spec
diga que sí — ver [[spec-consenso]] § "El código no abstiene donde esta spec dice que sí". Cargar
filas sin eso le daría al modelo ruido con signo en vez de una señal.

### Cambios aplicados

| Archivo | Cambio |
|---|---|
| `Modulos_Mysql.py` | `load_consenso_features(account)` → `{symbol: {consenso_suma, inst_score, fh_buy_ratio}}` |
| `Class_IA_modelos.py` | `ModeloOportunidadesBuy.COLS_CONSENSO`; `enriquecer_con_sentimiento` → `enriquecer_con_features(df, features, columnas)` (genérico, solo en la clase Buy); `scalar_features` sustituye las 4 de sentimiento por las 3 de Señal; llamada en `predecir_oportunidades` |
| `Class_SystemStatus.py` | entrenamiento Buy del panel (`:3755`) — tiene que quedar **simétrico** con la predicción o hay desvío entre entrenamiento y servicio |

**La clase Sell no se tocó.** Conserva `enriquecer_con_sentimiento` y sus 4 features de
sentimiento; la decisión de arriba se evaluó sobre Buy.

### Pendientes

| # | Qué | Por qué importa |
|---|---|---|
| 1 | **Reconstruir el binario** | El modelo se entrena con código de agosto: sin esto nada de lo anterior corre en producción |
| 2 | Doble `_` en los nombres de rango (`aplanar_datos_tecnicos`) | +6 features reales, costo cero |
| 3 | Reentrenar y comparar | Las features nuevas no existen hasta el próximo entrenamiento |
| 4 | Histórico de métricas | `modelo_buyv01_metrics.pkl` **se sobreescribe en cada entrenamiento**: no hay con qué comparar un antes y un después, ni cómo decidir cuándo el modelo es confiable para operar solo |
| 5 | Muestra | 111 casos para 15 features. Es el techo real de la automatización, no las features |

---

**Ver también:**
- [[ref-consenso]] — los 8 votos de la Señal, donde `Sent` es uno
- [[design-sentimiento]] — por qué las 4 features estaban en 0
- [[ref-oportunidades]] — de dónde salen las filas `oportunidadesbuysell`
- [[plan-market-crypto]] — los 5 pares cripto con features en 0.0, y qué hay que arreglar antes
- [[spec-consenso]] — el sesgo del denominador que afecta a `consenso_suma`
