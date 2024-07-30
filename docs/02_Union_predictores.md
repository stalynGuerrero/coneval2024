# 02_Union_predictores.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **02_Union_predictores.R** disponible en la ruta *Rcodes/2020/02_Union_predictores.R*.

Este script en R está orientado a la integración y limpieza de bases de datos a nivel municipal para el año 2020, combinando datos satelitales y de contexto. Primero, el entorno de trabajo se limpia y se cargan las bibliotecas necesarias para la manipulación y análisis de datos, incluyendo tidyverse, data.table, y openxlsx.

En la primera parte, se carga una base de datos satelital en formato RDS, se renombra una columna para alinear los nombres y se eliminan columnas no necesarias. Luego, se obtienen los códigos municipales únicos de esta base de datos. A continuación, se carga una base de datos adicional en formato Stata (.dta), se convierten algunas variables a factores y se escalan las variables numéricas. Se realiza un análisis para identificar códigos municipales presentes en la base de datos satelital pero no en la de contexto, y se comprueba si hay columnas sin valores faltantes en la base de datos de contexto. Finalmente, se realiza un inner_join para combinar ambas bases de datos utilizando solo las columnas sin valores faltantes y se guarda el resultado en un archivo RDS para futuros análisis.

#### Configuración Inicial y Carga de Librerías{-}

Este bloque de código está dedicado a la limpieza del entorno de trabajo y a la carga de las librerías necesarias para el análisis de datos en R.

Primero, `rm(list = ls())` se utiliza para eliminar todos los objetos en el entorno de trabajo de R, asegurando que no haya datos o variables residuales que puedan interferir con el análisis actual.

A continuación, se cargan varias librerías esenciales:

- **`tidyverse`**: Un conjunto de paquetes que incluye `ggplot2`, `dplyr`, `tidyr`, entre otros, para la manipulación y visualización de datos.
- **`data.table`**: Ofrece una manera rápida y eficiente de manejar y manipular datos en grandes conjuntos.
- **`openxlsx`**: Utilizado para leer y escribir archivos Excel.
- **`magrittr`**: Proporciona operadores para la manipulación fluida de datos, como `%>%`.
- **`DataExplorer`**: Facilita la exploración y visualización rápida de datos.
- **`haven`**: Permite la importación y exportación de datos en formatos utilizados por otros paquetes estadísticos, como Stata y SPSS.
- **`purrr`**: Ofrece herramientas para la programación funcional, como la aplicación de funciones a listas o vectores.
- **`labelled`**: Maneja y procesa datos con etiquetas, comúnmente utilizados en encuestas.

Finalmente, `cat("\f")` limpia la consola para asegurar que el entorno esté ordenado antes de comenzar el análisis.


``` r
### Cleaning R environment ###
rm(list = ls())

#################
### Libraries ###
#################
library(tidyverse)
library(data.table)
library(openxlsx)
library(magrittr)
library(DataExplorer)
library(haven)
library(purrr)
library(labelled)
cat("\f")
```

#### Lectura de bases de contexto 2020 {-}



Este bloque de código se encarga de cargar y procesar datos adicionales relacionados con el contexto municipal para el año 2020, que serán utilizados para enriquecer el análisis de datos.

**Carga y Procesamiento de Datos Satelitales:**

Se carga una base de datos previamente guardada en formato `.rds` que contiene predictores satelitales a nivel municipal. La columna "CVEGEO" se renombra a "cve_mun" para estandarizar los nombres de las variables, y se elimina la columna "ent" que no es necesaria para el análisis actual. Luego, se extraen los códigos municipales únicos de esta base de datos para su posterior uso.



``` r
## Covariables
# Contexto_munXX : la base de datos contexto_20XX.dta contiene información de 
# variables a nivel municipal. 

# Cargar la base de datos satelital y renombrar la columna "CVEGEO" a "cve_mun"
satelital <-
  readRDS("../input/2020/predictores/statelevel_predictors_satelite.rds") %>%
  rename(cve_mun  = CVEGEO) %>% mutate(ent = NULL)

# Obtener los códigos municipales únicos de la base de datos satelital
cod_mun <- satelital %>% distinct(cve_mun)
```

