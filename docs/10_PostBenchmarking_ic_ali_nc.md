# 10_PostBenchmarking_ic_ali_nc.R

Para la ejecución del presente archivo, debe abrir el archivo **10_PostBenchmarking_ic_ali_nc.R** disponible en la ruta *Rcodes/2020/10_PostBenchmarking_ic_ali_nc.R*.

Este script en R se centra en la validación y benchmarking de un modelo  multinivel para predecir la carencia en alimentos nutritivos y de calidad (ic_ali_nc) utilizando datos de encuestas y censos. Comienza limpiando el entorno de trabajo y cargando librerías necesarias. Además, se leen funciones auxiliares desde archivos externos para la validación y el benchmarking. Luego, se carga el modelo preentrenado desde un archivo RDS.

El proceso de benchmarking se realiza con la función `benchmarking`, utilizando covariables específicas como `ent`, `area`, `sexo`, `edad`, `discapacidad` y `hlengua`, y aplicando el método "logit". Los resultados se almacenan en `list_bench`. A continuación, se crea un diseño de encuesta y se realizan validaciones a nivel nacional comparando los resultados del modelo ajustado (`yk_stan_lmer`) y del modelo ajustado con benchmarking (`yk_bench`). Estas validaciones se llevan a cabo mediante la comparación de medias ponderadas.

Para validar el modelo a nivel de subgrupos, se definen diversos subgrupos como `ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua` y `nivel_edu`. Se crean gráficos de validación para cada subgrupo utilizando la función `plot_uni_validacion`. Estos gráficos se combinan y se guardan en un archivo JPG (`plot_uni_ic_ali_nc.jpg`). Además, se guarda la lista de gráficos y el dataframe postestratificado en archivos RDS (`plot_uni_ic_ali_nc.rds` y `poststrat_df_ic_ali_nc.rds` respectivamente), lo que permite una fácil recuperación y análisis posterior.



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

fit <- readRDS("../output/2020/modelos/fit_mrp_ic_ali_nc.rds")

###############################################################
# proceso de benchmarking
###############################################################

list_bench <- benchmarking(modelo = fit,
             names_cov =   c("ent",
                             "area",
                             "sexo",
                             "edad",
                             "discapacidad",
                             "hlengua"),                      
             metodo = "logit")


##############################################################
## Validaciones y plot uni 
###############################################################

poststrat_df <- fit$poststrat_df %>% data.frame() %>%  
  mutate(yk_stan_lmer = yk,
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
    Nacional_stan_lmer = sum(n * yk_stan_lmer) / sum(n),
    Nacional_bench = sum(n * yk_bench) / sum(n*gk_bench),
  )) %>% print()


###########################################
### Validaciones por subgrupo completo  ###
###########################################

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
       filename = "../output/2020/modelos/plot_uni_ic_ali_nc.jpg",
       scale = 3)

saveRDS(object =  plot_subgrupo,
        file = "../output/2020/modelos/plot_uni_ic_ali_nc.rds")

saveRDS(object =  poststrat_df,
        file = "../input/2020/muestra_ampliada/poststrat_df_ic_ali_nc.rds")
```

