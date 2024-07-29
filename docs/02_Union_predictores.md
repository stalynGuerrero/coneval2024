# Integración de bases de datos predictoras.



Este código se centra en la selección y procesamiento de variables de covariables a nivel municipal para el año 2020. A continuación, se describe cada bloque del código en términos generales.

#### Limpieza del Entorno de R {.unnumbered}

Para asegurar un entorno limpio y evitar conflictos con datos previos, se eliminan todas las variables existentes en el entorno de R usando `rm(list = ls())`. Esto permite que el análisis comience sin datos residuales que puedan interferir.


``` r
rm(list = ls())
```

#### Carga de librerías {.unnumbered}

Se cargan varias librerías esenciales para la manipulación y análisis de datos. `tidyverse` se utiliza para la manipulación de datos y visualización; `data.table` para trabajar con grandes conjuntos de datos en formato de tabla; `openxlsx` para la lectura y escritura de archivos Excel; `magrittr` proporciona el operador de tubería (`%>%`) para una sintaxis más clara; `DataExplorer` ayuda en la exploración y visualización de datos; `haven` permite la importación de archivos de datos en formatos como Stata; `purrr` se utiliza para aplicar funciones a listas y vectores; y `labelled` facilita el manejo de variables etiquetadas. Finalmente, `cat("\f")` limpia la consola para un entorno de trabajo más ordenado.


``` r
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



#### Lectura de Bases de Datos de Contexto {.unnumbered}

La sección de lectura de bases de datos comienza cargando los datos satelitales desde un archivo RDS. Posteriormente, se extraen los códigos municipales únicos de esta base de datos, lo que permite identificar todos los municipios representados en los datos satelitales.

Luego, se carga la base de datos "Contexto_mun20" en formato Stata, que contiene información de diversas variables a nivel municipal. Al convertir estas variables en factores y revisar el número de categorías distintas en cada una, se puede entender mejor la estructura de los datos. Este paso es crucial para preparar los datos para el análisis posterior.


``` r
satelital <-
  readRDS("../input/2020/predictores/statelevel_predictors_satelite.rds") 

# Obtener los códigos municipales únicos de la base de datos satelital
cod_mun <- satelital %>% distinct(cve_mun)

# Cargar la base de datos "Contexto_mun20" en formato Stata y renombrar la columna "CVEGEO" a "cve_mun"
Contexto_mun <- read_dta("../input/2020/predictores/contexto_2020.dta")
Contexto_mun %>% as_factor() %>% 
  apply(., 2, n_distinct) %>% sort()
```

```
##             elec_mun19             elec_mun20                   smg1 
##                      2                      2                      2 
##        ql_porc_cpa_urb        ql_porc_cpa_rur         dec_geologicas 
##                      5                      6                      9 
##         dec_hidrometeo            vabpc_15_19            desem_15_20 
##                     20                     32                     32 
##     transf_gobpc_15_20           itlpis_15_20       derhab_pea_15_20 
##                     32                     32                     32 
##          remespc_15_20            acc_muybajo              acc_medio 
##                     32                    275                    887 
##            prop_sucban        prop_hospitales          edad65mas_urb 
##                   1236                   1248                   1272 
##          edad65mas_rur            altitud1000         porc_10sal_urb 
##                   1429                   1447                   1478 
##            pob_ind_urb         porc_10sal_rur          TasaDesempurb 
##                   1591                   1614                   1617 
##                pob_urb porc_patnoagrnocal_urb               porc_rur 
##                   1619                   1626                   1632 
##               porc_urb         porc_cpagr_urb    porc_hogremesas_urb 
##                   1632                   1647                   1652 
##           porc_jub_urb                 po_urb            porc_expurb 
##                   1652                   1656                   1656 
##     porc_norep_ing_urb      porc_hogayotr_urb      porc_ing_ilpi_urb 
##                   1656                   1657                   1658 
##     porc_ic_asalud_urb     porc_hogprogob_urb       prom_esc_rel_urb 
##                   1658                   1659                   1659 
##               acc_alto              edad65mas porc_patnoagrnocal_rur 
##                   1826                   1883                   2005 
##               acc_bajo          TasaDesemprur            pob_ind_rur 
##                   2024                   2084                   2168 
##                pob_rur           porc_jub_rur    porc_hogremesas_rur 
##                   2264                   2310                   2319 
##            acc_muyalto      porc_hogayotr_rur         porc_cpagr_rur 
##                   2331                   2343                   2345 
##            porc_exprur     porc_norep_ing_rur            t_incdelic1 
##                   2356                   2368                   2382 
##      porc_ing_ilpi_rur                pob_tot             prop_clues 
##                   2386                   2390                   2401 
##                 po_rur     porc_hogprogob_rur     porc_ic_asalud_rur 
##                   2405                   2408                   2417 
##       prom_esc_rel_rur          porc_segsoc15                gini15m 
##                   2439                   2449                   2451 
##             porc_ali15                  plp15                ictpc15 
##                   2452                   2452                   2453 
##                cve_mun 
##                   2466
```

#### Procesamiento y Escalado de Variables {.unnumbered}

En esta sección, el código procesa la base de datos `Contexto_mun` para asegurar que las variables categóricas sean correctamente identificadas y que las variables numéricas sean escaladas. Primero, se convierte toda la base de datos a factores usando `as_factor()`. Luego, se seleccionan variables específicas (`elec_mun19`, `elec_mun20`, `smg1`, `ql_porc_cpa_urb`, `ql_porc_cpa_rur`, `dec_geologicas`, `dec_hidrometeo`) y se aseguran que se manejen como factores. Además, cualquier columna numérica se escala usando `scale()`, lo que transforma estas variables para que tengan media cero y desviación estándar uno.


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
```

