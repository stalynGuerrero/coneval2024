# 06_Modelo_unidad_ictpc.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **06_Modelo_unidad_ictpc.R** disponible en la ruta *Rcodes/2020/06_Modelo_unidad_ictpc.R*.

Este script en R se enfoca en la creación de un modelo multinivel utilizando datos de encuestas y censos para predecir ingresos. El proceso comienza con la limpieza del entorno de trabajo mediante `rm(list = ls())` y la carga de librerías esenciales para el análisis, tales como `patchwork`, `nortest`, `lme4`, `tidyverse`, `magrittr`, `caret`, `car`, y una fuente externa de modelos (`source("source/modelos_freq.R")`). Posteriormente, se definen las variables de agregación (`byAgrega`), se incrementa el límite de memoria disponible, y se cargan las bases de datos necesarias (`encuesta_sta`, `censo_sta`, `statelevel_predictors_df`). Se seleccionan las variables covariantes excluyendo aquellas que comienzan con `hog_` o `cve_mun`.

En la selección de variables relevantes, se mencionan algunos pasos comentados que realizan la selección de características utilizando el método `rfe` de `caret`. Las variables seleccionadas se listan explícitamente en `variables_seleccionadas` y se combinan con otras covariantes para formar `cov_names`. Luego, se construye la fórmula del modelo (`formula_model`), que incluye efectos aleatorios para `cve_mun`, `hlengua` y `discapacidad`, así como efectos fijos para las demás variables. Esto asegura que el modelo considere tanto la variabilidad dentro de cada grupo como las características específicas de cada variable.

Finalmente, se realiza una transformación la variable de ingreso (`ingreso`) a formato numérico y se ajusta el modelo utilizando la función `modelo_ingreso`, que está definida en el archivo fuente cargado previamente. Los datos de entrada incluyen `encuesta_sta`, `statelevel_predictors_df`, `censo_sta` y la fórmula del modelo. Los resultados del modelo se guardan en un archivo RDS (`fit_mrp_ictpc.rds`). Se generan y guardan gráficos de densidad y histogramas de las distribuciones posteriores utilizando `ggsave`, proporcionando una visualización clara de los resultados obtenidos.


#### Limpieza del Entorno y Carga de Bibliotecas {-}

Para preparar el entorno de análisis, se inicia con la limpieza del espacio de trabajo en R, eliminando todos los objetos existentes y ejecutando el recolector de basura para liberar memoria. Esta acción asegura que no haya conflictos o residuos de análisis anteriores que puedan afectar el trabajo actual. A continuación, se cargan las bibliotecas necesarias para el análisis. Entre estas se encuentran `patchwork` para la combinación de gráficos, `nortest` para pruebas de normalidad, `lme4` para modelos lineales mixtos, `tidyverse` para manipulación y visualización de datos, `magrittr` para la sintaxis de tuberías, `caret` para la creación de modelos predictivos, y `car` para técnicas de regresión y diagnóstico. Además, se incluye un archivo de funciones adicionales `modelos_freq.R`, que contiene scripts personalizados para el análisis de modelos.

Este entorno preparado permite realizar un análisis riguroso y detallado de los datos, asegurando que todas las herramientas y funciones necesarias estén disponibles y actualizadas.


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

