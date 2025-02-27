# 08_Modelo_unidad_py_ic_segsoc {-}

Para la ejecución del presente análisis, se debe abrir el archivo **08_Modelo_unidad_py_ic_segsoc.R** disponible en la ruta `Rcodes/2020/08_Modelo_unidad_py_ic_segsoc.R`.

Este script en R se enfoca en la creación de un modelo multinivel para analizar la carencia en por cobertura de seguridad social usando datos de encuestas y censos. El proceso inicia con la limpieza del entorno de trabajo y la carga de librerías esenciales para el análisis. Se define el límite de memoria con `memory.limit(10000000)` para asegurar suficiente espacio durante el procesamiento de datos. Luego, se cargan los datos necesarios (`encuesta_sta`, `censo_sta`, `statelevel_predictors_df`) y se excluyen ciertas variables de los datos de predictores.

La selección de variables relevantes para el modelo se realiza a través de un procedimiento comentado que emplea la técnica de Recursive Feature Elimination (RFE) usando un modelo Random Forest implementado en Python. Las variables seleccionadas se listan explícitamente y se combinan con otras covariantes para formar cov_names. Posteriormente, se construye una fórmula del modelo (`formula_model`) que incluye efectos aleatorios para cve_mun, hlengua y discapacidad, y efectos fijos para las demás variables, considerando tanto la variabilidad dentro de cada grupo como las características específicas de cada variable.

El ajuste del modelo se realiza mediante la función modelo_dummy, transformando la variable de interés (`ic_segsoc`) en formato binario y utilizando los datos de encuesta y censo junto con la fórmula del modelo y las variables de agregación. Los resultados del modelo se guardan en un archivo RDS (`fit_mrp_ic_segsoc.rds`) y se documentan para futuros análisis. Esto incluye la exportación de los resultados del modelo multinivel y la creación de visualizaciones pertinentes para evaluar el desempeño del modelo y la distribución de las predicciones.



#### Limpieza del Entorno y Carga de Bibliotecas {-}


En esta etapa, se realiza la limpieza del entorno de R para eliminar objetos previamente cargados y garantizar un inicio limpio para el análisis. Esto se hace utilizando el comando `rm(list = ls())`, que elimina todos los objetos del espacio de trabajo actual.

A continuación, se cargan las bibliotecas necesarias para el análisis. Estas bibliotecas incluyen:

- **`patchwork`**: Para la combinación de múltiples gráficos en una sola visualización.
- **`nortest`**: Proporciona pruebas de normalidad para analizar la distribución de los datos.
- **`lme4`**: Utilizada para ajustar modelos lineales mixtos, que permiten manejar datos con estructura jerárquica.
- **`tidyverse`**: Un conjunto de paquetes para la manipulación y visualización de datos.
- **`magrittr`**: Proporciona el operador `%>%` para facilitar el encadenamiento de operaciones en R.
- **`caret`**: Para la creación de modelos de machine learning y la selección de variables.
- **`car`**: Ofrece herramientas para el análisis de regresión y diagnóstico de modelos.
- **`randomForest`**: Implementa el algoritmo de bosques aleatorios para clasificación y regresión.
- **`reticulate`**: Permite la integración de Python en R, facilitando el uso de bibliotecas de Python desde el entorno R.

Además, se importan módulos específicos de Python usando `reticulate`:

- **`pandas`**: Para la manipulación de datos en estructuras tabulares.
- **`sklearn.feature_selection`**: Para la selección de características en modelos de machine learning.
- **`sklearn.ensemble`**: Proporciona métodos de ensamblaje, como los bosques aleatorios y gradientes impulsados.

Finalmente, se carga un script adicional, `modelos_freq.R`, que probablemente contiene funciones personalizadas o modelos específicos necesarios para el análisis. Esto asegura que todas las herramientas y funciones requeridas estén disponibles para proceder con el análisis.


``` r
rm(list =ls())

# Loading required libraries ----------------------------------------------

library(patchwork)
library(nortest)
library(lme4)
library(tidyverse)
library(magrittr)
library(caret)
library(car)
library(randomForest)
library(reticulate)

pd <- import("pandas")
sklearn_fs <- import("sklearn.feature_selection")
sklearn_ensemble <- import("sklearn.ensemble")
```

