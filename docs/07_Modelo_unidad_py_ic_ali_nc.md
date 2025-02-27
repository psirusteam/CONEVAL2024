# 07_Modelo_unidad_py_ic_ali_nc.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **07_Modelo_unidad_py_ic_ali_nc.R** disponible en la ruta *Rcodes/2020/07_Modelo_unidad_py_ic_ali_nc.R*.

Este script en R se centra en la creación de un modelo  multinivel para predecir la carencia en  alimentos nutritivos y de calidad , utilizando datos de encuestas y censos. Comienza limpiando el entorno de trabajo (`rm(list = ls())`) y cargando las librerías necesarias  para la integración con Python. Además, se importan los módulos `pandas` y las bibliotecas `sklearn.feature_selection` y `sklearn.ensemble` de Python. También se carga una fuente externa de modelos (`source("source/modelos_freq.R")`). Luego, se definen las variables para la agregación (`byAgrega`) y se aumenta el límite de memoria disponible. Las bases de datos necesarias (`encuesta_sta`, `censo_sta` y `statelevel_predictors_df`) se cargan, y se seleccionan las variables covariantes relevantes excluyendo aquellas que comienzan con `hog_` o `cve_mun`.

El script incluye un paso comentado para la selección de características utilizando el método RFE de `caret` y Python, pero en esta versión, se listan explícitamente las variables seleccionadas (`variables_seleccionadas`) y se combinan con otras covariantes para formar `cov_names`. Luego, se construye la fórmula del modelo (`formula_model`), que incluye efectos aleatorios para `cve_mun`, `hlengua` y `discapacidad`, así como efectos fijos para las demás variables. Se ajusta el modelo utilizando la función `modelo_dummy`, que está definida en el archivo fuente cargado previamente. Los datos de entrada incluyen `encuesta_sta` (con una nueva variable `yk`), `statelevel_predictors_df`, `censo_sta` y la fórmula del modelo. Este enfoque permite una modelización robusta y ajustada a las especificaciones de los datos disponibles.

Finalmente, los resultados del modelo se guardan en un archivo RDS (`fit_mrp_ic_ali_nc.rds`). Este paso asegura que los resultados del análisis estén disponibles para futuras referencias y análisis adicionales. Guardar los resultados en un archivo RDS facilita su recuperación y análisis posterior, permitiendo a los investigadores y analistas continuar trabajando con los resultados sin necesidad de recalcular el modelo cada vez. Además, se generan gráficos de densidad y histogramas de las distribuciones posteriores utilizando ggsave, lo que proporciona una visualización clara y comprensible de los resultados del modelo.


#### Limpieza del Entorno y Carga de Bibliotecas{-}

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

source("../source/modelos_freq.R")
```

#### Carga de Datos {-}

En esta fase, se procede a la carga de los datos necesarios para el análisis. Se utilizan las funciones `readRDS()` para leer archivos en formato RDS, que es un formato de almacenamiento de objetos en R.

1. **Carga de la Encuesta:** Se carga el conjunto de datos de la encuesta, almacenado en el archivo `../input/2020/enigh/encuesta_sta.rds`. Este conjunto de datos contiene información sobre las variables de interés para el análisis.

2. **Carga del Censo:** Se lee el archivo `../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds`, que contiene datos censales ampliados. Estos datos se usarán para la calibración y comparación con los datos de la encuesta.

3. **Carga de Predictores Estatales:** Se carga el archivo `../input/2020/predictores/statelevel_predictors_df.rds`, que incluye predictores a nivel estatal que se utilizarán para el análisis.

Además, se extraen los nombres de las variables del dataframe `statelevel_predictors_df` y se filtran para eliminar las variables que comienzan con "hog_" o "cve_mun", asegurando que solo se conserven las variables relevantes para el análisis posterior. Esto se realiza mediante la función `grepl()` para identificar y excluir las variables no deseadas.


``` r
# Loading data ------------------------------------------------------------

memory.limit(10000000)
encuesta_sta <- readRDS("../input/2020/enigh/encuesta_sta.rds")
censo_sta <- readRDS("../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")
statelevel_predictors_df <- readRDS("../input/2020/predictores/statelevel_predictors_df.rds")

