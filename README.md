# final_kaggle
Repo final de envios a kaggle

# Problema #02 – DR Data Drifting (Grupo A)

Workflow **Analista Jr** en R (`data.table` + LightGBM) para la competencia Kaggle `labo-1-virtual-2026-analista-jr`.
Validación: mes **202107** · Predicción futura: mes **202109**.

Este documento resume el trabajo hecho en los envíos y qué se modificó en cada notebook.

---

## 1. Idea general

El problema es el *data drifting* en variables monetarias (la inflación hace que los montos de 2021 no sean comparables con los de 2019). Se probaron distintos métodos de **DR (Data Drifting Reduction)** y se compararon con el mismo flujo:

1. **CA**: variables de meses dañados se ponen en NA.
2. **DR**: se corrigen las variables monetarias (`ninguno`, `deflacion` × IPC, `dolar_blue`, `dolar_oficial`, `UVA`, `rank_simple`, `rank_cero_fijo`, `estandarizar`, variantes `*_pond`, `deflacion_selectiva`).
3. **FE intra-mes**: `kmes`, `mpayroll_sobre_edad`.
4. **FE histórica**: `lag1`, `lag2`, `delta1`, `delta2`.
5. **Grid search LightGBM** sobre AUC en 202107.
6. **Semillerio**: un modelo por semilla, ensamble por promedio de probabilidades.
7. **Corte para Kaggle**: se envían los top-N de 202109 (N entre 1300 y 2600 según el notebook).

### Semillas fijas

`165463` (primigenia), `329969`, `454253`, `593899`, `980249`, `100183`, `250027`, `400031`, `550049`, `700057`.

### Convención de carpetas

`experimento = experimento_base + posición del método en metodos_DR − 1`

| Serie | Descripción | Ejemplos |
|---|---|---|
| 9600 | Todos los métodos, positivos habituales | WF9600 ninguno · 9601 deflacion · 9602 dolar_blue · 9603 dolar_oficial · 9604 rank_simple · 9605 rank_cero_fijo · **9606 estandarizar** · 9607 ninguno_pond · 9608 deflacion_pond · 9609 deflacion_selectiva |
| 9700 | Solo positivos BAJA+2 | — |
| 9800 | Envío final (estandarizar) | **WF9806** |

---

## 2. Resultados de referencia

- Mejor grid con `rank_cero_fijo` intra-mes: **AUC 0.936103** (`num_leaves=256`, `min_data_in_leaf=2048`, `feature_fraction=0.8`, 140 iteraciones).
- Kaggle con **estandarizar** (WF9606): **94,288 @1600**, **95,202 @1700**, **93,374 @1800**.
- El envío `BLEND=estandarizar (prob) ensamble=10sem envios=1750` salió del notebook *Blend_Cortes* (candidato `estandarizar` = WF9606, corte 1750).

---

## 3. Qué se modificó en cada notebook

### Línea z728 (notebook del profesor y derivados)

| Notebook | Experimento | Método DR | Semillas | Cortes | Grid | Cambios respecto del anterior |
|---|---|---|---|---|---|---|
| `z728_final_junior_power` (original) | 9110 | `deflacion` | 20 primos aleatorios | 1700–2300 (cada 100) | 5×5×2 = 50 | Base. Tiene un bug con los nombres de las funciones de dólar. |
| `z728_final_junior_power_lags` | 9110 | `deflacion` | 20 primos aleatorios | 1700–2300 | 50 | Original + **4 celdas nuevas**: variables macro y lags extra. El resto es idéntico (conserva el bug del dólar, inofensivo con `deflacion`). |
| `z728_final_junior_power_10semillas` | 9120 | `rank_cero_fijo` | **10 fijas** (primigenia 165463) | 1600–2600 | 50 | Corrige el bug de dólar; reemplaza los primos aleatorios por las 10 semillas fijas; agrega `stopifnot(semillas[1] == primigenia)`; `semillerio = length(semillas)`. |
| `z728_final_junior_power_semillerio10_corte` | 9120 | `rank_cero_fijo` | 10 fijas | 1300–1900 (cada 100) | `modo = "completa"` | 10semillas + modo de grid, celdas de tiempos, **modelos de validación por semilla**, candidatos/mezcla, **estimación del corte** (teórico, teórico ajustado, isotónico, óptimo local), celda de Kaggle con candidatos y mensaje `exp=… DR=… cand=… s=… envios=…`, bitácora `envios_kaggle.csv`. |
| `z728_final_junior_power_macro_lags` | 9130 | `rank_cero_fijo` | 10 fijas | 1300–1900 | completa | semillerio10_corte + `PARAM$FE$macro = c("IPC","dolar_oficial")`, `lags_extra = c(3,6)`, `delta_extra = FALSE`; columnas `macro_*` y lag3/lag6. |

