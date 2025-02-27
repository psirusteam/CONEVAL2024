# 13_Estimacion_ent_02.R {-}
 
Para la ejecución del presente archivo, debe abrir el archivo **13_Estimacion_ent_02.R** disponible en la ruta *Rcodes/2020/13_Estimacion_ent_02.R*.



#### Inicialización y Carga de Librerías {-}

En esta primera sección, se limpia el entorno de trabajo eliminando todas las variables y objetos existentes mediante `rm(list = ls())`. A continuación, se cargan diversas librerías necesarias para el análisis de datos y modelado, como `patchwork` para la visualización, `lme4` para modelos lineales mixtos, `tidyverse` para manipulación de datos, y otras librerías específicas para análisis de encuestas y predicción. También se incluye un archivo externo de funciones llamado `modelos_freq.R`.


``` r
rm(list = ls())
library(patchwork)
library(nortest)
library(lme4)
library(tidyverse)
library(magrittr)
library(caret)
library(car)
library(survey)
library(srvyr)
source("../source/modelos_freq.R")
```


#### Configuración del Nivel de Agregación y Carga de Datos {-}

Aquí se define un vector `byAgrega` que especifica los niveles de agregación para el análisis, como entidad, municipio, área, y variables demográficas.


``` r
byAgrega <-
  c("ent",
    "cve_mun",
    "area",
    "sexo",
    "edad",
    "discapacidad",
    "hlengua",
    "nivel_edu" )

memory.limit(10000000)
```

Esta sección carga y filtra los datos necesarios desde archivos RDS y CSV. Se establece un límite de memoria y se cargan varios conjuntos de datos relevantes para el análisis, como la encuesta `enigh`, datos del censo, y predictores a nivel estatal. También se carga un archivo de líneas de bienestar y se actualizan los nombres de las variables para el análisis.


``` r
encuesta_enigh <- readRDS("../input/2020/enigh/encuesta_sta.rds") %>% 
  filter(ent == "02", ictpc <= 10000)
censo_sta <- readRDS("../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds") %>% 
  filter(ent == "02")
statelevel_predictors_df <- readRDS("../input/2020/predictores/statelevel_predictors_df.rds") %>% 
  filter(ent == "02")
muestra_ampliada <- readRDS("output/2020/encuesta_ampliada.rds") %>% 
  filter(ent == "02")
LB <-
  read.delim(
    "../input/2020/Lineas_Bienestar.csv",
    header = TRUE,
    sep = ";",
    dec = ","
  ) %>% mutate(area = as.character(area))
cov_names <- names(statelevel_predictors_df)
cov_names <- cov_names[!grepl(x = cov_names, pattern = "^hog_|cve_mun")]
```


#### Preparación de Datos y Modelado {-}

Aquí se preparan los datos combinando diferentes conjuntos mediante `inner_join` para integrar información de predictores estatales y líneas de bienestar con la muestra ampliada.


``` r
col_names_muestra <- names(muestra_ampliada)
muestra_ampliada <-  inner_join(muestra_ampliada, statelevel_predictors_df)
muestra_ampliada <- muestra_ampliada %>% inner_join(LB)
```

#### Modelado del Ingreso {-}

Se seleccionan y preparan las variables para modelar el ingreso. Se define la fórmula del modelo que incluye efectos aleatorios y fijos, y se ajusta un modelo usando la función `modelo_ingreso`. Los resultados del modelo se guardan en `fit_ingreso`.



``` r
variables_seleccionadas <-
  c(
    "prom_esc_rel_urb",
    "smg1",
    "gini15m",
    "ictpc15",
    "altitud1000",
    "prom_esc_rel_rur",
    "porc_patnoagrnocal_urb",
    "acc_medio",
    "derhab_pea_15_20",
    "porc_norep_ing_urb"
  )
cov_names <- c(
  "modifica_humana", "acceso_hosp",
  "acceso_hosp_caminando", "cubrimiento_cultivo",
  "cubrimiento_urbano", "luces_nocturnas",
  variables_seleccionadas
)
cov_registros <- setdiff(
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
    "smg1",
    "ql_porc_cpa_urb",
    "ql_porc_cpa_rur"
  )
)
cov_registros <- paste0(cov_registros, collapse = " + ")
formula_model <- 
  paste0("ingreso ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad)  + nivel_edu +  edad + area + sexo +", 
      " + ", cov_registros)
encuesta_enigh$ingreso <- (as.numeric(encuesta_enigh$ictpc))
fit <- modelo_ingreso(
  encuesta_sta = encuesta_enigh,
  predictors = statelevel_predictors_df,
  censo_sta = censo_sta,
  formula_mod = formula_model,
  byAgrega = byAgrega
)
fit_ingreso <- fit$fit_mrp
```


#### Modelado de Alimentos de Calidad y Seguridad Social {-}

