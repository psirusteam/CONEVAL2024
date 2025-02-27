# 04_Armonizar_encuesta.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **04_Armonizar_encuesta.R** disponible en la ruta *Rcodes/2020/04_Armonizar_encuesta.R*. Este script está diseñado para llevar a cabo un análisis exhaustivo de datos a través de un proceso metódico en varias fases.

En la primera fase, el código elimina todos los objetos del entorno de R para asegurar que el análisis se inicie con un entorno limpio y sin datos residuales. A continuación, se cargan las bibliotecas necesarias para el análisis, que proporcionan herramientas esenciales para la manipulación de datos y la importación y exportación de archivos. Después, se leen las bases de datos del censo de personas 2020 y de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH). Estas bases se utilizan para identificar y verificar indicadores relacionados con carencias sociales.

Posteriormente, se definen y validan las variables clave de la encuesta, como códigos de entidad, municipio, área, sexo, edad, nivel educativo, discapacidad y lengua indígena. Luego, se estandarizan las variables de la encuesta para alinear los datos con los del censo, se filtran los datos para eliminar inconsistencias y se guarda el conjunto de datos estandarizado. Además, se actualiza la tabla censal utilizando la técnica de calibración IPFP (Iterative Proportional Fitting Procedure) para asegurar que las distribuciones marginales de las variables estandarizadas coincidan con las del censo. Finalmente, se generan histogramas y gráficos de dispersión para visualizar los resultados de la calibración y se guarda el conjunto de datos ampliado.


#### Limpieza del Entorno y Carga de Bibliotecas {-}

El código presentado realiza dos tareas principales: limpiar el entorno de trabajo en R y cargar las bibliotecas necesarias para el análisis de datos.

La primera parte del código se encarga de eliminar todos los objetos en el entorno de trabajo de R y ejecutar el recolector de basura para liberar memoria. Esta operación es esencial antes de comenzar un nuevo análisis para asegurarse de que el entorno esté limpio y libre de datos o objetos antiguos que puedan interferir con el nuevo análisis.


``` r
### Cleaning R environment ###
rm(list = ls())
gc()
```

La segunda parte del código se enfoca en cargar una serie de bibliotecas cruciales para el análisis de datos. Estas bibliotecas proporcionan diversas funcionalidades que facilitan la manipulación, visualización y análisis de datos. 

- `tidyverse`: Un conjunto de paquetes para la manipulación y visualización de datos.
- `data.table`: Paquete para la manipulación eficiente de grandes conjuntos de datos.
- `openxlsx`: Herramienta para leer y escribir archivos de Excel.
- `magrittr`: Proporciona el operador de tubería (`%>%`) para encadenar operaciones.
- `DataExplorer`: Paquete para la exploración rápida de datos.
- `haven`: Permite leer y escribir datos en formatos de Stata, SPSS y SAS.
- `purrr`: Facilita la programación funcional con listas y vectores.
- `labelled`: Trabaja con datos etiquetados, especialmente útil para datos de encuestas.
- `sampling`: Proporciona métodos para la toma de muestras estadísticas.


``` r
library(tidyverse)
library(data.table)
library(openxlsx)
library(magrittr)
library(DataExplorer)
library(haven)
library(purrr)
library(labelled)
library(sampling)
cat("\f")
```

La función `cat("\f")` se utiliza para limpiar la pantalla de la consola en R, proporcionando una interfaz más ordenada para el usuario.

#### Lectura de Datos {-}

En esta sección, se realiza la lectura de dos conjuntos de datos cruciales para el análisis y se lleva a cabo una verificación para asegurar la unicidad de los identificadores.

Primero, se cargan las bases de datos `muestra_cuestionario_ampliado.rds` y `enigh.rds` desde los archivos correspondientes. La base `muestra_cuestionario_ampliado.rds` contiene información ampliada del cuestionario, mientras que `enigh.rds` incluye datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares.


``` r
encuesta_ampliada <- readRDS("../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")
enigh <- readRDS("output//2020/enigh.rds")
```


A continuación, se verifica la unicidad de los identificadores de entidad y municipio en la base de datos `enigh`. Esta verificación es importante para asegurar que los datos estén correctamente identificados y evitar errores en análisis posteriores que puedan surgir de duplicados o inconsistencias en los identificadores.


