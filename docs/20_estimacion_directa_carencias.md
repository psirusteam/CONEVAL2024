# 20_estimacion_directa_carencias.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **20_estimacion_directa_carencias.R** disponible en la ruta *Rcodes/2020/20_estimacion_directa_carencias.R*.

##### Descripción General del Código {-}

Este código está diseñado para procesar y analizar datos de encuestas con el objetivo de generar estimaciones de carencia para diferentes entidades dentro de un país. A continuación, se detalla cada sección del código:

##### Preparación del Entorno {-}

Al inicio, el código limpia el entorno de trabajo de R mediante el siguiente bloque de código:


``` r
rm(list = ls())
```

Esto elimina todos los objetos existentes en el espacio de trabajo, asegurando que el análisis se realice en un entorno limpio y sin interferencias de datos anteriores.

##### Carga de Bibliotecas {-}

Se cargan varias bibliotecas necesarias para el análisis de datos mediante el bloque de código:


``` r
library(tidyverse)
library(data.table)
library(openxlsx)
library(magrittr)
library(haven)
library(labelled)
library(sampling)
library(lme4)
library(survey)
library(srvyr)
```

Estas bibliotecas proporcionan herramientas para la manipulación de datos, manejo de archivos, técnicas de muestreo y análisis de encuestas.

##### Lectura y Preparación de Datos {-}

Se lee un archivo de datos en formato RDS que contiene la encuesta ampliada y se prepara la encuesta intercensal con el siguiente código:


``` r
encuesta_ampliada <- readRDS("output/2020/encuesta_ampliada.rds")

encuesta_ampliada %<>% mutate(
  ic_asalud = ifelse(ic_asalud == 1, 1,0),
  ic_cv = ifelse(ic_cv == 1, 1,0),
  ic_sbv = ifelse(ic_sbv == 1, 1,0),
  ic_rezedu = ifelse(ic_rezedu == 1, 1,0)
) 
```

Aquí, se ajustan ciertos indicadores para que solo tomen valores binarios (0 o 1).

##### Creación de Directorios y Definición de Variables {-}

Se crea un directorio para almacenar los resultados de estimación y se define un vector con códigos de entidades:


``` r
dir.create("output/2020/estimacion_dir")

c("03", "06", "23", "04", "01", "22", "27", "25", "18",
  "05", "17", "28", "10", "26", "09", "32", "08", "19", "29", 
  "24", "11", "31", "13", "16", "12", "14", "15", "07", "21", 
  "30",  "20" ,"02")
```



##### Análisis y Estimación {-}

Se realiza un bucle `for` que itera sobre cada código de entidad con el siguiente bloque de código:


``` r
for(ii_ent in c("03", "06", "23", "04", "01", "22", "27", "25", "18",
  "05", "17", "28", "10", "26", "09", "32", "08", "19", "29", 
  "24", "11", "31", "13", "16", "12", "14", "15", "07", "21", 
  "30",  "20" ,"02" )){
  
  cat("################################################################################################################\n")
  inicio <- Sys.time()
  print(inicio)
  
  muestra_post = encuesta_ampliada %>% filter(ent == ii_ent)
  
  cat("\n Estado = ", ii_ent,"\n\n")

  diseno_post <-  muestra_post %>% 
    mutate(fep = factor) %>%
    as_survey_design(
      ids = upm,
      weights = fep,
      nest = TRUE,
      # strata = estrato
    )
  
  estima_carencia <-  diseno_post %>% group_by(cve_mun) %>% 
    summarise_at(.vars = yks, .funs = list(
      ~ survey_mean(., vartype ="var", na.rm = TRUE))
    )
  
  saveRDS(estima_carencia,
          file = paste0( "output/2020/estimacion_dir/estado_",
                         ii_ent,".rds"))
  gc(TRUE)
  fin <- Sys.time()
  tiempo_total <- difftime(fin, inicio, units = "mins")
  print(tiempo_total)
  cat("################################################################################################################\n")
}
```
