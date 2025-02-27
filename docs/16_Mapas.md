# 16_Mapas.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **16_Mapas.R** disponible en la ruta *Rcodes/2020/16_Mapas.R*.

Este script en R está diseñado para procesar y visualizar datos geoespaciales relacionados con la pobreza multidimensional en México para el año 2020. Primero, se limpia el entorno de trabajo con `rm(list = ls())` para eliminar objetos previos y `gc()` para liberar memoria, lo que garantiza que el script se ejecute en un entorno limpio. Luego, se cargan diversas librerías necesarias para la manipulación de datos (`dplyr`, `data.table`), para trabajar con datos espaciales (`sf`), y para la creación de gráficos (`tmap`).

La memoria disponible se limita a 250 GB con `memory.limit(250000000)`, lo que es crucial para manejar grandes conjuntos de datos y evitar errores por falta de memoria. A continuación, se cargan dos conjuntos de datos importantes: `ipm_mpios`, que contiene estimaciones de pobreza para los municipios, y `ShapeDAM`, que es un archivo shapefile con la geometría de los municipios. En `ShapeDAM`, se renombra y ajusta la columna `cve_mun` para extraer códigos de entidad y se eliminan columnas innecesarias.

El script luego define los cortes de clasificación para las estimaciones de pobreza y genera mapas temáticos para diferentes tipos de pobreza multidimensional (`IPM_I`, `IPM_II`, `IPM_III`, `IPM_IV`) y para pobreza extrema y moderada. Utiliza la función `tm_shape()` de la librería `tmap` para crear los mapas y `tm_polygons()` para definir la apariencia de las áreas en el mapa, incluyendo los colores, la leyenda, y el título. Los mapas se guardan en archivos PNG con `tmap_save()`, especificando dimensiones y resolución para asegurar una alta calidad en la visualización.

Finalmente, el script también visualiza y guarda los errores de estimación asociados con cada tipo de pobreza, utilizando el mismo procedimiento para crear mapas temáticos con los errores de estimación. La visualización de estos errores ayuda a evaluar la precisión de las estimaciones y a identificar áreas con alta incertidumbre. Este proceso proporciona una manera efectiva de comunicar visualmente la distribución de la pobreza y los errores asociados a través de mapas detallados y estéticamente ajustados.

#### Configuración y carga de bibliotecas {-}

El siguiente código configura el entorno de R, carga las bibliotecas necesarias y establece un límite de memoria para el análisis:


``` r
### Cleaning R environment ###
rm(list = ls())
gc()

#################
### Libraries ###
#################
library(dplyr)
library(data.table)
library(haven)
library(magrittr)
library(stringr)
library(openxlsx)
library(tmap)
library(sf)
select <- dplyr::select

###------------ Definiendo el límite de la memoria RAM a emplear ------------###
memory.limit(250000000)
```


Primero, se limpia el entorno de trabajo de R eliminando todos los objetos y se ejecuta la recolección de basura para liberar memoria. Luego, se cargan varias bibliotecas esenciales: `dplyr` y `data.table` para manipulación y análisis de datos; `haven` para importar datos de formatos específicos; `magrittr` para operaciones de encadenamiento; `stringr` para manipulación de cadenas de texto; `openxlsx` para trabajar con archivos Excel; y `tmap` y `sf` para análisis y visualización espacial. Finalmente, se establece un límite de memoria RAM de 250 MB para el uso de R, asegurando que el análisis se realice dentro de los recursos disponibles del sistema.

#### Carga de datos y creación de mapas de IPM{-}

El siguiente código se encarga de cargar los datos de estimaciones de pobreza multidimensional y el archivo de formas geográficas, para luego preparar los datos para la visualización en mapas.



``` r
ipm_mpios <- 
  readRDS("../output/Entregas/2020/result_mpios.RDS") %>% 
  mutate(est_pob_ext = ifelse(est_pob_ext < 0, 0, est_pob_ext))

ShapeDAM <- read_sf("../shapefile/2020/MEX_2020.shp")
ShapeDAM %<>% mutate(cve_mun = CVEGEO,
                     ent = substr(cve_mun, 1, 2),
                     CVEGEO = NULL)
```

Primero, se cargan los resultados de estimaciones de pobreza multidimensional desde un archivo RDS (`result_mpios.RDS`). Se ajusta la variable `est_pob_ext` para asegurarse de que no tenga valores negativos, estableciendo dichos valores en cero.

Luego, se lee el archivo de forma geográfica (`MEX_2020.shp`) que contiene la geometría de los municipios de México. Se renombra la columna `CVEGEO` a `cve_mun` para que coincida con la variable en el conjunto de datos de estimaciones. También se extrae el código de entidad federal (`ent`) de los primeros dos caracteres de `cve_mun`, y se elimina la columna original `CVEGEO` que ya no es necesaria. 

Estos pasos preparan los datos para ser usados en la creación de mapas que visualicen las estimaciones de pobreza multidimensional a nivel municipal.


