# Estandarizar muestra cuestionario ampliado. 



En esta sección se realiza la estandarización del formulario ampliado. El objetivo de este apartado es armonizar la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH) con la información disponible en la muestra ampliada del censo. De esta manera, se busca obtener una coincidencia precisa entre las variables predictoras de la ENIGH y las de la muestra ampliada, asegurando la comparabilidad y coherencia entre ambas fuentes de datos.

# Limpieza del entorno de R {.unnumbered}

Inicialmente se eliminan todos los objetos del entorno de R para asegurar que el análisis comience con una pizarra limpia.


``` r
rm(list = ls())
```

#### Carga de librerías {.unnumbered}

Se cargan las bibliotecas necesarias para el análisis de datos. Estas bibliotecas proporcionan herramientas para la manipulación de datos, importación/exportación de archivos y otras funciones útiles:


``` r
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
```



####  Lectura de Bases {.unnumbered}

Se leen las bases de datos de la muestra ampliada  y ENIGH. Estas bases se utilizarán para identificar y armonizar variables:


``` r
muestra_cuestionario_ampliado <-
  readRDS(
    "../output/2020/muestra_cuestionario_ampliado.rds"
  )
enigh <- readRDS("../output/2020/enigh.rds")
muestra_cuestionario_ampliado %>% dim()
```

#### Identificación de Indicadores {.unnumbered}

Se identifican y verifican diferentes indicadores del censo. Estos indicadores se relacionan con carencias en acceso a servicios de salud, calidad de vivienda, servicios básicos en la vivienda y rezago educativo:


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

####  Validaciones para la Armonización de Variables {.unnumbered}

Se validan las variables para asegurar la armonización entre las diferentes bases de datos, verificando que los códigos y etiquetas sean consistentes entre el censo y la encuesta. Se comparan los códigos de entidad y municipio entre la encuesta y el censo para asegurar la consistencia; Se validan los códigos que representan áreas urbanas y rurales; Se comparan los códigos de sexo, edad, nivel educativo, discapacidad y habla dialecto o lengua indígena entre la encuesta y el censo.


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

#### Estandarización del Censo {.unnumbered}

Se estandarizan las variables del censo para que coincidan con las de la encuesta, permitiendo una comparación y análisis coherentes:


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
```

El siguiente bloque de código tiene como objetivo filtrar los datos del censo para eliminar registros con valores faltantes en variables clave y calcular la proporción de los factores de expansión antes y después de la filtración.



``` r
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
```

Se guarda el conjunto de datos estandarizado:


``` r
saveRDS(censo_sta2,"../output/2020/encuesta_ampliada.rds")
```

Se resumen los datos por grupo:


``` r
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
```

Se guarda el conjunto de datos resumido:


``` r
saveRDS(censo_sta2, "../input/2020/muestra_ampliada/muestra_cuestionario_ampliado.rds")
```
