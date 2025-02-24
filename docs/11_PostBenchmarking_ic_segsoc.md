# 11_PostBenchmarking_ic_segsoc.r {-}

Para la ejecución del presente análisis, se debe abrir el archivo **11_PostBenchmarking_ic_segsoc.R** disponible en la ruta *Rcodes/2020/11_PostBenchmarking_ic_segsoc.R*.

Este script en R se centra en el proceso de benchmarking y validación de un modelo multinivel previamente ajustado para lala carencia en por cobertura de seguridad social. Inicia con la limpieza del entorno de trabajo y la carga de librerías esenciales como `scales`, `patchwork`, `srvyr`, `survey`, `haven`, `sampling` y `tidyverse`. Luego, se cargan funciones adicionales desde archivos externos para la validación y el benchmarking del modelo.

El modelo ajustado se lee desde un archivo RDS (`fit_mrp_ic_segsoc.rds`) y se procede a la etapa de benchmarking utilizando la función benchmarking, aplicando el método logit sobre un conjunto de covariables específicas (`ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`). Los resultados del benchmarking se incorporan al dataframe de post-estratificación y se recalculan las predicciones ponderadas.

El script también realiza validaciones a nivel nacional y por subgrupos (`ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`, `nivel_edu`). Para cada subgrupo, se generan gráficos de validación univariada utilizando la función plot_uni_validacion, que compara las estimaciones directas y ajustadas del modelo. Los gráficos resultantes se combinan y se guardan en un archivo JPG, proporcionando una visualización completa del desempeño del modelo. Finalmente, tanto los gráficos de subgrupos como el dataframe de post-estratificación se guardan en archivos RDS para futuros análisis y referencia.
#### Limpieza del Entorno y Carga de Bibliotecas{-}

El entorno de R se limpia al eliminar todos los objetos actuales y se limpia la consola para asegurar que el análisis comience desde un estado limpio. A continuación, se cargan las bibliotecas necesarias para el análisis, que incluyen `scales`, `patchwork`, `srvyr`, `survey`, `haven`, `sampling`, y `tidyverse`. Estas bibliotecas proporcionan herramientas para la manipulación de datos, la creación de gráficos, el análisis de encuestas y la realización de muestreos.






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
```

#### Lectura de Funciones{-}

Se cargan dos archivos de funciones que serán utilizadas en el análisis: `Plot_validacion_bench.R` y `Benchmarking.R`. Estos archivos contienen funciones personalizadas para la validación y el análisis comparativo.


``` r
source("../source/Plot_validacion_bench.R", encoding = "UTF-8")
source("../source/Benchmarking.R", encoding = "UTF-8")
```

#### Lectura del Modelo {-}

Se lee el modelo ajustado previamente, guardado en un archivo RDS (`fit_mrp_ictpc.rds`). Este modelo será utilizado para realizar análisis adicionales o generar nuevas predicciones basadas en los datos de entrada.


``` r
fit <- readRDS("../output/2020/modelos/fit_mrp_ic_segsoc.rds")
```

#### proceso de benchmarking{-}

En esta sección se realiza el proceso de benchmarking utilizando la función `benchmarking`, aplicada al modelo previamente ajustado. La función evalúa el rendimiento del modelo y compara los resultados basados en las variables seleccionadas. Se especifican las variables que se consideran en el análisis: `ent`, `area`, `sexo`, `edad`, `discapacidad`, y `hlengua`. El método de benchmarking utilizado es `logit`, que se emplea para analizar el ajuste del modelo en función de estas variables.

El resultado del proceso de benchmarking se guarda en un archivo RDS (`list_bench.rds`), que contiene las evaluaciones del modelo. Aunque el código para guardar y cargar este archivo está comentado, se proporciona la estructura para guardar los resultados y posteriormente cargarlos si es necesario.


``` r
list_bench <- benchmarking(modelo = fit,
             names_cov =   c("ent",
                             "area",
                             "sexo",
                             "edad",
                             "discapacidad",
                             "hlengua"),                      
             metodo = "logit")
```

#### Validaciones benchmarking {-}

En esta sección se llevan a cabo validaciones del modelo y se comparan los resultados con los datos de la encuesta. Primero, se prepara un DataFrame `poststrat_df` que incluye las predicciones del modelo (`yk_lmer`) y un ajuste de benchmarking (`yk_bench`). Esto permite comparar las estimaciones del modelo con los datos observados.

Luego, se crea un objeto de diseño de encuesta `diseno_encuesta` utilizando la función `as_survey_design` para aplicar los pesos de encuesta a los datos. Se calculan las medias de las predicciones del modelo y de los datos directos para verificar la consistencia entre ambos.

Para la validación nacional, se comparan las estimaciones nacionales obtenidas de la encuesta y del modelo. Esto se realiza mediante la comparación de las medias estimadas a nivel nacional del diseño de encuesta (`Nacional_dir`), el modelo ajustado (`Nacional_lmer`), y el benchmarking (`Nacional_bench`). Los resultados se resumen en un `cbind` para facilitar la comparación.

Este proceso de validación permite evaluar la precisión del modelo y asegurar que sus estimaciones sean consistentes con los datos observados y con los ajustes de benchmarking.


``` r
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
```

#### Validaciones por subgrupo completo {-}

En esta sección, se realiza una validación exhaustiva del modelo ajustado, evaluando su desempeño a nivel de diversos subgrupos. Los subgrupos se definen en el vector `subgrupo`, que incluye variables como `ent`, `area`, `sexo`, `edad`, `discapacidad`, `hlengua`, y `nivel_edu`.

La función `plot_uni_validacion` se utiliza para generar gráficos y tablas de validación para cada uno de estos subgrupos. Estos gráficos y tablas permiten comparar las estimaciones obtenidas mediante el modelo (`stan_lmer`) con los datos directos (`directo`) para cada subgrupo, calculando también el error relativo (ER), que refleja la diferencia porcentual entre las dos estimaciones.

Para el subgrupo `sexo`, se genera una tabla que muestra el error relativo calculado para cada categoría dentro del subgrupo, ordenado por el tamaño de la muestra (`n_sample`). Los gráficos generados para cada subgrupo se combinan en un único gráfico utilizando `patchwork`. Este gráfico combinado permite una visualización comparativa clara entre todos los subgrupos definidos.

Finalmente, el gráfico combinado se guarda como una imagen JPG en el directorio de salida, y los objetos `plot_subgrupo` y `poststrat_df` se almacenan en archivos RDS. Estos archivos proporcionan una referencia detallada de la validación del modelo a nivel de subgrupo, permitiendo revisiones y análisis adicionales si es necesario.



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
       filename = "../output/2020/modelos/plot_uni_ic_segsoc.jpg",
       scale = 3)

saveRDS(object =  plot_subgrupo,
        file = "../output/2020/modelos/plot_uni_ic_segsoc.rds")

saveRDS(object =  poststrat_df,
        file = "../input/2020/muestra_ampliada/poststrat_df_ic_segsoc.rds")
```
