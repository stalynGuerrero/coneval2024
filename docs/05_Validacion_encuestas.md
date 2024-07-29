# 05_Validacion_encuestas.R

Para ejecutar este archivo, es necesario abrir el archivo **05_Validacion_encuestas.R** ubicado en la ruta *Rcodes/2020/05_Validacion_encuestas.R*. Este script está diseñado para llevar a cabo un análisis integral de datos con varias fases que aseguran una evaluación rigurosa de la información disponible.

En la primera etapa del código, se eliminan todos los objetos del entorno de R para garantizar que el análisis se realice en un entorno limpio y sin interferencias. Luego, se cargan las librerías necesarias para llevar a cabo el análisis y se establece el tema de visualización predeterminado usando `bayesplot::theme_default()`. También se carga un archivo de script adicional que incluye funciones específicas para la creación de gráficos. Posteriormente, se leen las bases de datos `encuesta_sta` y `muestra_ampliada`, que contienen los datos de la encuesta y del censo, respectivamente.

El análisis procede realizando comparaciones entre los datos censales y los datos de la encuesta en diferentes variables categóricas, tales como edad, sexo, nivel educativo y estado. Para cada una de estas variables, se generan gráficos que permiten visualizar las diferencias. Utilizando la librería `patchwork`, se combinan estos gráficos en un solo diseño para facilitar la comparación. Además, se llevan a cabo análisis adicionales en variables como lengua indígena y discapacidad, y se exploran las interacciones entre variables como sexo y edad, nivel educativo y sexo, y estado y sexo. Finalmente, los gráficos que ilustran estas interacciones se integran en un diseño consolidado para una visualización comprensiva de los efectos.


``` r
### Cleaning R environment ###
rm(list = ls())

#################
### Libraries ###
#################
library(tidyverse)
library(reshape2)
library(stringr)
library(ggalt)
library(gridExtra)
library(scales)
library(formatR)
library(patchwork)

theme_set(bayesplot::theme_default())

source(file = "../source/Plot_validacion.R", encoding = "UTF-8")

################################################################################
# Lectura de las bases 
################################################################################
encuesta_sta <- readRDS("../input/2020/enigh/encuesta_sta.rds")
muestra_ampliada <- readRDS("../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")

### AGE ###
age_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "edad")
### Sex ###
sex_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "sexo")
### Level of schooling (LoS) ###
escolar_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "nivel_edu")

### States ###
depto_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "ent")

#--- Patchwork en acción ---#
(age_plot | sex_plot | escolar_plot) / (depto_plot)

hlengua <- Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "hlengua")

plot_discapacidad <- Plot_Compare(dat_censo = muestra_ampliada,
             dat_encuesta = encuesta_sta,
             by = "discapacidad")

(plot_discapacidad | hlengua )

# Interaction effects  ----------------------------------------------------

### AGE x SEX ###
encuesta_sta$pobreza <- encuesta_sta$ictpc
#--- Percentage of people in poverty by AGE x SEX ---#
p_sex_age <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "edad")

### Level of schooling (LoS) x SEX ###
p_sex_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "nivel_edu")

### State x SEX ###
p_sex_depto <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "ent")

#--- Patchwork in action ---#
(p_sex_age + p_sex_escolar) / p_sex_depto

### Level of schooling (LoS) x AGE ###
p_escolar_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "edad") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

### State x AGE ###
p_depto_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "edad",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "Edad")

p_escolar_edad / p_depto_edad

### Level of schooling (LoS) x State ###
p_depto_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

p_depto_escolar


### AGE x SEX ###
encuesta_sta$pobreza <- encuesta_sta$ic_segsoc
#--- Percentage of people in poverty by AGE x SEX ---#
p_sex_age <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "edad")

### Level of schooling (LoS) x SEX ###
p_sex_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "nivel_edu")

### State x SEX ###
p_sex_depto <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "ent")

#--- Patchwork in action ---#
(p_sex_age + p_sex_escolar) / p_sex_depto

### Level of schooling (LoS) x AGE ###
p_escolar_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "edad") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

### State x AGE ###
p_depto_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "edad",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "Edad")

p_escolar_edad / p_depto_edad

### Level of schooling (LoS) x State ###
p_depto_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

p_depto_escolar


### AGE x SEX ###

encuesta_sta$pobreza <- encuesta_sta$ic_ali_nc
#--- Percentage of people in poverty by AGE x SEX ---#
p_sex_age <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "edad")

### Level of schooling (LoS) x SEX ###
p_sex_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "nivel_edu")

### State x SEX ###
p_sex_depto <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "ent")

#--- Patchwork in action ---#
(p_sex_age + p_sex_escolar) / p_sex_depto

### Level of schooling (LoS) x AGE ###
p_escolar_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "edad") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

### State x AGE ###
p_depto_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "edad",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "Edad")

p_escolar_edad / p_depto_edad

### Level of schooling (LoS) x State ###
p_depto_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

p_depto_escolar
```