Esta sección es similar a la anterior, pero se enfoca en modelar la calidad de alimentos y seguridad social. Se ajustan modelos para alimentos de calidad y seguridad social, con fórmulas adecuadas y utilizando la función `modelo_dummy`.



``` r
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
  "modifica_humana", "acceso_hosp",
  "acceso_hosp_caminando", "cubrimiento_cultivo",
  "cubrimiento_urbano", "luces_nocturnas",
  variables_seleccionadas
)

cov_registros <- setdiff(
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
formula_model <- 
  paste0(
    "cbind(si, no) ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad) + nivel_edu + edad + area + sexo ",
    " + ",
    cov_registros
  )
fit <- modelo_dummy(
  encuesta_sta = encuesta_enigh %>%  
    mutate(yk = ifelse(ic_ali_nc == 1 , 1, 0)),
  predictors = statelevel_predictors_df,
  censo_sta = censo_sta,
  formula_mod = formula_model,
  byAgrega = byAgrega
)
fit_alimento <- fit$fit_mrp
```



``` r
################################################################################
# modelo para seguridad social  #
################################################################################

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
  "luces_nocturnas" ,
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

formula_model <-
  paste0(
    "cbind(si, no) ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad) +  nivel_edu + edad   + area + sexo "
    ,
    " + ",
    cov_registros
  )




fit <- modelo_dummy(
  encuesta_sta = encuesta_enigh  %>% 
    mutate(yk = ifelse(ic_segsoc == 1 ,1,0))  ,
  predictors = statelevel_predictors_df,
  censo_sta = censo_sta ,
  formula_mod = formula_model,
  byAgrega = byAgrega
)

fit_segsoc <- fit$fit_mrp
```


#### Predicción y Validación {-}

En esta parte, se realiza la predicción del ingreso utilizando el modelo ajustado y se evalúa la precisión de las predicciones. Se calcula el error estándar residual y se ajustan los valores en función de las predicciones realizadas.



``` r
encuesta_sta <- inner_join(encuesta_enigh, statelevel_predictors_df)
pred_ingreso <- (predict(fit_ingreso, newdata = encuesta_sta))
sum(pred_ingreso < 3560)
rm(encuesta_sta)
paso <- encuesta_enigh %>% mutate(pred_ingreso = pred_ingreso) %>% 
  survey::svydesign(id =~ upm , strata = ~ estrato , data = ., weights = ~ fep) %>% 
  as_survey_design()
sd_1 <- paso %>% group_by(cve_mun) %>% 
  summarise(media_obs = survey_mean(ingreso), 
            media_pred = survey_mean(pred_ingreso)) %>% 
  summarise(media_sd = sqrt(mean(c(media_obs - media_pred)^2)))
desv_estandar_residual <- min(c(as.numeric(sd_1), sigma(fit_ingreso)))
```


#### Cálculo del Índice de Pobreza Multidimensional (IPM) {-}

Finalmente, se calcula el índice de pobreza multidimensional (IPM) basado en varias dimensiones, se ajusta la encuesta con estos datos, y se calcula la media del IPM.


``` r
encuesta_enigh <-
  encuesta_enigh %>% inner_join(LB)  %>% 
  mutate(
    tol_ic = ic_segsoc + ic_ali_nc + ic_asalud + ic_cv +  ic_sbv + ic_rezedu,
    ipm   = case_when(
      # Población en situación de pobreza.
      ingreso < lp  &  tol_ic >= 1 ~ "I",
      # Población vulnerable por carencias sociales.
      ingreso >= lp & tol_ic >= 1 ~ "II",
      # Poblacion vulnerable por ingresos.
      ingreso <= lp & tol_ic < 1 ~ "III",
      # Población no pobre multidimensional y no vulnerable.
      ingreso >= lp & tol_ic < 1 ~ "IV"
    ),
    # Población en situación de pobreza moderada.
    pobre_moderada = ifelse(c(ingreso > li & ingreso < lp) &
                              tol_ic > 2, 1, 0),
    pobre_extrema = ifelse(ipm == "I" & pobre_moderada == 0, 1, 0)
    
  )
```


#### Predicción con los Modelos Ajustados {-}

Se generan predicciones para las variables de interés (`ingreso`, `ic_ali_nc`, `ic_segsoc`) usando los modelos ajustados (`fit_ingreso`, `fit_alimento`, `fit_segsoc`) y el conjunto de datos `muestra_ampliada`. Se permite el uso de nuevos niveles en las predicciones.


``` r
pred_ingreso <- predict(fit_ingreso, 
                        newdata = muestra_ampliada ,
                        allow.new.levels = TRUE, 
                        type = "response")

pred_ic_ali_nc <- predict(fit_alimento, 
                          newdata = muestra_ampliada ,
                          allow.new.levels = TRUE, 
                          type = "response")

pred_segsoc <- predict(fit_segsoc, 
                       newdata = muestra_ampliada ,
                       allow.new.levels = TRUE, 
                       type = "response")
```