#### Carga de Datos{-}

En esta fase, se procede a la carga de los datos necesarios para el análisis. Se utilizan las funciones `readRDS()` para leer archivos en formato RDS, que es un formato de almacenamiento de objetos en R.

1. **Carga de la Encuesta:** Se carga el conjunto de datos de la encuesta, almacenado en el archivo `../input/2020/enigh/encuesta_sta.rds`. Este conjunto de datos contiene información sobre las variables de interés para el análisis.

2. **Carga del Censo:** Se lee el archivo `../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds`, que contiene datos censales ampliados. Estos datos se usarán para la calibración y comparación con los datos de la encuesta.

3. **Carga de Predictores Estatales:** Se carga el archivo `../input/2020/predictores/statelevel_predictors_df.rds`, que incluye predictores a nivel estatal que se utilizarán para el análisis.

Además, se extraen los nombres de las variables del dataframe `statelevel_predictors_df` y se filtran para eliminar las variables que comienzan con "hog_" o "cve_mun", asegurando que solo se conserven las variables relevantes para el análisis posterior. Esto se realiza mediante la función `grepl()` para identificar y excluir las variables no deseadas.


``` r
memory.limit(10000000)
source("../source/modelos_freq.R")
# Loading data ------------------------------------------------------------

memory.limit(10000000)
encuesta_sta <- readRDS("../input/2020/enigh/encuesta_sta.rds")
censo_sta <- readRDS("../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")
statelevel_predictors_df <- readRDS("../input/2020/predictores/statelevel_predictors_df.rds")

cov_names <- names(statelevel_predictors_df)
cov_names <- cov_names[!grepl(x = cov_names, pattern = "^hog_|cve_mun")]
```

#### Selección de Variables{-}

Se definen las variables de agregación necesarias para el análisis y se seleccionan las variables predictoras más relevantes. El vector `byAgrega` incluye variables clave como `ent`, `cve_mun`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`, y `nivel_edu`, que se utilizarán para la agregación de los datos. 

Aunque el código para la selección de variables mediante un proceso de RFE (Recursive Feature Elimination) usando Python está comentado, se detallan los pasos para el uso de esta técnica. En lugar de ejecutar el código Python, se ha seleccionado manualmente un conjunto de variables relevantes basado en el análisis previo. Estas variables seleccionadas incluyen indicadores como `porc_rur`, `porc_urb`, `porc_ing_ilpi_rur`, `porc_ing_ilpi_urb`, `porc_jub_urb`, `porc_segsoc15`, `plp15`, `ictpc15`, `pob_urb`, y `pob_tot`.

El conjunto de variables `cov_names` se compone de las variables seleccionadas más otras variables relevantes para el análisis, excluyendo algunas que no se consideran necesarias para el modelo final. Las variables que se excluyen se encuentran en el conjunto `cov_registros`, y el resultado es un vector de variables que se utilizará en la definición de la fórmula del modelo.


``` r
## Selección de variables

byAgrega <-
  c("ent",
    "cve_mun",
    "area",
    "sexo",
    "edad",
    "discapacidad",
    "hlengua",
    "nivel_edu" )

# encuesta_sta2 <- encuesta_sta %>%   mutate(
#   yk = as.factor(ifelse(ic_asegsoc == 1 ,1,0))) %>%
#   inner_join(statelevel_predictors_df[, c("cve_mun",cov_names)])
# 
# table(encuesta_sta2$yk, encuesta_sta2$ic_segsoc)
# 
# # Convertir 'encuesta_sta2' a un dataframe de Python
# encuesta_sta2_py <- pd$DataFrame(encuesta_sta2)
# 
# # Obtener 'X' y 'y' del dataframe de Python
# X <- encuesta_sta2_py[cov_names]
# y <- encuesta_sta2_py[['yk']]
# 
# # Crear el modelo de clasificación, por ejemplo, un Random Forest
# modelo <- sklearn_ensemble$RandomForestClassifier()
# 
# # Crear el selector RFE con el modelo y el número de características a seleccionar
# selector <- sklearn_fs$RFE(modelo, n_features_to_select = as.integer(10))
# 
# # Ajustar los datos
# selector$fit(X, y)
# 
# # Obtener las variables seleccionadas
# variables_seleccionadas <- X[selector$support_] %>% names()

