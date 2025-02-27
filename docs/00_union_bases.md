# 00_union_bases.R {-}



Para la ejecución del presente archivo, debe abrir el archivo **00_union_bases.R** disponible en la ruta *Rcodes/2020/00_union_bases.R*.

El código comienza limpiando el entorno de R y cargando varias bibliotecas esenciales para la manipulación y análisis de datos. Posteriormente, define un conjunto de variables a validar y obtiene listas de archivos de datos en formato `.dta` correspondientes a diferentes conjuntos de datos del censo 2020. A continuación, el código itera sobre estos archivos para leer, combinar y almacenar los datos de cada estado en archivos `.rds` individuales. Estos datos se combinan en un único dataframe que se guarda para su uso posterior.

En la sección final, el código se centra en los datos de la ENIGH 2020. Lee los archivos de hogares y pobreza, y los combina utilizando un `inner_join`. Se seleccionan y renombran variables clave para asegurar la consistencia en el análisis. Finalmente, el dataframe combinado se guarda en un archivo `.rds` para facilitar el acceso y análisis futuros.



#### Limpieza del Entorno y Carga de Bibliotecas {-}

Se limpia el entorno de R eliminando todos los objetos y se ejecuta el recolector de basura para liberar memoria.


``` r
rm(list = ls())
gc()
```

Se cargan varias bibliotecas esenciales para la manipulación y análisis de datos, incluyendo `tidyverse`, `data.table`, `haven` y otras.


``` r
library(tidyverse)
library(data.table)
library(openxlsx)
library(magrittr)
library(DataExplorer)
library(haven)
library(purrr)
library(furrr)
library(labelled)
cat("\f")
```


#### Configuración de la Memoria {-}
Se define un límite para la memoria RAM a utilizar, en este caso 250 GB.


``` r
memory.limit(250000000)
```


#### Definición de Variables y Obtención de Archivos{-}

Para llevar a cabo la validación y procesamiento de datos del censo 2020, se define un conjunto de variables que serán validadas. Estas variables incluyen indicadores clave como rezago educativo (`ic_rezedu`), acceso a servicios de salud (`ic_asalud`), seguridad social (`ic_segsoc`), calidad de la vivienda (`ic_cv`), servicios básicos en la vivienda (`ic_sbv`), acceso a alimentación nutritiva y de calidad (`ic_ali_nc`), y el índice de pobreza multidimensional (`ictpc`).


``` r
validar_var <- c(
  "ic_rezedu",
  "ic_asalud",
  "ic_segsoc",
  "ic_cv",
  "ic_sbv",
  "ic_ali_nc",
  "ictpc"
)
```

Posteriormente, se obtienen listas de archivos con extensión `.dta`, que corresponden a diferentes conjuntos de datos del censo 2020. Estos archivos se organizan en tres categorías principales:

1. **Archivos de muestra del censo 2020 por estado**: Se buscan archivos en la ruta `../input/2020/muestra_ampliada/SegSocial/SegSoc/` que contengan información de seguridad social.
  

``` r
file_muestra_censo_2020_estado <- list.files(
  "../input/2020/muestra_ampliada/SegSocial/SegSoc/",
  full.names = TRUE,
  pattern = "dta$"
)
```

2. **Archivos complementarios de muestra del censo 2020 por estado**: Se buscan archivos adicionales en la ruta 
`../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/` que contienen datos complementarios de seguridad social.


``` r
file_muestra_censo_2020_estado_complemento <- list.files(
  "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/",
  full.names = TRUE,
  pattern = "dta$"
)
```

3. **Archivos de cuestionario ampliado del censo 2020 por estado**: Se buscan archivos en la ruta `../input/2020/muestra_ampliada/IndicadoresCenso/` que incluyen indicadores generales del censo.


``` r
muestra_cuestionario_ampliado_censo_2020_estado <- list.files(
  "../input/2020/muestra_ampliada/IndicadoresCenso/",
  full.names = TRUE,
  pattern = "dta$"
)
```