cov_names <- names(statelevel_predictors_df)
cov_names <- cov_names[!grepl(x = cov_names,pattern = "^hog_|cve_mun")]
```

#### Selección de Variables {-}

En esta etapa, se definen y seleccionan las variables que serán utilizadas en el análisis posterior. 

1. **Variables de Agregación:** Se establece un vector llamado `byAgrega` que contiene las variables de agregación. Estas variables son fundamentales para estructurar y resumir los datos de acuerdo a diferentes dimensiones y características. El vector incluye:
   - `ent` (Entidad Federativa)
   - `cve_mun` (Clave de Municipio)
   - `area` (Área)
   - `sexo` (Sexo)
   - `edad` (Edad)
   - `discapacidad` (Discapacidad)
   - `hlengua` (Lengua Indígena)
   - `nivel_edu` (Nivel Educativo)

2. **Selección de Variables Predictoras:** Se identifican las variables predictoras más relevantes para el análisis. En el código comentado, se había planificado usar un modelo de Random Forest y el método RFE (Recursive Feature Elimination) de Python para seleccionar las variables predictoras más importantes. Sin embargo, el código relacionado con la integración de Python se encuentra comentado.

En lugar de ello, se definen directamente las variables seleccionadas y se especifica un vector `cov_names` que incluye las variables predictoras relevantes para el análisis. Estas variables son:
   - `modifica_humana` (Modificación Humana)
   - `acceso_hosp` (Acceso a Hospital)
   - `acceso_hosp_caminando` (Acceso a Hospital Caminando)
   - `cubrimiento_cultivo` (Cobertura de Cultivo)
   - `cubrimiento_urbano` (Cobertura Urbana)
   - `luces_nocturnas` (Luces Nocturnas)
   - Variables seleccionadas como `porc_ing_ilpi_urb`, `pob_ind_rur`, `pob_ind_urb`, `porc_hogremesas_rur`, `porc_segsoc15`, `porc_ali15`, `plp15`, `pob_rur`, y `altitud1000`.

Estos pasos aseguran que se utilicen las variables más relevantes para el análisis, optimizando la capacidad predictiva y el rendimiento del modelo.


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
#   yk = as.factor(ifelse(ic_ali_nc == 1 ,1,0))) %>%
#   inner_join(statelevel_predictors_df[, c("cve_mun",cov_names)])
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
# # Obtener las variables seleccionadas
# variables_seleccionadas <- X[selector$support_] %>% names()

variables_seleccionadas <- c(
  "porc_ing_ilpi_urb",
  "pob_ind_rur",
  "pob_ind_urb",
  "porc_hogremesas_rur",
  "porc_segsoc15",
  "porc_ali15",
  "plp15",
  "pob_rur",
  "altitud1000"
)
cov_names <- c(
  "modifica_humana",  "acceso_hosp",           
  "acceso_hosp_caminando",  "cubrimiento_cultivo",   
  "cubrimiento_urbano",     "luces_nocturnas" ,
  variables_seleccionadas
)
```

#### Definición de la Fórmula del Modelo {-}


Se define una fórmula para el modelo que incluye tanto efectos aleatorios como variables predictoras seleccionadas. Primero, se determina el conjunto de variables relevantes excluyendo algunas específicas que no se incluirán en el análisis. Estas exclusiones se hacen utilizando la función `setdiff`, que elimina variables como `elec_mun20`, `elec_mun19`, `transf_gobpc_15_20`, y otras relacionadas con transferencias, electrificación, y características del desempeño y calidad. 

El vector resultante de variables seleccionadas se concatena en una cadena de texto para construir la fórmula del modelo. Esta fórmula se configura para ajustar el modelo a datos binomiales, utilizando `cbind(si, no)` como la variable dependiente. La fórmula también incorpora efectos aleatorios para la clave de municipio `(1 | cve_mun)`, la lengua indígena `(1 | hlengua)`, y la discapacidad `(1 | discapacidad)`. Además, incluye efectos fijos para `nivel_edu`, `edad`, `ent`, `area`, y `sexo`. Finalmente, se agregan las variables predictoras seleccionadas.



``` r
cov_registros <-
  setdiff(
    cov_names,
    c(
      "elec_mun20",
      "elec_mun19",
      "transf_gobpc_15_20",
      "derhab_pea_15_20",
      "vabpc_15_19" ,
      "itlpis_15_20"  ,
      "remespc_15_20",
      "desem_15_20",
      "porc_urb" ,  
      "edad65mas_urb", 
      "pob_tot" ,
      "acc_muyalto" ,
      "smg1",
      "ql_porc_cpa_rur",
      "ql_porc_cpa_urb"
    )
  )

cov_registros <- paste0(cov_registros, collapse = " + ")
```
La fórmula completa se define como:


``` r
formula_model <-
  paste0(
    "cbind(si, no) ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad) +  nivel_edu + edad  + ent + area + sexo "
    ,
    " + ",
    cov_registros
  )
```

Esta estructura permite capturar la variabilidad entre los municipios y otras dimensiones de la población, mientras que también evalúa el impacto de las variables predictoras sobre la variable dependiente.

#### Ajuste del Modelo {-}

Se ajusta el modelo utilizando la función `modelo_dummy`, que se aplica a los datos de la encuesta junto con los predictores y el censo. Primero, se transforma la variable `ic_ali_nc` en una variable binaria `yk`, donde se asigna el valor 1 si `ic_ali_nc` es igual a 1, y 0 en caso contrario. Esta transformación permite modelar la variable como un resultado binomial en el análisis.

Luego, se pasa el conjunto de datos transformado (`encuesta_sta` con la nueva variable `yk`), los predictores a nivel estatal (`statelevel_predictors_df`), y el censo estatal (`censo_sta`) a la función `modelo_dummy`, junto con la fórmula del modelo previamente definida y el vector `byAgrega` para la agregación.

Finalmente, se guarda el resultado del ajuste del modelo en un archivo RDS en la carpeta especificada, utilizando el nombre `fit_mrp_ic_ali_nc.rds`. Esto permite almacenar los resultados del modelo para su posterior análisis o uso.


``` r
fit <- modelo_dummy(
  encuesta_sta = encuesta_sta %>%  
    mutate(yk = ifelse(ic_ali_nc == 1 , 1, 0))  ,
  predictors = statelevel_predictors_df,
  censo_sta = censo_sta,
  formula_mod = formula_model,
  byAgrega = byAgrega
)

#--- Exporting Bayesian Multilevel Model Results ---#

saveRDS(fit, 
        file = "../output/2020/modelos/fit_mrp_ic_ali_nc.rds")
```

