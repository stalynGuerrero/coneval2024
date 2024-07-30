# 03_Estandarizar_muestra_cuestionario_ampliado.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **02_Union_predictores.R** disponible en la ruta *Rcodes/2020/02_Union_predictores.R*. Este script está diseñado para llevar a cabo un análisis detallado de datos mediante un proceso estructurado en varias fases.

En la etapa inicial, el código limpia el entorno de R eliminando todos los objetos existentes para comenzar con un espacio de trabajo limpio. A continuación, carga las bibliotecas necesarias, como `tidyverse` y `data.table`, que son fundamentales para la manipulación y análisis de datos, así como para la importación y exportación de archivos. Luego, el código lee las bases de datos correspondientes al censo de personas 2020 y a la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH). Estas bases se utilizan para identificar indicadores clave relacionados con carencia en salud, calidad de vivienda, servicios básicos y rezago educativo.

Seguidamente, el código valida y armoniza las variables entre ambas bases de datos para asegurar la consistencia en códigos y etiquetas. Esto incluye la comparación de variables como códigos de entidad y municipio, áreas urbanas y rurales, sexo, edad, nivel educativo, discapacidad y lengua indígena. Posteriormente, estandariza las variables del censo para alinearlas con las de la encuesta y filtra los datos para eliminar cualquier inconsistencia. Finalmente, resume el conjunto de datos estandarizado por grupo y guarda los datos procesados en dos versiones: una con los datos estandarizados y otra con los datos resumidos.

#### Lectura y Preparación de Datos {-}

En este bloque de código, se inicia el entorno de trabajo en R, cargando las bibliotecas necesarias y preparando los datos para el análisis.

Primero, se limpia el entorno de R con `rm(list = ls())` para asegurar que no haya variables o datos residuales que puedan interferir con el análisis. Luego, se cargan diversas bibliotecas que proporcionan herramientas para la manipulación de datos, la lectura de archivos, la exploración de datos y el análisis estadístico. Entre estas bibliotecas se encuentran `tidyverse`, `data.table`, `openxlsx`, `magrittr`, `DataExplorer`, `haven`, `purrr`, `labelled` y `sampling`.

A continuación, se lee el archivo `muestra_cuestionario_ampliado.rds` usando `readRDS()`, que contiene un dataframe con los datos ampliados del censo. También se lee el archivo `enigh.rds`, que contiene datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH) de 2020. Finalmente, se muestra la dimensión del dataframe `muestra_cuestionario_ampliado` con `dim()` para obtener una visión general de su tamaño y estructura.



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
library(sampling)
cat("\f")

################################################################################
# Lectura de bases censo persona 2020
################################################################################
muestra_cuestionario_ampliado <-
  readRDS(
    "../output/2020/muestra_cuestionario_ampliado.rds"
  )
enigh <- readRDS("../output/2020/enigh.rds")
muestra_cuestionario_ampliado %>% dim()
```

#### Identificando indicadores {-}


En este bloque de código, se realiza una revisión inicial de los indicadores clave presentes en el dataframe `muestra_cuestionario_ampliado` para comprender su contenido y estructura.

Se comienza revisando el indicador de carencia por acceso a los servicios de salud (`ic_asalud`) mediante la función `head()`, que muestra las primeras entradas de este indicador en el dataframe. Esto ayuda a obtener una visión general de los datos disponibles para este indicador específico.

Luego, se exploran las etiquetas asociadas a los atributos de los indicadores de carencia por calidad y espacios de la vivienda (`ic_cv`) y carencia por servicios básicos en la vivienda (`ic_sbv`) utilizando la función `attributes()` y `$label`. Esto proporciona información sobre las descripciones o categorías asociadas con estos indicadores, facilitando la comprensión de los datos.

Finalmente, se revisa el indicador de carencia por rezago educativo (`ic_rezedu`) de la misma manera, explorando sus etiquetas para entender las descripciones asociadas con este indicador.


``` r
# Indicador de carencia por acceso a los servicios de salud (CENSO)
head(muestra_cuestionario_ampliado$ic_asalud)

# Carencia por la calidad y espacios de la vivienda. (CENSO)
attributes(muestra_cuestionario_ampliado$ic_cv)$label	