**Carga y Exploración de Datos Contextuales:**

Se carga una base de datos en formato Stata (`contexto_2020.dta`) que contiene variables contextuales a nivel municipal. La función `as_factor()` se usa para asegurar que las variables sean tratadas como factores, y `apply(., 2, n_distinct) %>% sort()` se emplea para obtener y ordenar el número de valores únicos por variable. Esto ayuda a entender la estructura de los datos y la cantidad de información contenida en cada variable.



``` r
# Cargar la base de datos "Contexto_mun20" en formato Stata y renombrar la columna "CVEGEO" a "cve_mun"
Contexto_mun <- read_dta("../input/2020/predictores/contexto_2020.dta")
Contexto_mun %>% as_factor() %>% 
  apply(., 2, n_distinct) %>% sort()

Contexto_mun$elec_mun19
Contexto_mun$elec_mun20
Contexto_mun$smg1
Contexto_mun$ql_porc_cpa_urb
Contexto_mun$ql_porc_cpa_rur   
Contexto_mun$dec_geologicas 
Contexto_mun$dec_hidrometeo
```

#### Procesamiento y Unión de Datos Contextuales{-}

Este bloque de código se centra en la preparación final y la combinación de datos contextuales a nivel municipal, asegurando que se utilicen solo las variables relevantes y sin valores faltantes.



1. **Estandarización y Escalado de Variables Contextuales:**
   La base de datos `Contexto_mun` se transforma para asegurar que las variables categóricas sean tratadas como factores, mientras que las variables numéricas se escalan. Se utilizan las funciones `mutate_at` y `mutate_if` para realizar estas transformaciones en las variables especificadas y en todas las variables numéricas, respectivamente.

2. **Identificación de Códigos Municipales Inexistentes:**
   Se realiza una operación de `anti_join` entre los códigos municipales de la base de datos satelital (`cod_mun`) y la base de datos de contexto (`Contexto_mun`). Esta operación identifica los códigos municipales que están presentes en la base de datos satelital pero que no se encuentran en la base de datos de contexto.

3. **Detección de Columnas sin Valores Faltantes:**
   Se analiza cada columna de `Contexto_mun` para verificar si tiene valores faltantes utilizando `apply` y `any(is.na(x))`. El resultado es una lista que indica si cada columna contiene valores faltantes o no. La función `table(paso)` se utiliza para mostrar la frecuencia de columnas sin valores faltantes.

4. **Unión de Bases de Datos:**
   Se realiza una `inner_join` entre las bases de datos `satelital` y `Contexto_mun`, pero solo utilizando las columnas sin valores faltantes, como se determinó en el paso anterior. Esto garantiza que la base de datos resultante solo incluya variables completas y coherentes.

5. **Guardado del Conjunto de Datos Combinado:**
   Finalmente, la base de datos combinada `statelevel_predictors` se guarda en un archivo `.rds` para su uso futuro.


``` r
Contexto_mun %<>% as_factor() %>%
  mutate_at(
    .vars = c(
      "elec_mun19",
      "elec_mun20",
      "smg1",
      "ql_porc_cpa_urb",
      "ql_porc_cpa_rur",
      "dec_geologicas",
      "dec_hidrometeo"
    ),
    as.factor
  ) %>%
  mutate_if(is.numeric, function(x)
    as.numeric(scale(x)))

# Realizar una anti_join para identificar códigos municipales presentes en "cod_mun" pero no en "Contexto_mun"
anti_join(cod_mun, Contexto_mun)

# Realizar un análisis para determinar si alguna columna de "Contexto_mun" no tiene valores faltantes
paso <- apply(Contexto_mun, 2, function(x) !any(is.na(x)))

# Mostrar la frecuencia de columnas sin valores faltantes
table(paso)

# Realizar una inner_join entre las bases de datos "satelital" y "Contexto_mun" utilizando solo las columnas sin valores faltantes
statelevel_predictors <- inner_join(satelital, Contexto_mun[, paso])

# Obtener las dimensiones de la base de datos resultante
dim(statelevel_predictors)

## Guardar 
saveRDS(statelevel_predictors,file = "../input/2020/predictores/statelevel_predictors_df.rds")
```

