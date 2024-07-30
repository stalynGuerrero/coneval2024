# 09_PostBenchmarking_ictpc.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **09_PostBenchmarking_ictpc.R** disponible en la ruta *Rcodes/2020/09_PostBenchmarking_ictpc.R*.

Este script en R se centra en la validación y benchmarking de un modelo multinivel para predecir ingresos, utilizando datos de encuestas y censos. Comienza limpiando el entorno de trabajo (`rm(list = ls())`) y cargando las librerías necesarias. Además, se leen las funciones auxiliares desde archivos externos (`Plot_validacion_bench.R` y `Benchmarking.R`). Posteriormente, se carga el modelo preentrenado desde un archivo RDS (`fit_mrp_ictpc.rds`).

El proceso de benchmarking se realiza mediante la función `benchmarking`, utilizando covariables específicas como `ent`, `area`, `sexo`, `edad`, `discapacidad` y `hlengua`. El resultado se guarda en `list_bench`. A continuación, se crea un diseño de encuesta (`diseno_encuesta`) y se realizan validaciones a nivel nacional comparando los resultados del modelo ajustado (`yk_lmer`) y del modelo ajustado con benchmarking (`yk_bench`). Estas validaciones se llevan a cabo mediante la comparación de medias ponderadas.

Para validar el modelo a nivel de subgrupos, se definen diversos subgrupos como `ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua` y `nivel_edu`. Se crean gráficos de validación para cada subgrupo utilizando la función `plot_uni_validacion`. Estos gráficos se combinan y se guardan en un archivo JPG (`plot_uni.jpg`). Además, se guarda la lista de gráficos y el dataframe postestratificado en archivos RDS (`plot_uni.rds` y `poststrat_df_ictpc.rds` respectivamente), lo que permite una fácil recuperación y análisis posterior.




``` r
rm(list =ls())
cat("\f")
###############################################################
# Loading required libraries ----------------------------------------------
###############################################################

library(scales)
library(patchwork)
library(srvyr)
library(survey)
library(haven)
library(sampling)
library(tidyverse)

###############################################################
# Lectura de funciones 
###############################################################

source("../source/Plot_validacion_bench.R", encoding = "UTF-8")
source("../source/Benchmarking.R", encoding = "UTF-8")

###############################################################
# Lectura del modelo 
###############################################################

fit <- readRDS("../output/2020/modelos/fit_mrp_ictpc.rds")
```

#### proceso de benchmarking{-}


``` r
list_bench <- benchmarking(modelo = fit,
             names_cov =   c("ent",
                             "area",
                             "sexo",
                             "edad",
                             "discapacidad",
                             "hlengua"),                      
             metodo = "logit")

# saveRDS(list_bench,"output/2020/plots/ictpc/list_bench.rds")
# list_bench <- readRDS("output/2020/plots/ictpc/list_bench.rds")
```

#### Validaciones y plot uni {-}


``` r
poststrat_df <- fit$poststrat_df %>% data.frame() %>%  
  mutate(yk_lmer = yk,
         gk_bench = list_bench$gk_bench,
         yk_bench = yk * gk_bench )

diseno_encuesta <- fit$encuesta_mrp %>% 
  mutate(yk_dir = yk) %>% 
  as_survey_design(weights = fep)

mean(predict(fit$fit_mrp,type = "response"))
mean(diseno_encuesta$variables$yk)

## validación nacional.
cbind(
  diseno_encuesta %>% summarise(Nacional_dir = survey_mean(yk_dir)) %>% 
    select(Nacional_dir),
  poststrat_df %>% summarise(
    Nacional_lmer = sum(n * yk_lmer) / sum(n),
    Nacional_bench = sum(n * yk_bench) / sum(n*gk_bench),
  )) %>% print()
```

#### Validaciones por subgrupo completo{-}


``` r
subgrupo <- c("ent",
              "area",
              "sexo",
              "edad",
              "discapacidad",
              "hlengua","nivel_edu")


plot_subgrupo <- map(
  .x = setNames(subgrupo, subgrupo),
  ~ plot_uni_validacion(
    sample_diseno = diseno_encuesta,
    poststrat = poststrat_df,
    by1 = .x
  )
)


plot_subgrupo$sexo$tabla %>% arrange(desc(n_sample) ) %>%
  mutate(ER = abs((directo - stan_lmer)/directo )*100) %>%
  data.frame() %>% select(sexo:directo_upp,stan_lmer, ER)

plot_subgrupo$ent$gg_plot

plot_subgrupo$sexo$gg_plot + plot_subgrupo$nivel_edu$gg_plot +
  plot_subgrupo$edad$gg_plot + plot_subgrupo$hlengua$gg_plot +
  plot_subgrupo$area$gg_plot + plot_subgrupo$discapacidad$gg_plot  

plot_uni <- plot_subgrupo$ent$gg_plot  /
  (
    plot_subgrupo$sexo$gg_plot + plot_subgrupo$nivel_edu$gg_plot +
      plot_subgrupo$edad$gg_plot + plot_subgrupo$hlengua$gg_plot +
      plot_subgrupo$area$gg_plot + plot_subgrupo$discapacidad$gg_plot
  )

ggsave(plot = plot_uni,
       filename = "../output/2020/modelos/plot_uni.jpg",
       scale = 3)

saveRDS(object =  plot_subgrupo,
        file = "../output/2020/modelos/plot_uni.rds")

saveRDS(object =  poststrat_df,
        file = "../input/2020/muestra_ampliada/poststrat_df_ictpc.rds")
```
