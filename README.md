# Entrega final - Laboratorio de Implementacion I (Analista Jr) - Grupo B

**Autora:** Ana Lucia Hernandez (Grupo B, individual) - **Competencia Kaggle:** `labo-1-virtual-2026-analista-jr`

## Archivo entregado
`KA9390_s125_2100.csv`: ensamble de **125 modelos LightGBM** (5 corridas de 25 semillas), entrenados **sin los meses 202003 a 202012**, con **2100 envios** (los 2100 clientes de mayor probabilidad en 202109). 

## Como reproducir EXACTAMENTE ese archivo

Entorno: el de la materia (VM con R, `data.table`, `lightgbm`, `primes`, `yaml`, `R.utils`, `mice`), con el dataset original en
`/content/datasets/analistajr_competencia_2026.csv.gz` y la carpeta de trabajo `/content/buckets/b1/exp`.
**El dataset no se incluye en este repositorio.**

1. **`notebooks/01_z729_final_junior_semillerio25.ipynb`** (experimento 9300). Genera `WF9300/impo.txt`, la importancia de variables que usa el paso 2 para elegir las 60 variables sobre las que se arman las variables nuevas. Se incluye una copia de ese archivo en `datos_intermedios/impo.txt` (MD5 `761134693a1b2f338eb1b44641465bef`); si se la copia a `/content/buckets/b1/exp/WF9300/impo.txt`, este paso se puede omitir. `datos_intermedios/WF9300_PARAM.yml` es el registro de parametros de esa corrida.
2. **`notebooks/02_z729_final_sin2020_x5.ipynb`** (experimento 9390). Ejecutarlo completo de arriba hacia abajo. Deja en `/content/buckets/b1/exp/WF9390/kaggle/` los archivos `KA9390_s125_<corte>.csv` para los cortes 1900 a 2300; **el entregado es `KA9390_s125_2100.csv`**.
   * Al terminar, el notebook graba `WF9390/PARAM.yml`; se puede comparar con `datos_intermedios/WF9390_PARAM.yml` (el de la corrida entregada): mismas 125 semillas, mismos `final_train` y mismos hiperparametros.
   * En este repositorio `PARAM$kaggle$enviar` esta en `FALSE` (solo genera los archivos). En la corrida original estaba en `TRUE`; es la unica diferencia con lo que se ejecuto y no afecta al archivo.

**Determinismo:** la semilla primigenia es `100103` y todas las semillas salen de `set.seed(100103, kind = "L'Ecuyer-CMRG"); sample(primos)[1:125]` (primos entre 100000 y 1000000). Con la misma semilla primigenia y el mismo entorno, las corridas dan resultados identicos (se verifico volviendo a correr con otro numero de experimento: mismos puntajes en Kaggle).

**Verificacion:** el MD5 del archivo entregado es `f00b5b10e6a9c73316442c443de85b05` (`md5sum KA9390_s125_2100.csv`). El archivo tiene 33080 filas (una por cliente de 202109), con `Predicted = 1` en exactamente 2100 de ellas.

## Que se hizo (resumen)
Partiendo de la solucion oficial (workflow con data drifting por deflacion, LightGBM, `num_iterations = 64`):