``` r
n_distinct(enigh$ent) # cod_dam
n_distinct(enigh$cve_mun) # cod_mun
```

La función `n_distinct()` se utiliza para contar el número de identificadores únicos en las columnas `ent` y `cve_mun`. Este paso garantiza que cada entidad y municipio tenga un identificador único en los datos, lo cual es crucial para la integridad y precisión del análisis.

#### Definición de Variables para la Encuesta {-}

Se definen y transforman las variables relacionadas con el área urbana o rural en la base de datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH). Primero, se revisa la distribución de la variable `rururb` para identificar el número de registros en cada categoría de área urbana o rural. Posteriormente, se convierte `rururb` en una variable de factor con niveles definidos por la variable "values" usando la función `as_factor()` del paquete `haven`, y luego se transforma a formato de carácter para simplificar su uso en análisis posteriores. Finalmente, se verifica que la conversión se haya realizado correctamente, asegurando que los valores de `rururb` y `area` sean consistentes. 


``` r
# codigo municipio

# area urabo o rural 
enigh %>% group_by(rururb) %>% summarise(n = sum(factor))

enigh %<>% mutate(
  area = haven::as_factor(rururb, levels  = "values"),
  area = as.character(area)
)

enigh %>%   distinct(rururb,area)
```


#### Transformación de Variables Demográficas y Socioeconómicas {-}


Se realizan transformaciones en las variables demográficas y socioeconómicas de la base de datos `enigh` para prepararlas para el análisis. Primero, se revisa la distribución de la variable de `sexo`, que tiene dos categorías: "2" para mujeres y "1" para hombres. Luego, se examina la variable de `edad`, transformando las edades en grupos de edad específicos: "1" para menores de 14 años, "2" para 15 a 29 años, "3" para 30 a 44 años, "4" para 45 a 64 años, y "5" para 65 años o más. La transformación se realiza utilizando la función `case_when()`.

En cuanto al `nivel educativo`, se revisa la distribución actual y se transforma en un factor con niveles específicos definidos por la variable "values", asegurando que los valores sean consistentes y descriptivos. Posteriormente, se realiza la misma transformación para la variable `discapacidad`, convirtiéndola en un factor con niveles apropiados y luego a formato de carácter para su uso en el análisis.

Finalmente, para la variable `hli`, que indica si se habla algún dialecto o lengua indígena, se transforma en un factor con niveles definidos, asegurando la consistencia en la representación de esta variable. Las transformaciones aseguran que todas las variables sean adecuadas para el análisis, con valores claros y uniformes.



``` r
# Sexo 
enigh %>% group_by(sexo) %>% summarise(n = sum(factor))
# 2	Mujer
# 1	Hombre

# Edad 
enigh %>% group_by(edad) %>% summarise(n = sum(factor))
encuesta_ampliada %>% distinct(edad)
# 1	Menor de 14 años
# 2	15 a 29 años
# 3	30 a 44 años
# 4	45 a 64 años
# 5	65 años o más

enigh %<>% mutate(g_edad = case_when(edad <= 14 ~ "1", # 0 a 14
                                     edad <= 29 ~ "2", # 15 a 29
                                     edad <= 44 ~ "3", # 30 a 44
                                     edad <= 64 ~ "4", # 45 a 64
                                     edad >  64 ~ "5", # 65 o mas
                                     TRUE ~ NA_character_)) 
enigh %>% group_by(g_edad) %>% summarise(n = sum(factor))

# nivel educativo

enigh %>% group_by(niv_ed) %>%
  summarise(n = sum(factor), .groups = "drop") %>%
  mutate(prop = n / sum(n))

enigh %>% 
  mutate(nivel_edu = haven::as_factor(niv_ed, levels  = "values")) %>%
  group_by(nivel_edu, g_edad) %>% summarise(n = sum(factor)) %>%
  data.frame()

enigh %<>% mutate(
  nivel_edu = haven::as_factor(niv_ed, levels  = "values"),
  nivel_edu = as.character(nivel_edu)
)

enigh %>%   distinct(niv_ed,nivel_edu)

# Discapacidad

enigh %>% group_by(discap) %>% summarise(n = sum(factor)) 

enigh %<>% mutate(
  discapacidad = haven::as_factor(discap, levels  = "values"),
  discapacidad = as.character(discapacidad)
)

enigh %>%   distinct(discap,discapacidad)

# Habla dialecto o lengua indígena
attributes(enigh$hli)

enigh %>% group_by(hli) %>% summarise(n = sum(factor)) 

enigh %<>% mutate(
  hlengua = haven::as_factor(hli, levels  = "values"),
  hlengua = as.character(hlengua)
)

enigh %>%   distinct(hlengua,hli)
```