#### Identificación de Códigos Municipales Faltantes {.unnumbered}

El siguiente paso implica una `anti_join` entre `cod_mun` y `Contexto_mun`. Esto permite identificar los códigos municipales presentes en la base de datos satelital que no están presentes en `Contexto_mun`. Esta operación es útil para detectar inconsistencias o faltantes en los datos municipales que podrían impactar en los análisis posteriores.


``` r
anti_join(cod_mun, Contexto_mun) %>% 
  tba(cap = "Tabla de municipios sin información")
```

<table class="table table-striped lightable-classic" style="width: auto !important; margin-left: auto; margin-right: auto; font-family: Arial Narrow; width: auto !important; margin-left: auto; margin-right: auto;">
<caption>(\#tab:unnamed-chunk-6)Tabla de municipios sin información</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> cve_mun </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> 04012 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 07125 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 29048 </td>
  </tr>
</tbody>
</table>

#### Verificación de Columnas Completas {.unnumbered}

Para garantizar que sólo se utilicen columnas sin valores faltantes, se aplica una función que verifica si alguna columna en `Contexto_mun` tiene valores `NA`. El resultado es almacenado en `paso`, y la frecuencia de columnas completas es mostrada con `table(paso)`. Este paso asegura que las columnas usadas en el análisis posterior están completas y no contienen datos faltantes.


``` r
paso <- apply(Contexto_mun, 2, function(x) !any(is.na(x)))
table(paso)
```



| FALSE| TRUE|
|-----:|----:|
|     2|   65|

#### Combinación de Bases de Datos {.unnumbered}

Se realiza una `inner_join` entre las bases de datos `satelital` y `Contexto_mun`, utilizando únicamente las columnas sin valores faltantes identificadas previamente. Esta combinación asegura que se unan únicamente los datos completos de ambas bases, resultando en una base de datos consolidada `statelevel_predictors`. Las dimensiones de esta nueva base de datos se obtienen usando `dim(statelevel_predictors)`, proporcionando información sobre el tamaño de los datos resultantes.

``` r
statelevel_predictors <- inner_join(satelital, Contexto_mun[, paso])
dim(statelevel_predictors)
```

```
## [1] 2466   72
```


#### Guardado de la Base de Datos Final {.unnumbered}

Finalmente, la base de datos consolidada `statelevel_predictors` se guarda en un archivo RDS para su uso posterior, utilizando `saveRDS()`. Este archivo es almacenado en la ruta especificada, permitiendo que los datos procesados estén disponibles para futuros análisis sin necesidad de repetir los pasos de limpieza y combinación.

```{r. eval = FALSE}
saveRDS(statelevel_predictors,file = "../input/2020/predictores/statelevel_predictors_df.rds")


```