##### Definición de Cortes{-}

Esta sección establece los cortes utilizados para clasificar las estimaciones de pobreza multidimensional en diferentes categorías. Los cortes están definidos como fracciones de 100 (por ejemplo, 0, 15, 30, 50, 80, y 100) para facilitar la visualización en los mapas temáticos. Estos cortes se utilizan para determinar los intervalos de los valores de estimación en los mapas.


``` r
cortes <- c(0, 15, 30, 50, 80, 100) / 100
```

##### Creación del Mapa para IPM Tipo I{-}

En esta sección, se genera un mapa temático para la estimación del Índice de Pobreza Multidimensional (IPM) Tipo I. Primero, se combina la información geoespacial con las estimaciones de IPM por municipio. Luego, se crea un mapa usando la función `tm_polygons`, donde los colores representan los diferentes intervalos de pobreza Tipo I. El mapa incluye una leyenda detallada con el diseño y formato especificados, y se guarda como una imagen PNG.


``` r
P1_mpio_norte <-
  tm_shape(ShapeDAM %>%
             inner_join(ipm_mpios, by = "cve_mun"))

Mapa_I <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "est_ipm_I",
    title = "Estimación de la pobreza \nmultidimensional 2020 - Tipo I",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_I,
  filename = "../output/Entregas/2020/mapas/IPM_I.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Creación del Mapa para IPM Tipo II{-}

En esta sección, se crea un mapa temático para la estimación del IPM Tipo II. Similar al mapa del IPM Tipo I, se utiliza la función `tm_polygons` para representar los intervalos de pobreza Tipo II con colores específicos. El mapa incluye una leyenda con el formato y diseño configurados para una mejor interpretación. Finalmente, el mapa se guarda como una imagen PNG en la ubicación especificada.


``` r
Mapa_II <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "est_ipm_II",
    title = "Estimación de la pobreza \nmultidimensional 2020 - Tipo II",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_II,
  filename = "../output/Entregas/2020/mapas/IPM_II.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Creación del Mapa para IPM Tipo III{-}

Esta sección está dedicada a la creación del mapa temático para el IPM Tipo III. Se sigue el mismo procedimiento que para los mapas de IPM Tipo I y Tipo II, adaptando el título y la variable correspondiente a la estimación del IPM Tipo III. La visualización se guarda como una imagen PNG, proporcionando una representación clara y detallada de la distribución de pobreza Tipo III.


``` r
Mapa_III <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "est_ipm_III",
    title = "Estimación de la pobreza \nmultidimensional 2020 - Tipo III",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_III,
  filename = "../output/Entregas/2020/mapas/IPM_III.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Creación del Mapa para IPM Tipo IV{-}

Finalmente, esta sección se encarga de generar el mapa temático para el IPM Tipo IV. El proceso es similar al de los mapas anteriores, pero se ajusta para representar la estimación del IPM Tipo IV. El mapa incluye una leyenda informativa y se guarda como un archivo PNG para su distribución y análisis.


``` r
Mapa_IV <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "est_ipm_IV",
    title = "Estimación de la pobreza \nmultidimensional 2020 - Tipo IV",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_IV,
  filename = "../output/Entregas/2020/mapas/IPM_IV.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

#### Creación pobreza extrema y moderada, y errores de estimació{-}

Esta sección establece los cortes utilizados para clasificar las estimaciones de pobreza en categorías de pobreza extrema y moderada. Los cortes están definidos como fracciones de 100 (por ejemplo, 0, 15, 30, 50, 80, y 100) para facilitar la visualización en los mapas temáticos. Estos cortes ayudan a segmentar los datos en intervalos significativos para su representación gráfica.


``` r
cortes <- c(0, 15, 30, 50, 80, 100) / 100
```
Aquí se muestra un resumen estadístico de la variable `est_pob_ext`, que representa la estimación de la pobreza extrema en los municipios. El resumen proporciona una visión general de las estadísticas descriptivas de esta variable, como mínimo, máximo, media, y cuartiles.


``` r
summary(ipm_mpios$est_pob_ext)
```

##### Creación del Mapa de Pobreza Extrema{-}

En esta sección, se genera un mapa temático para la estimación de pobreza extrema. Se utiliza la función `tm_polygons` para representar los intervalos de pobreza extrema en colores distintos, con una leyenda detallada que facilita la interpretación. El mapa incluye configuraciones específicas para la leyenda y se guarda como una imagen PNG.


``` r
Mapa_ext <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "est_pob_ext",
    title = "Estimación de la pobreza extrema",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_ext,
  filename = "../output/Entregas/2020/mapas/pob_ext.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Resumen de Pobreza Moderada{-}

Esta sección muestra un resumen estadístico de la variable `est_pob_mod`, que representa la estimación de la pobreza moderada en los municipios. Similar al resumen de pobreza extrema, se proporciona una visión general de las estadísticas descriptivas de esta variable.


``` r
summary(ipm_mpios$est_pob_mod)
```

##### Creación del Mapa de Pobreza Moderada{-}

Aquí se genera un mapa temático para la estimación de pobreza moderada. Se utiliza la función `tm_polygons` para representar los intervalos de pobreza moderada con diferentes colores, junto con una leyenda informativa. Este mapa proporciona una visualización clara de la pobreza moderada en los municipios y se guarda como una imagen PNG.


``` r
Mapa_mod <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "est_pob_mod",
    title = "Estimación de la pobreza moderada",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_mod,
  filename = "../output/Entregas/2020/mapas/pob_mod.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