### Línea z620 (workflow v6, envío final)

| Notebook | Experimento | Método DR | Semillas | Cortes | Grid | Cambios |
|---|---|---|---|---|---|---|
| `z620_WorkFlow_06_DR_grupoA` (v6) / `_01` (v6.1) | según serie | configurable | — | — | — | Base del workflow por método. v6.1 solo agrega `deflacion_selectiva` (WF9609). |
| `z620_..._final_z728` | **9800 → WF9806** | `estandarizar` | 10 fijas | **1500–1900 (cada 50)** | **3×4×2 = 24** | Ver detalle abajo. |
| `z620_..._final_z728_f2` | idéntico al anterior (mismo md5) | | | | | Copia del mismo notebook. |

**Detalle de `final_z728`:**

- `PARAM$DR$metodo <- "estandarizar"`, `PARAM$experimento_base <- 9800`.
- Grid reducido y centrado en la zona útil: `num_leaves ∈ {128, 256, 512}`, `min_data_in_leaf ∈ {512, 1024, 2048, 4096}`, `feature_fraction ∈ {0.5, 0.8}`.
- Cortes Kaggle `seq(1500, 1900, by = 50)`.
- **Guarda de NA**: `stopifnot(!anyNA(tb_nueva$AUC))` antes de `setorder(-AUC)` (el orden descendente pone los NA primero y elegiría un grid inválido).
- **Registro de envíos** `registro_envios.csv` (fecha, experimento, método, corte, archivo, md5, unos, subido, mensaje, respuesta).
- **Training por semilla reemplazado por el de z728**: `crear_modelo_final`, modelos en `modelos/modelo_<semilla>.txt`, scoring con `agregar_a_ensemble` (columnas `prob_<isem>`), salida `prediccion.txt`, mensaje Kaggle con `length(PARAM$FT$semillas)`.
- **Sección nueva "Validación por semilla"** (después de `prediccion.txt`): entrena con ≤202105 y puntúa 202107 por cada semilla, y escribe `validacion_clases.txt`, `firma_hiperparametros.txt`, `prediccion_val_<semilla>.txt`, `tb_semillas_validacion.txt` y `tb_resumen_validacion.txt`. Con esto el notebook *Blend_Cortes* puede leer WF9806 igual que leyó WF9606.
- Textos de encabezado y markdown actualizados.

---

## 4. Decisiones de diseño

- **Una carpeta por corrida.** `permitir_carpeta_existente = FALSE` frena el notebook si la carpeta no está vacía, para no mezclar resultados.
- **Ensamble por promedio de probabilidades** de las 10 semillas.
- **Blend_Cortes** ordena por id y guarda `KB_<cand>_<corte>.csv`; el notebook 06 ordena por probabilidad y guarda `KA<exp>_<corte>.csv`. No son intercambiables.
- **Umbral teórico de corte = 2.5 %** (25000 / (975000 + 25000)).

---

## 5. Advertencias conocidas

1. **WF9806 puede existir ya.** Si aparece "La carpeta WF9806 ya existe y no esta vacia": usar `PARAM$experimento_base <- 9900`, o mover la carpeta anterior. Poner `permitir_carpeta_existente <- TRUE` funciona, pero el training de z728 **saltea** los `modelo_<semilla>.txt` existentes y se pueden reusar modelos viejos sin avisar.
2. **Los envíos 1550 y 1750 salieron de un Blend editado** que no está en este repositorio; solo se conserva `KB_estandarizar_1550.csv` (1,550 unos, orden por id).
3. **El costo de −25000 es un supuesto** y condiciona el corte teórico.

---

## 6. Cómo correr el envío final

Parámetros a revisar en el notebook `final_z728`:

```r
PARAM$DR$metodo        <- "estandarizar"
PARAM$experimento_base <- 9800          # cambiar a 9900 si WF9806 ya existe
PARAM$kaggle$cortes    <- seq(1500, 1900, by = 50)
```

Ejecutar en orden: preparación de datos → grid → training por semilla → scoring → validación por semilla → envío a Kaggle (con registro).
Los resultados se guardan en `/content/buckets/b1/exp/` (`gs://enrique_larrea_mcd_buckito2026/exp/`).