#### Análisis de Variables de Carencia {-}

Para el análisis de variables de carencia, se realiza lo siguiente:

1. **Distribución Logarítmica del Ingreso Per Cápita**: Se visualiza la distribución del ingreso per cápita (`ictpc`) mediante un histograma en escala logarítmica. Esta transformación es útil para observar la distribución de datos con alta variabilidad y para identificar patrones o anomalías en la distribución del ingreso.

2. **Indicadores de Carencia**:
   - **Acceso a Servicios de Salud**: Se revisa el atributo de la variable `ic_segsoc`, que representa el indicador de carencia por acceso a servicios de salud.
   - **Acceso a Alimentación Nutritiva y de Calidad**: Se revisa el atributo de la variable `ic_ali_nc`, que mide la carencia en el acceso a alimentación nutritiva y de calidad.

3. **Distribución de Carencia por Acceso a Servicios de Salud**: Se agrupan los datos por la variable `ic_segsoc` (carencia por acceso a servicios de salud) y se calcula la suma del factor en cada grupo para analizar la distribución de esta carencia.



``` r
hist(log(enigh$ictpc))

# Indicador de carencia por acceso a servicios de salud
attributes(enigh$ic_segsoc)

# Indicador de carencia por acceso a la alimentación nutritiva y de calidad
attributes(enigh$ic_ali_nc)

enigh %>% group_by(ic_segsoc) %>% summarise(n = sum(factor)) 
```

#### Preparación y Guardado del Conjunto de Datos {-}

El código realiza un proceso exhaustivo para preparar y guardar un conjunto de datos final denominado `encuesta_sta`. Este conjunto de datos se crea a partir de la base de datos `enigh`, aplicando transformaciones y ajustes necesarios para garantizar que los datos estén en el formato adecuado para el análisis posterior.

En primer lugar, el código transforma varias variables clave. La variable `ent`, que representa la entidad, se ajusta para asegurar que siempre tenga dos caracteres, añadiendo ceros a la izquierda si es necesario. Las variables relacionadas con la edad, el nivel educativo, la discapacidad y la lengua indígena se procesan para reemplazar los valores faltantes con valores predeterminados específicos, como "99" para las edades no especificadas y "0" para la discapacidad y la lengua indígena no declarada. Además, las variables relacionadas con las carencias (`ictpc`, `ic_segsoc`, `ic_ali_nc`, `ic_rezedu`, `ic_asalud`, `ic_sbv`, `ic_cv`) se convierten a tipo numérico para facilitar el análisis cuantitativo.

El conjunto de datos también incluye variables de diseño, como `estrato`, `upm`, y `fep`, que son esenciales para el análisis de diseño y ponderación. Estas variables ayudan a asegurar que los resultados del análisis reflejen correctamente la estructura de la muestra y las características del estudio.

Posteriormente, se realiza un análisis de frecuencias para cada una de las variables clave. Esto incluye calcular la frecuencia absoluta y la proporción relativa para variables como el código de municipio, área (urbano o rural), sexo, edad, nivel educativo, discapacidad, y lengua indígena, así como para los indicadores de carencia. Este análisis proporciona una visión general de la distribución de los datos y permite verificar la representatividad y la calidad del conjunto de datos.

Finalmente, el conjunto de datos transformado y analizado se guarda en un archivo con formato `.rds`. Este archivo es ahora accesible para su uso en análisis posteriores, garantizando que los datos están bien preparados y que todas las transformaciones necesarias se han aplicado correctamente.