# Carencia por servicios básicos en la vivienda. (CENSO)
attributes(muestra_cuestionario_ampliado$ic_sbv)$label

# Carencia por rezago educativo. 
attributes( muestra_cuestionario_ampliado$ic_rezedu)$label
```

#### Validaciones para la armonización de variables {-}

En este bloque de código, se realizan una serie de validaciones para asegurar la armonización de variables entre los datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH) y el censo ampliado.


Primero, se revisan los códigos de entidad (`ent`) y municipio (`cve_mun`) en ambos conjuntos de datos. Se utiliza `n_distinct()` para contar el número de valores únicos en cada variable, comparando así los códigos de entidad y municipio entre la encuesta y el censo.

A continuación, se analiza la variable `rururb`, que indica si un área es urbana o rural. Se exploran los atributos de esta variable en la encuesta y se utiliza `table()` para mostrar la distribución de esta variable en el censo, verificando los valores únicos y su correspondencia.

Luego, se validan las variables relacionadas con el sexo (`sexo`). Se revisan los atributos de esta variable en la encuesta y se utiliza `table()` para mostrar la distribución en el censo. Esto asegura que las categorías de sexo sean consistentes entre ambos conjuntos de datos.

Se revisa la variable de edad (`edad`). Se exploran los atributos de esta variable en la encuesta y se muestran las primeras entradas de la variable `edad_cat` en el censo para comparar y validar las categorías de edad.

Para la variable de nivel educativo (`niv_ed`), se comparan los atributos de esta variable en la encuesta y el censo. Se asegura que las categorías de nivel educativo sean consistentes en ambos conjuntos de datos.

Se valida la variable de discapacidad (`discap`). Se revisan los atributos de esta variable en ambos conjuntos de datos para determinar si se necesita realizar alguna conversión o armonización.

Finalmente, se revisa la variable que indica si la persona habla un dialecto o lengua indígena (`hli`). Se comparan los atributos de esta variable en la encuesta y el censo para asegurar que los datos sean coherentes.



``` r
## codigos de ent y municipio 

## Encuesta 
n_distinct(enigh$ent) # cod_dam
n_distinct(enigh$cve_mun) # cod_mun

## censo 
n_distinct(muestra_cuestionario_ampliado$ent)
n_distinct(muestra_cuestionario_ampliado$cve_mun)

################################################################################

## area 

## Encuesta 
attributes(enigh$rururb) 

## censo 
table(muestra_cuestionario_ampliado$rururb, useNA = "a")
# 0	Urbano
# 1	Rural


################################################################################

## Sexo 

## Encuesta 
attributes(enigh$sexo) 
# 2	Mujer
# 1	Hombre

## censo 
table(muestra_cuestionario_ampliado$sexo, useNA = "a")
# 0	Mujer
# 1	Hombre

################################################################################
## edad 

## Encuesta 
attributes(enigh$edad) 

## censo 
head(muestra_cuestionario_ampliado$edad_cat)

################################################################################
## nivel_edu # 

## Encuesta 
attributes(enigh$niv_ed) 

# 0 [Con primaria incompleta o menos]                 
# 1 [Primaria completa o secundaria incompleta]       
# 2 [Secundaria completa o media superior incompleta] 
# 3 [Media superior completa o mayor nivel educativo] 
# NA  Niños de 0 a 3 años

## censo 
attributes(muestra_cuestionario_ampliado$niv_ed)

# 0 [Con primaria incompleta o menos]                 
# 1 [Primaria completa o secundaria incompleta]       
# 2 [Secundaria completa o media superior incompleta] 
# 3 [Media superior completa o mayor nivel educativo] 
# NA  Niños de 0 a 3 años


################################################################################
## Discapacidad # 

## Encuesta 
attributes(enigh$discap) 

## censo
attributes(muestra_cuestionario_ampliado$discap) 


################################################################################
## Habla dialecto o lengua indígena

## Encuesta 
attributes(enigh$hli) 