#### Creación de Variables Dummy y Preparación de Datos {-}

Aquí, se crean variables dummy para indicadores de salud, seguridad social, y otras características. Luego, se preparan datos adicionales para `muestra_ampliada_pred`, generando variables simuladas (`ic_segsoc`, `ic_ali_nc`) y ajustando el ingreso con ruido aleatorio. Se calcula el total del índice de carencias (`tol_ic`).


``` r
muestra_ampliada %<>% mutate(
  ic_asalud = ifelse(ic_asalud == 1, 1,0),
  ic_cv = ifelse(ic_cv == 1, 1,0),
  ic_sbv = ifelse(ic_sbv == 1, 1,0),
  ic_rezedu = ifelse(ic_rezedu == 1, 1,0)) 

muestra_ampliada_pred <-
  muestra_ampliada %>%
  select(all_of(col_names_muestra), li, lp) %>%
  mutate(
    ic_segsoc = rbinom(n = n(), size = 1, prob = pred_segsoc),
    ic_ali_nc = rbinom(n = n(), size = 1, prob = pred_ic_ali_nc),
    ingreso = pred_ingreso + rnorm(n = n(), mean = 0, desv_estandar_residual),
    tol_ic = ic_segsoc + ic_ali_nc + ic_asalud + ic_cv +
      ic_sbv + ic_rezedu)
```


#### Validación de las Predicciones {-}

Se validan las predicciones comparando las medias de las variables simuladas con las predicciones generadas por los modelos. Se calcula la diferencia entre estas medias para verificar la precisión de las predicciones.


``` r
mean(muestra_ampliada_pred$ic_ali_nc) -
  mean(pred_ic_ali_nc)

mean(muestra_ampliada_pred$ic_segsoc) -
  mean(pred_segsoc)

mean(muestra_ampliada_pred$ingreso) -
  mean(pred_ingreso)
```


#### Cálculo del Índice de Pobreza Multidimensional (IPM) con Predicciones {-}

Se clasifica la población en diferentes categorías de pobreza multidimensional (`ipm`) y se identifican las personas en pobreza moderada y extrema. Se calcula la proporción de cada categoría en `muestra_ampliada_pred`.


``` r
muestra_ampliada_pred <- muestra_ampliada_pred %>% mutate(
  ipm   = case_when(
    ingreso < lp  &  tol_ic >= 1 ~ "I",
    ingreso >= lp & tol_ic >= 1 ~ "II",
    ingreso <= lp & tol_ic < 1 ~ "III",
    ingreso >= lp & tol_ic < 1 ~ "IV"
  ),
  pobre_moderada = ifelse(c(ingreso > li & ingreso < lp) &
                            tol_ic > 2, 1, 0),
  pobre_extrema = ifelse(ipm == "I" & pobre_moderada == 0, 1, 0) )
prop.table(table(muestra_ampliada_pred$ipm))
```

#### Estimación y Comparación del IPM con Predicciones {-}

Se estima el IPM para las predicciones y se compara con la estimación real basada en `encuesta_enigh`. Se calculan las proporciones de cada categoría IPM para ambas estimaciones.



``` r
muestra_ampliada_pred %>% filter() %>% group_by(ipm) %>% 
  summarise(num_ipm =sum(factor)) %>% 
  mutate(est_ipm = num_ipm/sum(num_ipm))

encuesta_enigh %>% group_by(ipm) %>% 
  summarise(num_ipm =sum(fep)) %>% 
  mutate(est_ipm = num_ipm/sum(num_ipm))
```


#### Ajuste de las Predicciones y Calibración {-}

Se actualizan las variables de `muestra_ampliada_pred` con información adicional y se elimina cualquier dato faltante en `encuesta_enigh`.


``` r
muestra_ampliada_pred   %<>% mutate(
  tol_ic4 = ic_asalud + ic_cv +  ic_sbv + ic_rezedu,
  pred_segsoc = pred_segsoc,
  pred_ingreso = pred_ingreso,
  desv_estandar_residual = desv_estandar_residual,
  pred_ic_ali_nc = pred_ic_ali_nc
) 

ii_ent = "02"
encuesta_enigh %<>% na.omit()
```


#### Cálculo de la Población y Pobreza por Municipio {-}

Se calcula la densidad de población por municipio y se determina el porcentaje de población en cada categoría de IPM. Se estima la pobreza moderada y extrema en función de las encuestas y la población total.


