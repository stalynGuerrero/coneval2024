# Lectura y consolidación de bases de datos



En el siguiente apartado se describe el proceso de integración y ordenamiento de las bases de datos proporcionadas por la Universidad Nacional Autónoma de México (UNAM) y el Consejo Nacional de Evaluación de la Política de Desarrollo Social (CONEVAL). Este proceso tiene como propósito asegurar el adecuado desarrollo de los scripts necesarios para el análisis.

## Consolidando la base de datos de la encuesta ampliada del censo 

### Limpieza del espacio de trabajo {-}
   Esta línea elimina todos los objetos del espacio de trabajo actual. Es útil para asegurar que no haya objetos residuales que puedan interferir con el nuevo análisis.


``` r
rm(list = ls())
```



### Carga de bibliotecas {-}
   Estas líneas cargan diversas bibliotecas en R:


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
```

### Definición de variables para validación {-}
   Se define un vector `validar_var` que contiene una lista de nombres de variables. Estos nombres podrían ser utilizados más adelante para validar o verificar datos en el análisis.

``` r
validar_var <- c("ic_rezedu",
                 "ic_asalud",
                 "ic_segsoc",
                 "ic_cv",
                 "ic_sbv",
                 "ic_ali_nc",
                 "ictpc")
```

### Obtener la lista de archivos .dta {-}
  Estas líneas generan listas de archivos con extensión `.dta` en los directorios especificados:
  
   - `file_muestra_censo_2020_estado`: Contiene la lista de archivos `.dta` en el directorio `SegSocial/SegSoc/`.
   
   - `file_muestra_censo_2020_estado_complemento`: Contiene la lista de archivos `.dta` en el directorio `Complemento_SegSoc/`.
   
   - `muestra_cuestionario_ampliado_censo_2020_estado`: Contiene la lista de archivos `.dta` en el directorio `IndicadoresCenso/`.


``` r
(file_muestra_censo_2020_estado <-
  list.files(
    "../input/2020/muestra_ampliada/SegSocial/SegSoc/",
    full.names = TRUE,
    pattern = "dta$"
  ))
```

```
##  [1] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_01.dta"
##  [2] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_02.dta"
##  [3] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_03.dta"
##  [4] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_04.dta"
##  [5] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_05.dta"
##  [6] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_06.dta"
##  [7] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_07.dta"
##  [8] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_08.dta"
##  [9] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_09.dta"
## [10] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_10.dta"
## [11] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_11.dta"
## [12] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_12.dta"
## [13] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_13.dta"
## [14] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_14.dta"
## [15] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_15.dta"
## [16] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_16.dta"
## [17] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_17.dta"
## [18] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_18.dta"
## [19] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_19.dta"
## [20] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_20.dta"
## [21] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_21.dta"
## [22] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_22.dta"
## [23] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_23.dta"
## [24] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_24.dta"
## [25] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_25.dta"
## [26] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_26.dta"
## [27] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_27.dta"
## [28] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_28.dta"
## [29] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_29.dta"
## [30] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_30.dta"
## [31] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_31.dta"
## [32] "../input/2020/muestra_ampliada/SegSocial/SegSoc/base_personas_32.dta"
```

``` r
(file_muestra_censo_2020_estado_complemento <-
  list.files(
    "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/",
    full.names = TRUE,
    pattern = "dta$"
  ))
```

```
##  [1] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc01.dta"
##  [2] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc02.dta"
##  [3] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc03.dta"
##  [4] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc04.dta"
##  [5] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc05.dta"
##  [6] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc06.dta"
##  [7] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc07.dta"
##  [8] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc08.dta"
##  [9] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc09.dta"
## [10] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc10.dta"
## [11] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc11.dta"
## [12] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc12.dta"
## [13] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc13.dta"
## [14] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc14.dta"
## [15] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc15.dta"
## [16] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc16.dta"
## [17] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc17.dta"
## [18] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc18.dta"
## [19] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc19.dta"
## [20] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc20.dta"
## [21] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc21.dta"
## [22] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc22.dta"
## [23] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc23.dta"
## [24] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc24.dta"
## [25] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc25.dta"
## [26] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc26.dta"
## [27] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc27.dta"
## [28] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc28.dta"
## [29] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc29.dta"
## [30] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc30.dta"
## [31] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc31.dta"
## [32] "../input/2020/muestra_ampliada/SegSocial/Complemento_SegSoc/Complemento_segsoc32.dta"
```

``` r
(muestra_cuestionario_ampliado_censo_2020_estado <-
  list.files(
    "../input/2020/muestra_ampliada/IndicadoresCenso/",
    full.names = TRUE,
    pattern = "dta$"
  ))