Estas listas de archivos permiten acceder y gestionar de manera eficiente los datos necesarios para el análisis y validación de los indicadores de pobreza y carencias sociales del censo 2020.


#### Lectura y Combinación de Datos del Censo{-}

Este bloque de código está diseñado para leer, combinar y guardar datos provenientes de múltiples archivos del censo. El objetivo es consolidar la información de diferentes fuentes en un único dataframe y almacenar los resultados intermedios y finales en archivos `.rds`.



``` r
df <- data.frame()

for (ii in 1:32) {
  muestra_censo_2020_estado_ii <-
    read_dta(file_muestra_censo_2020_estado[ii])
  muestra_censo_2020_estado_complemento_ii <-
    read_dta(file_muestra_censo_2020_estado_complemento[ii]) %>%
    mutate(id_per = id_persona)
  muestra_cuestionario_ampliado_censo_2020_estado_ii <-
    read_dta(muestra_cuestionario_ampliado_censo_2020_estado[ii])
  
  muestra_censo <-
    inner_join(muestra_censo_2020_estado_ii,
               muestra_censo_2020_estado_complemento_ii) %>%
    select(-tamloc) %>%
    inner_join(muestra_cuestionario_ampliado_censo_2020_estado_ii)
  
  saveRDS(muestra_censo,
          paste0("../output/2020/muestra_censo/depto_", ii, ".rds"))
  
  df <- bind_rows(df, muestra_censo)
  cat(file_muestra_censo_2020_estado[ii], "\n")
}
```

Se guarda el dataframe combinado en un archivo `.rds`.


``` r
saveRDS(df, file = "../output/2020/muestra_cuestionario_ampliado.rds")
```


#### Lectura y Combinación de Datos de la ENIGH{-}

Este bloque de código está diseñado para leer, verificar y combinar datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH) 2020. La finalidad es consolidar la información de diferentes archivos de datos en un único dataframe para su posterior análisis.


``` r
enigh_hogares <- read_dta("../input/2020/enigh/base_hogares20.dta")
enigh_pobreza <- read_dta("../input/2020/enigh/pobreza_20.dta") %>% 
  select(-ent)
```
Se leen los datos de hogares y pobreza de la ENIGH 2020.


``` r
n_distinct(enigh_pobreza$folioviv)
n_distinct(enigh_hogares$folioviv)
```
Se verifica la unicidad de los identificadores de viviendas.


``` r
enigh <- inner_join(
  enigh_pobreza,
  enigh_hogares,
  by = join_by(
    folioviv, foliohog, est_dis, upm,
    factor, rururb, ubica_geo
  ), 
  suffix = c("_pers", "_hog")
)
```
Se combinan los datos de pobreza y hogares utilizando un `inner_join`.

#### Selección y Renombramiento de Variables {-}

En esta sección, se seleccionan y renombran algunas variables clave del dataframe combinado para asegurar la consistencia en el análisis y facilitar su manipulación posterior.



``` r
   enigh$ic_ali_nc
   enigh$ictpc_pers
   enigh$ictpc <- enigh$ictpc_pers
   enigh$ic_segsoc
```
   - `enigh$ic_ali_nc`: Selección de la variable que indica carencia por acceso a la alimentación nutritiva y de calidad.
   - `enigh$ictpc_pers`: Selección de la variable que representa el ingreso corriente total por persona.
   - `enigh$ictpc <- enigh$ictpc_pers`: Renombramiento de `ictpc_pers` a `ictpc` para simplificar su uso en análisis posteriores.
   - `enigh$ic_segsoc`: Selección de la variable que indica carencia por acceso a la seguridad social.

#### Guardado del DataFrame Combinado {-}

Finalmente, se guarda el dataframe combinado en un archivo `.rds` para facilitar el acceso y análisis futuros.



``` r
   saveRDS(enigh, file = "../output/2020/enigh.rds")
```
Se guarda el dataframe `enigh` en un archivo `.rds` en la ruta especificada `../output/2020/enigh.rds`.