| # | Cambio | Solucion oficial (`z728_final_junior_power`) | Esta entrega | Fundamento |
|---|---|---|---|---|
| 1 | Variables nuevas "V3" | Lags y delta lags de orden 1 y 2 sobre todas las variables | Lo mismo **mas**, para las 60 variables mas importantes (corrida 9300): ventana de 3 meses (promedio, minimo, maximo) y lags/deltas de orden 3 y 6 | Experimento del laboratorio (`z729_FE_experimentos_junior`, 10 semillas): V3 mejora la ganancia local en +3.61 M contra la base (9 de 10 semillas, p = 0.008) |
| 2 | `numero_de_cliente` y `foto_mes` | Se usan como variables | Se quitan de las variables de entrenamiento (se siguen usando para ordenar y calcular lags) | Problema #16 (Grupo B, 10 semillas): sacar ambas juntas mejora la ganancia (p = 0.0059); ademas `foto_mes` toma en 202109 un valor que el modelo nunca vio en el entrenamiento |
| 3 | Meses de entrenamiento final | 201901 a 202107 | 201901 a 202107 **sin 202003 a 202012** | Meses de pandemia atipicos; ver "Como se decidio" |
| 4 | Hiperparametros | Grid Search por AUC (learning rate 0.03, hojas, `min_data_in_leaf`, `feature_fraction`) | Fijos (ver abajo), sin busqueda | En validacion temporal con 25 semillas, el regimen del Grid Search no mejoro (-0.04 % a -0.6 %) |
| 5 | Semillerio | 20 semillas | 125 modelos: 5 corridas de 25 semillas | Menor varianza entre semillas (con una sola semilla el publico variaba de 62 a 89) |
| 6 | Corte de envios | Cortes 1700 a 2300 | 2100 | Elegido por validacion temporal (menor perdida contra el mejor corte de cada mes), no por el leaderboard publico |

Hiperparametros fijos: `objective = binary`, `max_bin = 31`, `num_iterations = 64`, `learning_rate = 0.12`, `num_leaves = 32`, `min_data_in_leaf = 711`, `feature_fraction = 0.20`, `extra_trees = FALSE`.

## Como se decidio (validacion temporal, no k-fold)
Para validar en el mes `m` se entrena con los meses hasta `m - 2`; la ganancia (+975000 si BAJA+2, -25000 si no) se calcula con el corte escalado al tamano del mes. Se usaron ensambles de 25 semillas.

* **Variables V3:** en el experimento de variables del laboratorio (10 semillas) mejoraron la ganancia local en +3.61 M contra la base (9 de 10 semillas, p = 0.008). V1 (solo ventana de 3 meses) no fue significativo (p = 0.22) y V2 (solo lags/deltas 3 y 6) dio +3.10 M (10 de 10, p = 0.006).
* **Sin pandemia contra con pandemia:** en 202105-202107 dio +3.5 % (81.45 M contra 78.70 M). Al ampliar a **6 meses** (202102-202107) la ventaja **no se sostiene en todos**: gana en 2 de 6 meses (los dos mas recientes) y el promedio de 6 meses es -1.2 %. Se mantuvo `sin2020` por ser mas fuerte en los meses mas recientes, los mas parecidos a 202109.
* **Corte 2100:** es el de menor perdida contra el mejor corte de cada mes en 202105-202107 (a lo sumo 1.0 M); con 6 meses, 2200-2300 quedan ligeramente por encima (diferencias dentro del ruido).
* **Descartados (dentro del ruido o peores):** hiperparametros distintos de los fijos (learning rate, hojas, `min_data_in_leaf`, `feature_fraction`), el regimen del Grid Search del profesor, mezclas de modelos, ventanas de training cortas, y mas de 25 semillas por corrida.
* **Leaderboard publico:** no se uso para elegir. Con 80 semillas distintas del mismo modelo y el mismo corte, el puntaje publico fue de 62 a 89 (media 79.4, desvio 5.1): el publico (30 % de los datos) mide principalmente suerte de semilla.

## Estructura
```
README.md
notebooks/
  01_z729_final_junior_semillerio25.ipynb   # experimento 9300 (genera impo.txt)
  02_z729_final_sin2020_x5.ipynb            # experimento 9390 (genera el archivo entregado)
datos_intermedios/
  impo.txt                                  # importancia de variables de la corrida 9300
  WF9300_PARAM.yml                          # parametros registrados de la corrida 9300
  WF9390_PARAM.yml                          # parametros registrados de la corrida entregada (9390)
```
No se incluyen datasets, credenciales de Kaggle ni tokens.