``` r
################################################################################
encuesta_sta <- enigh %>%
  transmute(
    ent = str_pad(
      string = ent,
      width = 2,
      pad = "0"
    ),
    cve_mun,
    area,
    sexo,
    edad = ifelse(is.na(g_edad), "99", g_edad),
    nivel_edu = ifelse(is.na(nivel_edu), "99", nivel_edu),
    discapacidad = ifelse(is.na(discapacidad), "0", discapacidad),
    hlengua = ifelse(is.na(hlengua), "0", hlengua),
    
    ## Variable de estudio
    ictpc = as.numeric(ictpc),
    ic_segsoc = as.numeric(ic_segsoc),
    ic_ali_nc = as.numeric(ic_ali_nc),
    ic_rezedu = as.numeric(ic_segsoc),
    ic_asalud = as.numeric(ic_segsoc),
    ic_sbv = as.numeric(ic_sbv_hog),
    ic_cv = as.numeric(ic_cv_hog),
    
    ## Variables diseño
    estrato = est_dis,
    upm = upm,
    fep = factor
  )

map(c(
  "ent",
  "cve_mun",
  "area",
  "sexo",
  "edad",
  "nivel_edu",
  "hlengua",
  "discapacidad",
  "ic_segsoc",
  "ic_ali_nc"
),
function(x) {
  encuesta_sta %>% group_by_at(x ) %>%
    summarise(Nd = sum(fep)) %>% 
    mutate(N = sum(Nd),
           prop = Nd / N)
})

# Se guarda el conjunto de datos
saveRDS(encuesta_sta, file = "../input/2020/enigh/encuesta_sta.rds")
```


#### Actualización de la Tabla Censal{-}

El proceso de actualización de la tabla censal mediante la calibración de pesos es crucial para asegurar que los datos reflejen de manera precisa la estructura de la población y las características de la muestra. A continuación, se describe el proceso detallado llevado a cabo en el código:

En primer lugar, se identifican y seleccionan las covariables que están presentes tanto en la base de datos `encuesta_sta` como en la base de datos `encuesta_ampliada`. Esto se realiza mediante la comparación de los nombres de las covariables en ambas bases de datos y asegurando que solo se consideren aquellas con niveles completos en ambas tablas. Esta selección es fundamental para evitar problemas derivados de niveles incompletos en las covariables, lo que podría afectar la precisión de la calibración.

Luego, se utiliza una función auxiliar (`auxSuma`) para calcular las sumas ponderadas de las covariables. Esta función convierte las variables categóricas en variables dummy, multiplica cada variable dummy por su peso correspondiente, y luego calcula la suma total para cada categoría. Este cálculo se realiza tanto para la muestra (`encuesta_sta`) como para el censo (`encuesta_ampliada`), permitiendo una comparación directa entre los valores estimados en la muestra y los valores esperados en la población censal.

La comparación de las sumas ponderadas permite calibrar los pesos de la muestra para que se ajusten a las características observadas en el censo. Los valores obtenidos para cada categoría en la muestra (`N.g`) se comparan con los valores correspondientes en el censo (`N_censo.g`). Esta calibración asegura que la muestra sea representativa de la población en términos de las covariables seleccionadas, mejorando la precisión y la fiabilidad de los análisis posteriores basados en esta tabla censal actualizada.

Finalmente, se guarda el conjunto de datos calibrado, lo cual es esencial para que los análisis posteriores se realicen utilizando datos ajustados y representativos. Este proceso de calibración es un paso crucial en el manejo de datos de encuestas y censos, ya que permite corregir desviaciones y mejorar la precisión de las estimaciones derivadas de la muestra.


``` r
# Actualización de tabla censal- IPFP -------------------------------------
names_cov <- c("ent", "cve_mun", "area", "sexo", "edad", "discapacidad", "hlengua")
names_cov <- names_cov[names_cov %in% names(encuesta_sta)]

num_cat_censo <- apply(encuesta_ampliada[names_cov], MARGIN  = 2, function(x)
  length(unique(x)))

num_cat_sample <- apply(encuesta_sta[names_cov], MARGIN  = 2, function(x)
  length(unique(x)))

names_cov <- names_cov[num_cat_censo == num_cat_sample]

# MatrizCalibrada creada únicamente para los niveles completos 
# IMPORTANTE: Excluir las covariables que tengan niveles incompletos

auxSuma <- function(dat, col, ni) {
  dat %>% ungroup() %>% select(all_of(col))  %>%
    fastDummies::dummy_cols(remove_selected_columns = TRUE) %>% 
    mutate_all(~.*ni) %>% colSums()  
}

N.g <- map(names_cov,
           ~ auxSuma(encuesta_sta, col = .x, ni = encuesta_sta$fep)) %>%
  unlist()

N_censo.g <- map(names_cov,
                 ~ auxSuma(encuesta_ampliada, col = .x, ni = encuesta_ampliada$n)) %>%
  unlist()

names_xk <- intersect(names(N.g), names(N_censo.g))

N.g <- N.g[names_xk]
N_censo.g <- N_censo.g[names_xk]
```