variables_seleccionadas <-
  c(
    "porc_rur",
    "porc_urb",
    "porc_ing_ilpi_rur",
    "porc_ing_ilpi_urb",
    "porc_jub_urb",
    "porc_segsoc15",
    "plp15",
    "ictpc15",
    "pob_urb",
    "pob_tot"
  )

cov_names <- c(
  "modifica_humana",
  "acceso_hosp",
  "acceso_hosp_caminando",
  "cubrimiento_cultivo",
  "cubrimiento_urbano",
  "luces_nocturnas",
  variables_seleccionadas
)

cov_registros <-
  setdiff(
    cov_names,
    c(
      "elec_mun20",
      "elec_mun19",
      "transf_gobpc_15_20",
      "derhab_pea_15_20",
      "vabpc_15_19",
      "itlpis_15_20",
      "remespc_15_20",
      "desem_15_20",
      "porc_urb",
      "edad65mas_urb",
      "pob_tot",
      "acc_muyalto",
      "smg1",
      "ql_porc_cpa_rur",
      "ql_porc_cpa_urb"
    )
  )

cov_registros <- paste0(cov_registros, collapse = " + ")
```

#### Definición de la Fórmula del Modelo {-}

Se define la fórmula del modelo que será utilizada para el análisis. La fórmula especifica la estructura del modelo incluyendo los efectos aleatorios y las variables predictoras seleccionadas. En este caso, el modelo se formula para predecir una variable dependiente binaria con la estructura `cbind(si, no)`, que representa la proporción de casos positivos y negativos.

La fórmula incluye efectos aleatorios para las variables `cve_mun` (clave de municipio), `hlengua` (lengua indígena), y `discapacidad` (discapacidad), lo cual permite capturar la variabilidad no explicada a nivel de estos factores. Además, se incorporan variables fijas como `nivel_edu` (nivel educativo), `edad` (edad), `ent` (estado), `area` (área), y `sexo` (sexo), que se consideran relevantes para el modelo.

Las variables predictoras adicionales, definidas en el vector `cov_registros`, se añaden a la fórmula para completar el conjunto de variables explicativas del modelo. La fórmula resultante es utilizada para ajustar el modelo y analizar las relaciones entre las variables seleccionadas y la variable dependiente.


``` r
formula_model <-
  paste0(
    "cbind(si, no) ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad) +  nivel_edu + edad  + ent + area + sexo ",
    " + ",
    cov_registros
  )
```

#### Ajuste del Modelo {-}

El ajuste del modelo se realiza utilizando la función `modelo_dummy`, que se aplica a los datos para estimar un modelo basado en la fórmula definida anteriormente. En este paso, la variable dependiente `yk` se crea a partir del indicador `ic_segsoc`, donde se asigna un valor de 1 si el indicador es positivo y 0 en caso contrario. Esta transformación convierte el problema en una tarea de clasificación binaria.

La función `modelo_dummy` toma como entrada los datos de la encuesta (`encuesta_sta`), los predictores a nivel estatal (`statelevel_predictors_df`), los datos del censo (`censo_sta`), y la fórmula del modelo (`formula_model`). Además, utiliza el vector de variables de agregación (`byAgrega`) para realizar el ajuste del modelo.

Una vez ajustado el modelo, los resultados se guardan en un archivo RDS en la ruta especificada (`../output/2020/modelos/fit_mrp_ic_segsoc.rds`). Este archivo contiene el modelo ajustado, que puede ser utilizado para análisis posteriores o para generar predicciones basadas en los datos de entrada.


``` r
fit <- modelo_dummy(
  encuesta_sta = encuesta_sta %>%  mutate(yk = ifelse(ic_segsoc == 1 ,1,0)),
  predictors = statelevel_predictors_df,
  censo_sta = censo_sta,
  formula_mod = formula_model,
  byAgrega = byAgrega
)

#--- Exporting Bayesian Multilevel Model Results ---#

saveRDS(fit, file = "../output/2020/modelos/fit_mrp_ic_segsoc.rds")
```


