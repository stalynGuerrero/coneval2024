# 11_PostBenchmarking_ic_segsoc.r

Para la ejecución del presente análisis, se debe abrir el archivo **11_PostBenchmarking_ic_segsoc.R** disponible en la ruta *Rcodes/2020/11_PostBenchmarking_ic_segsoc.R*.

Este script en R se centra en el proceso de benchmarking y validación de un modelo multinivel previamente ajustado para lala carencia en por cobertura de seguridad social. Inicia con la limpieza del entorno de trabajo y la carga de librerías esenciales como `scales`, `patchwork`, `srvyr`, `survey`, `haven`, `sampling` y `tidyverse`. Luego, se cargan funciones adicionales desde archivos externos para la validación y el benchmarking del modelo.

El modelo ajustado se lee desde un archivo RDS (`fit_mrp_ic_segsoc.rds`) y se procede a la etapa de benchmarking utilizando la función benchmarking, aplicando el método logit sobre un conjunto de covariables específicas (`ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`). Los resultados del benchmarking se incorporan al dataframe de post-estratificación y se recalculan las predicciones ponderadas.

El script también realiza validaciones a nivel nacional y por subgrupos (`ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`, `nivel_edu`). Para cada subgrupo, se generan gráficos de validación univariada utilizando la función plot_uni_validacion, que compara las estimaciones directas y ajustadas del modelo. Los gráficos resultantes se combinan y se guardan en un archivo JPG, proporcionando una visualización completa del desempeño del modelo. Finalmente, tanto los gráficos de subgrupos como el dataframe de post-estratificación se guardan en archivos RDS para futuros análisis y referencia.


``` r
rm(list =ls())
cat("\f")
###############################################################
# Loading required libraries ----------------------------------
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

fit <- readRDS("../output/2020/modelos/fit_mrp_ic_segsoc.rds")

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
       filename = "../output/2020/modelos/plot_uni_ic_segsoc.jpg",
       scale = 3)

saveRDS(object =  plot_subgrupo,
        file = "../output/2020/modelos/plot_uni_ic_segsoc.rds")

saveRDS(object =  poststrat_df,
        file = "../input/2020/muestra_ampliada/poststrat_df_ic_segsoc.rds")
```