El proceso de calibración y ajuste de pesos en la tabla censal es una etapa crítica para asegurar que los datos reflejen fielmente la estructura de la población. A continuación, se detalla el flujo de trabajo descrito en el código:

1. **Generación de Variables Dummy:** Primero, se crea un conjunto de datos con variables dummy para las covariables seleccionadas (`names_cov`). Esto se realiza usando la función `fastDummies::dummy_cols`, que transforma las variables categóricas en variables dummy, facilitando así la calibración posterior. Se seleccionan únicamente las variables de interés (`names_xk`) que coinciden entre la muestra y el censo.

2. **Calibración de Pesos:** Utilizando la función `calib`, se calibra la muestra (`Xk`) para que los totales ponderados coincidan con los valores esperados en la población censal (`N.g`). La calibración se realiza mediante el método lineal, que ajusta los pesos de manera que las sumas ponderadas en la muestra se alineen con los totales esperados en la población.

3. **Verificación de Calibración:** Se verifica la calidad de la calibración mediante la función `checkcalibration`, que compara los totales ajustados con los totales esperados. Esta verificación es crucial para asegurarse de que el proceso de calibración ha sido efectivo.

4. **Análisis de Pesos Calibrados:** Se realiza un análisis de la distribución de los pesos calibrados (`gk`) mediante un histograma y un resumen estadístico. Esto permite evaluar la variabilidad y la distribución de los pesos ajustados.

5. **Ajuste de Conteos Censales:** Los pesos calibrados se aplican a los conteos censales (`n1`) para obtener los conteos ajustados. Se compara gráficamente el conteo censal original con el ajustado mediante un gráfico de dispersión, donde la línea roja discontinua representa una relación 1:1. Esto ayuda a visualizar cómo los conteos censales han cambiado después de la calibración.

6. **Actualización de Datos:** Finalmente, se actualizan los conteos en la base de datos `encuesta_ampliada` con los valores ajustados y se guarda el conjunto de datos calibrado en un archivo `.rds`. Este archivo contiene la tabla censal actualizada, lista para ser utilizada en análisis posteriores.

Este proceso asegura que los datos censales reflejen de manera precisa la estructura poblacional y corrige cualquier desviación en los conteos, mejorando la calidad y la representatividad de la información.


``` r
Xk <- encuesta_ampliada %>% ungroup() %>% select(all_of(names_cov)) %>%
  fastDummies::dummy_cols(remove_selected_columns = TRUE) %

>% 
  select(all_of(names_xk))

gk <- calib(Xs = Xk, 
            d = encuesta_ampliada$n,
            total = N.g,
            method = "linear") # linear primera opcion  

checkcalibration(Xs = Xk, 
                 d = encuesta_ampliada$n,
                 total = N.g,
                 g = gk)

hist(gk)
summary(gk)
ggplot(data.frame(x = gk), aes(x = x)) +
  geom_histogram(binwidth = diff(range(gk)) / 20,
                 color = "black", alpha = 0.7) +
  labs(title = "", x = "", y = "") +
  theme_minimal() +
  theme(axis.text.y = element_blank(),
        axis.ticks.y = element_blank())

n1 <- encuesta_ampliada$n * gk

ggplot(data = data.frame(x = encuesta_ampliada$n, y = n1), aes(x = x, y = y)) +
  geom_point() +  # Agregar puntos
  geom_abline(intercept = 0, slope = 1, color = "red", linetype = "dashed") + 
  labs(title = "", x = "Conteo censales antes", y = "Conteos censales después") +
  theme_minimal(20)

encuesta_ampliada$n <- encuesta_ampliada$n * gk

# Se guarda el conjunto de datos ampliado:
saveRDS(encuesta_ampliada, "../output/2020/encuesta_ampliada.rds")
```

