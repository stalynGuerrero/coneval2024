# 00_union_bases.R



Para la ejecución del presente archivo, debe abrir el archivo **00_union_bases.R** disponible en la ruta *Rcodes/2020/00_union_bases.R*.

El código comienza limpiando el entorno de R y cargando varias bibliotecas esenciales para la manipulación y análisis de datos. Posteriormente, define un conjunto de variables a validar y obtiene listas de archivos de datos en formato `.dta` correspondientes a diferentes conjuntos de datos del censo 2020. A continuación, el código itera sobre estos archivos para leer, combinar y almacenar los datos de cada estado en archivos `.rds` individuales. Estos datos se combinan en un único dataframe que se guarda para su uso posterior.

En la sección final, el código se centra en los datos de la ENIGH 2020. Lee los archivos de hogares y pobreza, y los combina utilizando un `inner_join`. Se seleccionan y renombran variables clave para asegurar la consistencia en el análisis. Finalmente, el dataframe combinado se guarda en un archivo `.rds` para facilitar el acceso y análisis futuros.



#### Limpieza del Entorno y Carga de Bibliotecas {-}

Se limpia el entorno de R eliminando todos los objetos y se ejecuta el recolector de basura para liberar memoria.


``` r
rm(list = ls())
gc()
```

Se cargan varias bibliotecas esenciales para la manipulación y análisis de datos, incluyendo `tidyverse`, `data.table`, `haven` y otras.


``` r
library(tidyverse)
library(data.table)
library(openxlsx)
library(magrittr)
library(DataExplorer)
library(haven)
library(purrr)
library(furrr)
library(labelled)
cat("\f")
```


#### Configuración de la Memoria {-}
Se define un límite para la memoria RAM a utilizar, en este caso 250 GB.


``` r
memory.limit(250000000)
```


#### Definición de Variables y Obtención de Archivos{-}


``` r
validar_var <- c(
  "ic_rezedu",
  "ic_asalud",
  "ic_segsoc",
  "ic_cv",
  "ic_sbv",
  "ic_ali_nc",
  "ictpc"
)
```
Se define un conjunto de variables que serán validadas.


``` r
file_muestra_censo_2020_estado <- list.files(
  "../input/2020/muestra_ampliada/SegSocial/SegSoc/",
  full.names = TRUE,
  pattern = "dta$"
)
file_muestra_censo_2020_estado_complemento <- list.files(
  "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/",
  full.names = TRUE,
  pattern = "dta$"
)
muestra_cuestionario_ampliado_censo_2020_estado <- list.files(
  "../input/2020/muestra_ampliada/IndicadoresCenso/",
  full.names = TRUE,
  pattern = "dta$"
)
```
Se obtienen listas de archivos `.dta` correspondientes a diferentes conjuntos de datos del censo 2020.

#### Lectura y Combinación de Datos del Censo{-}


``` r
df <- data.frame()

for (ii in 1:32) {
  muestra_censo_2020_estado_ii <- read_dta(file_muestra_censo_2020_estado[ii])
  muestra_censo_2020_estado_complemento_ii <- read_dta(file_muestra_censo_2020_estado_complemento[ii]) %>%
    mutate(id_per = id_persona)
  muestra_cuestionario_ampliado_censo_2020_estado_ii <- read_dta(muestra_cuestionario_ampliado_censo_2020_estado[ii])
  
  muestra_censo <- inner_join(muestra_censo_2020_estado_ii, muestra_censo_2020_estado_complemento_ii) %>%
    select(-tamloc) %>%
    inner_join(muestra_cuestionario_ampliado_censo_2020_estado_ii)
  
  saveRDS(muestra_censo, paste0("../output/2020/muestra_censo/depto_", ii, ".rds"))
  
  df <- bind_rows(df, muestra_censo)
  cat(file_muestra_censo_2020_estado[ii], "\n")
}
```
Se iteran los archivos de datos del censo, se combinan y se almacenan en archivos `.rds` individuales y en un dataframe único.


``` r
saveRDS(df, file = "../output/2020/muestra_cuestionario_ampliado.rds")
```
Se guarda el dataframe combinado en un archivo `.rds`.



#### Lectura y Combinación de Datos de la ENIGH{-}


``` r
enigh_hogares <- read_dta("../input/2020/enigh/base_hogares20.dta")
enigh_pobreza <- read_dta("../input/2020/enigh/pobreza_20.dta") %>% 
  select(-ent)
```
Se leen los datos de hogares y pobreza de la ENIGH 2020.


``` r
n_distinct(enigh_pobreza$folioviv)
n_distinct(enigh_hogares$folioviv)
```
Se verifica la unicidad de los identificadores de viviendas.


``` r
enigh <- inner_join(
  enigh_pobreza,
  enigh_hogares,
  by = join_by(
    folioviv, foliohog, est_dis, upm,
    factor, rururb, ubica_geo
  ), 
  suffix = c("_pers", "_hog")
)
```
Se combinan los datos de pobreza y hogares utilizando un `inner_join`.

#### Selección y Renombramiento de Variables {-}

``` r
enigh$ic_ali_nc
enigh$ictpc_pers
enigh$ictpc <- enigh$ictpc_pers
enigh$ic_segsoc
```
Se seleccionan y renombran variables clave para asegurar la consistencia en el análisis.

#### Guardado del DataFrame Combinado{-}

``` r
saveRDS(enigh, file = "../output/2020/enigh.rds")
```
Se guarda el dataframe combinado en un archivo `.rds` para facilitar el acceso y análisis futuros.
