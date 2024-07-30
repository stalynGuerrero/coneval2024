# 06_Modelo_unidad_ictpc.R

Para la ejecución del presente archivo, debe abrir el archivo **06_Modelo_unidad_ictpc.R** disponible en la ruta *Rcodes/2020/06_Modelo_unidad_ictpc.R*.

Este script en R se enfoca en la creación de un modelo multinivel utilizando datos de encuestas y censos para predecir ingresos. El proceso comienza con la limpieza del entorno de trabajo mediante `rm(list = ls())` y la carga de librerías esenciales para el análisis, tales como `patchwork`, `nortest`, `lme4`, `tidyverse`, `magrittr`, `caret`, `car`, y una fuente externa de modelos (`source("source/modelos_freq.R")`). Posteriormente, se definen las variables de agregación (`byAgrega`), se incrementa el límite de memoria disponible, y se cargan las bases de datos necesarias (`encuesta_sta`, `censo_sta`, `statelevel_predictors_df`). Se seleccionan las variables covariantes excluyendo aquellas que comienzan con `hog_` o `cve_mun`.

En la selección de variables relevantes, se mencionan algunos pasos comentados que realizan la selección de características utilizando el método `rfe` de `caret`. Las variables seleccionadas se listan explícitamente en `variables_seleccionadas` y se combinan con otras covariantes para formar `cov_names`. Luego, se construye la fórmula del modelo (`formula_model`), que incluye efectos aleatorios para `cve_mun`, `hlengua` y `discapacidad`, así como efectos fijos para las demás variables. Esto asegura que el modelo considere tanto la variabilidad dentro de cada grupo como las características específicas de cada variable.

Finalmente, se realiza una transformación la variable de ingreso (`ingreso`) a formato numérico y se ajusta el modelo utilizando la función `modelo_ingreso`, que está definida en el archivo fuente cargado previamente. Los datos de entrada incluyen `encuesta_sta`, `statelevel_predictors_df`, `censo_sta` y la fórmula del modelo. Los resultados del modelo se guardan en un archivo RDS (`fit_mrp_ictpc.rds`). Se generan y guardan gráficos de densidad y histogramas de las distribuciones posteriores utilizando `ggsave`, proporcionando una visualización clara de los resultados obtenidos.


#### Limpieza del Entorno y Carga de Bibliotecas {-}

Se limpia el entorno de R y se cargan las bibliotecas necesarias para el análisis.


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
source("../source/modelos_freq.R")
```

#### Variables de Agregación {-}

Se define un vector de variables que se utilizarán para la agregación de datos.


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
```

#### Carga de Datos {-}

Se cargan los datos de la encuesta, el censo y los predictores a nivel estatal.


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

Se seleccionan las variables predictoras que se utilizarán en el modelo.


``` r
# Selección de variables

# encuesta_sta2 <- encuesta_sta %>% group_by_at((byAgrega)) %>%
#   summarise(ingreso = mean(ictpc), .groups = "drop",
#             n = n()) %>%
#   inner_join(statelevel_predictors_df)
# 
# 
# resultado_rfe <-
#   rfe(
#     encuesta_sta2[, cov_names],
#     encuesta_sta2$ingreso,
#     sizes = c(1:10),
#     rfeControl = rfeControl(functions = lmFuncs, method = "LOOCV")
#   )

variables_seleccionadas <-
  c(
    "prom_esc_rel_urb",
    "smg1",
    "gini15m" ,
    "ictpc15" ,
    "altitud1000",
    "prom_esc_rel_rur" ,
    "porc_patnoagrnocal_urb",
    "acc_medio"  ,
    "derhab_pea_15_20" ,
    "porc_norep_ing_urb"
  )
cov_names <- c(
  "modifica_humana",  "acceso_hosp",           
  "acceso_hosp_caminando",  "cubrimiento_cultivo",   
  "cubrimiento_urbano",     "luces_nocturnas" ,
  variables_seleccionadas
)

cov_registros <- setdiff(
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
    "smg1",
    "ql_porc_cpa_urb",
    "ql_porc_cpa_rur"
  )
)
```

#### Fórmula del Modelo {-}

Se define la fórmula del modelo incluyendo los efectos aleatorios y las variables predictoras seleccionadas.


``` r
cov_registros <- paste0(cov_registros, collapse = " + ")

formula_model <- 
  paste0("ingreso ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad) + ent + nivel_edu +  edad + area + sexo +", 
      " + ", cov_registros)
```

#### Ajuste del Modelo {-}

Se ajusta el modelo de ingreso utilizando la función `modelo_ingreso` y se guarda el resultado.


``` r
encuesta_sta$ingreso <- as.numeric(encuesta_sta$ictpc)

fit <- modelo_ingreso(
  encuesta_sta = encuesta_sta ,
  predictors = statelevel_predictors_df,
  censo_sta = censo_sta,
  formula_mod = formula_model,
  byAgrega = byAgrega
)

#--- Exporting Bayesian Multilevel Model Results ---#

saveRDS(fit, 
        file = "../output/2020/modelos/fit_mrp_ictpc.rds")

ggsave(plot = fit$plot_densy,
       "../output/2020/plots/01_densidad_ictpc.png",scale = 3)
fit$plot_hist_post
```
