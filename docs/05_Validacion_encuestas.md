# 05_Validacion_encuestas.R {-}

Para ejecutar este archivo, es necesario abrir el archivo **05_Validacion_encuestas.R** ubicado en la ruta *Rcodes/2020/05_Validacion_encuestas.R*. Este script está diseñado para llevar a cabo un análisis integral de datos con varias fases que aseguran una evaluación rigurosa de la información disponible.

En la primera etapa del código, se eliminan todos los objetos del entorno de R para garantizar que el análisis se realice en un entorno limpio y sin interferencias. Luego, se cargan las librerías necesarias para llevar a cabo el análisis y se establece el tema de visualización predeterminado usando `bayesplot::theme_default()`. También se carga un archivo de script adicional que incluye funciones específicas para la creación de gráficos. Posteriormente, se leen las bases de datos `encuesta_sta` y `muestra_ampliada`, que contienen los datos de la encuesta y del censo, respectivamente.

El análisis procede realizando comparaciones entre los datos censales y los datos de la encuesta en diferentes variables categóricas, tales como edad, sexo, nivel educativo y estado. Para cada una de estas variables, se generan gráficos que permiten visualizar las diferencias. Utilizando la librería `patchwork`, se combinan estos gráficos en un solo diseño para facilitar la comparación. Además, se llevan a cabo análisis adicionales en variables como lengua indígena y discapacidad, y se exploran las interacciones entre variables como sexo y edad, nivel educativo y sexo, y estado y sexo. Finalmente, los gráficos que ilustran estas interacciones se integran en un diseño consolidado para una visualización comprensiva de los efectos.


#### Limpieza del Entorno y Carga de Bibliotecas{-}

Se limpia el entorno de R y se cargan las bibliotecas necesarias para la manipulación de datos y la creación de gráficos.


``` r
#### Cleaning R environment ###
rm(list = ls())

#################
#### Libraries ###
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
```

#### Lectura de Datos {-}

Se leen las bases de datos necesarias para el análisis.


``` r
encuesta_sta <- readRDS("../input/2020/enigh/encuesta_sta.rds")
muestra_ampliada <- readRDS("../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")
```

#### Comparación de Variables {-}

Se comparan las distribuciones de diferentes variables entre los datos censales y los de la encuesta utilizando la función `Plot_Compare`.

#### Comparación de Edad, Sexo y Nivel Educativo {-}


``` r
#### AGE ###
age_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "edad")
#### Sex ###
sex_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "sexo")
#### Level of schooling (LoS) ###
escolar_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "nivel_edu")

#### States ###
depto_plot <-
  Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "ent")
```

Se crean gráficos comparativos para edad, sexo, nivel educativo y estados. Estos gráficos se combinan utilizando `patchwork`.


``` r
#--- Patchwork en acción ---#
(age_plot | sex_plot | escolar_plot) / (depto_plot)
```

#### Comparación de Lengua Indígena y Discapacidad {-}


``` r
hlengua <- Plot_Compare(dat_censo = muestra_ampliada,
               dat_encuesta = encuesta_sta,
               by = "hlengua")

plot_discapacidad <- Plot_Compare(dat_censo = muestra_ampliada,
             dat_encuesta = encuesta_sta,
             by = "discapacidad")

(plot_discapacidad | hlengua )
```

#### Efectos de Interacción {-}

Se analizan los efectos de interacción entre diferentes variables utilizando la función `plot_interaction`.

#### Interacción Edad x Sexo {-}


``` r
#### AGE x SEX ###
encuesta_sta$pobreza <- encuesta_sta$ictpc
#--- Percentage of people in poverty by AGE x SEX ---#
p_sex_age <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "edad")
```

#### Interacción Nivel Educativo x Sexo {-}


``` r
#### Level of schooling (LoS) x SEX ###
p_sex_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "nivel_edu")
```

#### Interacción Estado x Sexo {-}


``` r
#### State x SEX ###
p_sex_depto <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "ent")
```

Se combinan los gráficos de interacción utilizando `patchwork`.


``` r
#--- Patchwork in action ---#
(p_sex_age + p_sex_escolar) / p_sex_depto
```

#### Interacción Nivel Educativo x Edad {-}


``` r
#### Level of schooling (LoS) x AGE ###
p_escolar_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "edad") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")
```

#### Interacción Estado x Edad {-}


``` r
#### State x AGE ###
p_depto_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "edad",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "Edad")

p_escolar_edad / p_depto_edad
```

#### Interacción Nivel Educativo x Estado {-}


``` r
#### Level of schooling (LoS) x State ###
p_depto_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

p_depto_escolar
```

#### Repetición del Análisis para Diferentes Indicadores de Pobreza {-}

El análisis de interacción se repite para diferentes indicadores de pobreza (`ic_segsoc` e `ic_ali_nc`), utilizando el mismo procedimiento descrito anteriormente.


``` r
#### AGE x SEX ###
encuesta_sta$pobreza <- encuesta_sta$ic_segsoc
#--- Percentage of people in poverty by AGE x SEX ---#
p_sex_age <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "edad")

#### Level of schooling (LoS) x SEX ###
p_sex_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "nivel_edu")

#### State x SEX ###
p_sex_depto <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "ent")

#--- Patchwork in action ---#
(p_sex_age + p_sex_escolar) / p_sex_depto

#### Level of schooling (LoS) x AGE ###
p_escolar_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "edad") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

#### State x AGE ###
p_depto_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "edad",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "Edad")

p_escolar_edad / p_depto_edad

#### Level of schooling (LoS) x State ###
p_depto_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

p_depto_escolar

#### AGE x SEX ###
encuesta_sta$pobreza <- encuesta_sta$ic_ali_nc
#--- Percentage of people in poverty by AGE x SEX ---#
p_sex_age <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "edad")

#### Level of schooling (LoS) x SEX ###
p_sex_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "nivel_edu")

#### State x SEX ###
p_sex_depto <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "sexo",
                   by2 = "ent")

#--- Patchwork in action ---#
(p_sex_age + p_sex_escolar) / p_sex_depto

#### Level of schooling (LoS) x AGE ###
p_escolar_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "edad") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

#### State x AGE ###
p_depto_edad <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "edad",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "Edad")

p_escolar_edad / p_depto_edad

#### Level of schooling (LoS) x State ###
p_depto_escolar <-
  plot_interaction(dat_encuesta = encuesta_sta,
                   by = "nivel_edu",
                   by2 = "ent") +
  theme(legend.position = "bottom") + labs(colour = "nivel_edu")

p_depto_escolar
```

Este análisis detallado permite verificar la consistencia de los datos de la encuesta en comparación con los datos censales y entender mejor las interacciones entre diferentes variables demográficas y socioeconómicas.