``` r
total_mpio  <- muestra_ampliada_pred %>% group_by(cve_mun) %>% 
  summarise(den_mpio = sum(factor),.groups = "drop") %>% 
  mutate(tot_ent = sum(den_mpio))

tot_pob <- encuesta_enigh %>% 
  group_by(ipm) %>% 
  summarise(num_ent = sum(fep),.groups = "drop") %>% 
  transmute(ipm,   prop_ipm = num_ent/sum(num_ent),
            tx_ipm =  sum(total_mpio$den_mpio)*prop_ipm) %>% 
  filter(ipm != "I")

pobreza <-  encuesta_enigh %>% summarise(
  pobre_ext = weighted.mean(pobre_extrema, fep),
  pob_mod = weighted.mean(pobre_moderada, fep),
  pobre_moderada = sum(total_mpio$den_mpio)*pob_mod,
  pobre_extrema = sum(total_mpio$den_mpio)*pobre_ext)
```



#### Iteraciones de Calibración {-}

Este bloque realiza iteraciones para ajustar y calibrar los datos, actualizando las predicciones en cada iteración y comparando los resultados con la población real. Se calculan y guardan los resultados en archivos `.rds` para cada iteración.



``` r
for( iter in 1:200){
  cat("\n iteracion = ", iter,"\n\n")    
  
  muestra_ampliada_pred %<>% 
    mutate(
      ic_segsoc = rbinom(n = n(), size = 1, prob = pred_segsoc),
      ic_ali_nc = rbinom(n = n(), size = 1, prob = pred_ic_ali_nc),
      ingreso = pred_ingreso + rnorm(n = n(),mean = 0,
                                     desv_estandar_residual),
      tol_ic = tol_ic4 + ic_segsoc + ic_ali_nc
    ) %>% mutate(
      ipm   = case_when(
        ingreso < lp  &  tol_ic >= 1 ~ "I",
        ingreso >= lp & tol_ic >= 1 ~ "II",
        ingreso <= lp & tol_ic < 1 ~ "III",
        ingreso >= lp & tol_ic < 1 ~ "IV"
      ),
      pobre_moderada = ifelse(c(ingreso > li & ingreso < lp) &
                                tol_ic > 2, 1, 0),
      pobre_extrema = ifelse(ipm == "I" & pobre_moderada == 0, 1, 0)
    )  

  Xk <- muestra_ampliada_pred %>% select("ipm", "cve_mun") %>% 
    fastDummies::dummy_columns(select_columns = c("ipm", "cve_mun")) %>% 
    select(names(Tx_hat[-c(1:2)]))

  diseno_post <- bind_cols(muestra_ampliada_pred,Xk) %>% 
    mutate(fep = factor) %>%
    as_survey_design(
      ids = upm,
      weights = fep,
      nest = TRUE,
      # strata = estrato
    )

  mod_calib <-  as.formula(paste0("~ -1+",paste0(names(Tx_hat), collapse = " + ")))

  diseno_calib <- calibrate(diseno_post, formula = mod_calib, 
                            population = Tx_hat,calfun = "raking")

  estima_calib_ipm  <- diseno_calib %>% group_by(cve_mun,ipm) %>% 
    summarise(est_ipm = survey_mean(vartype ="var"))

  estima_calib_pob  <- diseno_calib %>% group_by(cve_mun) %>% 
    summarise(est_pob_ext = survey_mean(pobre_extrema , vartype ="var"),
              est_pob_mod = survey_mean(pobre_moderada  , vartype ="var" ))

  estima_calib

 <- pivot_wider(
    data = estima_calib_ipm,
    id_cols = "cve_mun",
    names_from = "ipm",
    values_from = c("est_ipm", "est_ipm_var"),values_fill = 0
  ) %>% full_join(estima_calib_pob)

  valida_ipm <- estima_calib_ipm %>% inner_join(total_mpio, by = "cve_mun") %>% 
    group_by(ipm) %>% 
    summarise(prop_ampliada = sum(est_ipm*den_mpio)/unique(tot_ent)) %>% 
    full_join(tot_pob, by = "ipm")

  valida_pob <- estima_calib_pob %>% select(cve_mun, est_pob_ext , est_pob_mod) %>% 
    inner_join(total_mpio, by = "cve_mun") %>%
    summarise(pob_ext_ampliada = sum(est_pob_ext * den_mpio) / unique(tot_ent),
              pob_mod_ampliada = sum(est_pob_mod * den_mpio) / unique(tot_ent),
              ipm_I = pob_ext_ampliada + pob_mod_ampliada) %>% 
    bind_cols(pobreza[,1:2])

  saveRDS(list(valida = list(valida_pob = valida_pob, 
                             valida_ipm = valida_ipm),
               estima_calib = estima_calib),
          file = paste0( "../output/2020/iteraciones/mpio_calib/",
                         ii_ent,"/iter",iter,".rds"))
}
fin <- Sys.time()
tiempo_total <- difftime(fin, inicio, units = "mins")
print(tiempo_total)
cat("####################################################################\n")
```