## censo
attributes(muestra_cuestionario_ampliado$hli)
```

#### Estandarizando el censo {-}

Este bloque de código se centra en la estandarización y limpieza del conjunto de datos del censo para prepararlo para el análisis.

Primero, se crea un nuevo dataframe `censo_sta` a partir del dataframe `muestra_cuestionario_ampliado`. En este proceso, se realiza una transformación y estandarización de las variables. Se convierten las variables categóricas a tipo `character` o `factor`, y se manejan los valores faltantes. Específicamente:

- La variable `area` se convierte a tipo `character`.
- La variable `sexo` se estandariza para que los valores sean "1" para hombre y "2" para mujer.
- Las variables de edad, nivel educativo y discapacidad se transforman a tipo `character`, con los valores de discapacidad y lengua indígena establecidos en "0" si son `NA`.
- Las variables `ic_asalud`, `ic_cv`, `ic_sbv`, e `ic_rezedu` se convierten a tipo `factor` utilizando `haven::as_factor()`.

Después, se filtran ciertos municipios excluidos mediante `filter()`, eliminando aquellos con códigos específicos (04012, 07125, 29048).

A continuación, se crea un segundo dataframe `censo_sta2` a partir de `censo_sta`, filtrando los registros con valores faltantes en las variables de interés (`nivel_edu`, `edad`, `ic_sbv`, `ic_cv`, `ic_rezedu`, `ic_asalud`).

Finalmente, se calcula la proporción del total del `factor` en `censo_sta2` respecto a `censo_sta`, lo que permite verificar la representatividad del subconjunto filtrado. El dataframe estandarizado `censo_sta2` se guarda en un archivo `.rds` para su uso en análisis futuros.



``` r
censo_sta <- muestra_cuestionario_ampliado %>%
  transmute(
    ent,
    cve_mun,
    upm,
    estrato,
    area = as.character(rururb),
    sexo = if_else(sexo == 1, "1", "2"),
    edad = as.character(edad_cat),
    nivel_edu = haven::as_factor(niv_ed, levels  = "values"),
    discapacidad = as.character(discap),
    discapacidad = if_else(is.na(discapacidad), "0", discapacidad),
    hlengua = as.character(hli),
    hlengua = if_else(is.na(hlengua), "0", hlengua),
    
    ic_asalud= haven::as_factor(ic_asalud, levels  = "values"),
    ic_cv = haven::as_factor(ic_cv, levels  = "values"),
    ic_sbv = haven::as_factor(ic_sbv, levels  = "values"),
    ic_rezedu = haven::as_factor(ic_rezedu, levels  = "values"),
    factor
  ) %>% 
  filter( !cve_mun %in% c("04012", "07125", "29048"))




censo_sta2 <- censo_sta %>%
  filter(
    !is.na(nivel_edu),
    !is.na(edad),
    !is.na(ic_sbv),
    !is.na(ic_cv),
    !is.na(ic_rezedu),
    !is.na(ic_asalud),
  )

sum(censo_sta2$factor) / sum(censo_sta$factor)

#Se guarda el conjunto de datos estandarizado:

saveRDS(censo_sta2,"../output/2020/encuesta_ampliada.rds")
```

### Resumen y Guardado de Datos Agrupados{-}

Este bloque de código realiza un resumen de los datos estandarizados del censo y guarda el conjunto de datos resultante.

Primero, el dataframe `censo_sta2` se agrupa utilizando `group_by()` en base a múltiples variables: `ent`, `cve_mun`, `area`, `hlengua`, `sexo`, `edad`, `nivel_edu`, `discapacidad`, `ic_asalud`, `ic_cv`, `ic_sbv`, e `ic_rezedu`. Para cada grupo definido por estas variables, se calcula la suma de la variable `factor` utilizando `summarise()`. El resultado es un dataframe que contiene el total de observaciones (`n`) para cada combinación única de los grupos especificados.

Finalmente, el dataframe resumido se guarda en un archivo `.rds` en la ruta `"../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds"`. Este archivo puede ser utilizado para análisis posteriores, proporcionando una versión compacta y organizada del conjunto de datos.




``` r
#Se resumen los datos por grupo:

censo_sta2 %<>% group_by(
  ent,
  cve_mun,
  area,
  hlengua,
  sexo,
  edad,
  nivel_edu,
  discapacidad,
  ic_asalud,
  ic_cv,
  ic_sbv,
  ic_rezedu
) %>%
  summarise(n = sum(factor), .groups = "drop") 

#Se guarda el conjunto de datos resumido:

saveRDS(censo_sta2, "../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")
```