```

```
##  [1] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_01.dta"
##  [2] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_02.dta"
##  [3] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_03.dta"
##  [4] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_04.dta"
##  [5] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_05.dta"
##  [6] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_06.dta"
##  [7] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_07.dta"
##  [8] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_08.dta"
##  [9] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_09.dta"
## [10] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_10.dta"
## [11] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_11.dta"
## [12] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_12.dta"
## [13] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_13.dta"
## [14] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_14.dta"
## [15] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_15.dta"
## [16] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_16.dta"
## [17] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_17.dta"
## [18] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_18.dta"
## [19] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_19.dta"
## [20] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_20.dta"
## [21] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_21.dta"
## [22] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_22.dta"
## [23] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_23.dta"
## [24] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_24.dta"
## [25] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_25.dta"
## [26] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_26.dta"
## [27] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_27.dta"
## [28] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_28.dta"
## [29] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_29.dta"
## [30] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_30.dta"
## [31] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_31.dta"
## [32] "../input/2020/muestra_ampliada/IndicadoresCenso/CarSoc_32.dta"
```

### Conslidar bases de la muestra ampliada {-}

El código inicia creando un dataframe vacío y cargando las bibliotecas necesarias para el análisis de datos. Luego, itera sobre los archivos `.dta` para leer y combinar datos de diferentes fuentes, realizando uniones internas y ajustes según sea necesario. Cada conjunto de datos combinado se guarda en archivos `.rds` individuales durante el proceso. Después, los datos combinados de cada archivo se acumulan en un dataframe principal. Finalmente, el dataframe completo se guarda en un archivo `.rds`. Este procedimiento facilita la integración y el análisis de grandes volúmenes de datos provenientes de diversas fuentes.



``` r
# Crear un dataframe vacío para almacenar los datos
df <- data.frame()

# Iterar sobre cada archivo y leerlo
for (ii in 1:32) {
  muestra_censo_2020_estado_ii <-
    read_dta(file_muestra_censo_2020_estado[ii])
 
   muestra_censo_2020_estado_complemento_ii <-
    read_dta(file_muestra_censo_2020_estado_complemento[ii]) %>%
    mutate(id_per = id_persona)
  
   muestra_cuestionario_ampliado_censo_2020_estado_ii <-
    read_dta(muestra_cuestionario_ampliado_censo_2020_estado[ii])
  
  muestra_censo <-
    inner_join(muestra_censo_2020_estado_ii,
               muestra_censo_2020_estado_complemento_ii) %>%
    select(-tamloc) %>%
    inner_join(muestra_cuestionario_ampliado_censo_2020_estado_ii)
  
  saveRDS(muestra_censo,
          paste0("output/2020/muestra_censo/depto_", ii, ".rds"))
  
  df <- bind_rows(df, muestra_censo)
  cat(file_muestra_censo_2020_estado[ii], "\n")
}

# Guardar el dataframe en un archivo .rds
saveRDS(df, file = "output/2020/muestra_cuestionario_ampliado.rds")
```

## Cambiando de formato .dta a .rds la base de la ENIGH. 

El código se enfoca en la integración y preparación de datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH). Primero, lee archivos de datos relevantes para los hogares y la pobreza desde fuentes especificadas. Luego, combina estos datos mediante uniones internas, basándose en identificadores comunes para asegurar una correcta correspondencia entre registros. Se ajustan algunas variables, como la creación de una nueva columna para los datos de ictpc, y se elimina una columna innecesaria. Finalmente, el dataframe combinado se guarda en un archivo .rds para su posterior análisis. Este proceso facilita la consolidación y preparación de los datos de la ENIGH para análisis más detallados.


``` r
enigh_hogares <- read_dta("../input/2020/enigh/base_hogares20.dta")
enigh_pobreza <- read_dta("../input/2020/enigh/pobreza_20.dta") %>% 
  select(-ent)

n_distinct(enigh_pobreza$folioviv)
n_distinct(enigh_hogares$folioviv)

enigh <- inner_join(
  enigh_pobreza,
  enigh_hogares,
  by = join_by(
    folioviv, foliohog, est_dis, upm,
    factor, rururb, ubica_geo
  ), 
  suffix = c("_pers", "_hog"),
)

enigh$ic_ali_nc
enigh$ictpc_pers
enigh$ictpc <- enigh$ictpc_pers
enigh$ic_segsoc

#saveRDS(enigh, file = "output/2020/enigh.rds")
```