#### Definición de Cortes para Errores de Estimación{-}

En esta sección se establecen los cortes utilizados para clasificar los errores de estimación en categorías significativas. Los cortes están definidos como fracciones de 100 (0, 1, 5, 10, 25, 50, y 100) para permitir una visualización detallada de la variabilidad en los errores de estimación de pobreza multidimensional y sus tipos asociados.


``` r
cortes <- c(0, 1, 5, 10, 25, 50, 100) / 100
```

##### Mapa de Error de Estimación del IPM Tipo I

Se genera un mapa temático que visualiza el error de estimación del Índice de Pobreza Multidimensional (IPM) Tipo I. Utilizando la función `tm_polygons`, los errores se representan en diferentes colores según los intervalos definidos por los cortes. El mapa incluye una leyenda detallada y se guarda como una imagen PNG para su posterior análisis y presentación.


``` r
Mapa_I_ee <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "ee_ipm_I",
    title = "Error de estimación de la pobreza \nmultidimensional 2020 - Tipo I",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_I_ee,
  filename = "../output/Entregas/2020/mapas/IPM_I_ee.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Mapa de Error de Estimación del IPM Tipo II{-}

Aquí se crea un mapa temático para visualizar el error de estimación del IPM Tipo II. Los errores se clasifican en diferentes categorías de color según los cortes definidos. La leyenda y la disposición gráfica están configuradas para facilitar la interpretación de los resultados, y el mapa se guarda como una imagen PNG.


``` r
Mapa_II_ee <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "ee_ipm_II",
    title = "Error de estimación de la pobreza \nmultidimensional 2020 - Tipo II",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_II_ee,
  filename = "../output/Entregas/2020/mapas/IPM_II_ee.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Mapa de Error de Estimación del IPM Tipo III{-}

Este mapa temático ilustra el error de estimación del IPM Tipo III, utilizando los cortes para segmentar los errores en diferentes niveles de color. La leyenda está diseñada para mostrar claramente las categorías de errores y el mapa se guarda como un archivo PNG.


``` r
Mapa_III_ee <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "ee_ipm_III",
    title = "Error de estimación de la pobreza \nmultidimensional 2020 - Tipo III",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_III_ee,
  filename = "../output/Entregas/2020/mapas/IPM_III_ee.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### 5. Mapa de Error de Estimación del IPM Tipo IV{-}

Se genera un mapa temático que representa el error de estimación del IPM Tipo IV. Los errores se visualizan mediante diferentes colores según los intervalos definidos por los cortes. La leyenda y el diseño del mapa están ajustados para facilitar la interpretación visual de los errores, y el mapa se guarda como una imagen PNG.


``` r
Mapa_IV_ee <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "ee_ipm_IV",
    title = "Error de estimación de la pobreza \nmultidimensional 2020 - Tipo IV",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_IV_ee,
  filename = "../output/Entregas/2020/mapas/IPM_IV_ee.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Mapa de Error de Estimación de la Pobreza Extrema{-}

Este mapa temático ilustra el error de estimación para la pobreza extrema. Los datos se representan en diferentes colores según los cortes definidos, con una leyenda que facilita la interpretación. El mapa se guarda como un archivo PNG para su uso en informes y análisis.


``` r
Mapa_ext_ee <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "ee_pob_ext",
    title = "Error de estimación de la pobreza extrema",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_ext_ee,
  filename = "../output/Entregas/2020/mapas/pob_ext_ee.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```

##### Mapa de Error de Estimación de la Pobreza Moderada{-}

Finalmente, se genera un mapa temático que visualiza el error de estimación para la pobreza moderada. Los intervalos de error están representados por diferentes colores, y la leyenda está configurada para mostrar claramente las categorías de error. El mapa se guarda como un archivo PNG para su análisis y presentación.


``` r
Mapa_mod_ee <-
  P1_mpio_norte + tm_polygons(
    breaks = cortes,
    "ee_pob_mod",
    title = "Error de estimación de la pobreza moderada",
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(
    legend.show = TRUE,
    legend.text.size = 1.5,
    legend.outside.position = 'left',
    legend.hist.width = 1,
    legend.hist.height = 3,
    legend.stack = 'vertical',
    legend.title.fontface = 'bold',
    legend.text.fontface = 'bold'
  )

tmap_save(
  Mapa_mod_ee,
  filename = "../output/Entregas/2020/mapas/pob_mod_ee.png",
  width = 4000,
  height = 3000,
  asp = 0
)
```