Se define un vector de variables llamado `byAgrega`, que incluye los campos `ent`, `cve_mun`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`, y `nivel_edu`. Estas variables se utilizarán para la agregación de datos en el análisis, permitiendo estructurar y resumir la información según diferentes dimensiones y características.


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

Se cargan los datos necesarios para el análisis, incluyendo la encuesta (`encuesta_sta`), el censo (`censo_sta`), y los predictores a nivel estatal (`statelevel_predictors_df`). 

La memoria se ajusta a un límite de 10 millones para manejar estos datos. Tras la carga de los archivos RDS, se extraen los nombres de las covariables de `statelevel_predictors_df`, excluyendo aquellos que comienzan con "hog_" y el identificador de municipio (`cve_mun`). Esto permite enfocarse en las variables relevantes para el análisis.


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

Se procede a la selección de las variables predictoras que se utilizarán en el modelo. Se inicia por definir un conjunto inicial de variables y luego se elige un subconjunto relevante para el análisis. En esta fase, se han identificado y seleccionado variables predictoras clave, entre las que se encuentran indicadores relacionados con educación, ingreso, y características geográficas y económicas.

Primero, se propone una opción para seleccionar las variables usando una técnica de selección de características denominada `rfe` (recursive feature elimination), que evalúa el impacto de diferentes variables en la variable objetivo (`ingreso`). Sin embargo, esta opción está comentada en el código.

Luego, se definen explícitamente las variables seleccionadas para el análisis. Estas incluyen indicadores como `prom_esc_rel_urb`, `smg1`, `gini15m`, `ictpc15`, `altitud1000`, entre otros. Estas variables se consideran relevantes para el modelo y son seleccionadas para su uso en el análisis.

Finalmente, se filtran las variables que se registrarán para la inclusión en el análisis, excluyendo ciertas variables específicas que no serán consideradas. Este paso asegura que solo las covariables pertinentes sean utilizadas en la modelización, basándose en las necesidades específicas del análisis y los objetivos del estudio.


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

Se define la fórmula del modelo para el análisis, la cual especifica cómo se relacionan las variables predictoras con la variable dependiente `ingreso`. Esta fórmula incluye tanto efectos aleatorios como efectos fijos para capturar diferentes niveles de variabilidad en los datos.

En la fórmula del modelo, se incorporan efectos aleatorios para los códigos municipales (`cve_mun`), el habla de lenguas indígenas (`hlengua`), y la discapacidad (`discapacidad`). Estos efectos aleatorios permiten capturar la variabilidad específica entre estos grupos y ajustar el modelo en consecuencia.

Además, se incluyen varios efectos fijos que representan variables independientes clave en el análisis: entidad (`ent`), nivel educativo (`nivel_edu`), edad (`edad`), área (`area`), y sexo (`sexo`). Estas variables fijas proporcionan información sobre cómo estos factores afectan la variable dependiente.

Finalmente, se incorporan las variables predictoras seleccionadas, que se concatenan en la fórmula como términos adicionales. Estas variables predictoras se han seleccionado previamente como relevantes para el modelo y se añaden para evaluar su impacto en el ingreso. La fórmula resultante combina estos elementos para construir un modelo completo que considera tanto efectos aleatorios como fijos en el análisis de los datos.


``` r
cov_registros <- paste0(cov_registros, collapse = " + ")

formula_model <- 
  paste0("ingreso ~ (1 | cve_mun) + (1 | hlengua) + (1 | discapacidad) + ent + nivel_edu +  edad + area + sexo +", 
      " + ", cov_registros)
```

#### Ajuste del Modelo {-}

Se ajusta el modelo de ingreso utilizando la función `modelo_ingreso`, la cual se aplica a los datos de la encuesta (`encuesta_sta`) junto con los predictores a nivel estatal (`statelevel_predictors_df`) y la tabla censal (`censo_sta`). La fórmula del modelo, previamente definida (`formula_model`), se emplea para especificar las relaciones entre las variables y los efectos aleatorios y fijos en el análisis.

Una vez ajustado el modelo, se guardan los resultados en un archivo `.rds` en la ruta especificada, lo que permite almacenar y compartir el modelo ajustado para su posterior análisis o uso. Además, se generan y guardan visualizaciones de los resultados del modelo, incluyendo un gráfico de densidad y un histograma de la posteriori, que se almacenan como archivos de imagen (`.png`). Estos gráficos permiten examinar la distribución de los resultados y la calidad del ajuste del modelo, proporcionando una representación visual de los resultados obtenidos.


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
