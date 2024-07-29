# Proceso para la descarga de imágenes satelitales



En esta sección, se describe el procedimiento para la descarga de imágenes satelitales utilizando R y su integración con Google Earth Engine (GEE). La capacidad de acceder a datos satelitales es esencial para diversos análisis geoespaciales, que van desde el monitoreo ambiental hasta la planificación urbana y la investigación científica. A través de la biblioteca rgee, que actúa como un puente entre R y GEE, se pueden aprovechar las vastas colecciones de datos satelitales y herramientas analíticas de GEE directamente desde el entorno de R. Esta integración permite una manipulación y análisis eficientes de grandes volúmenes de datos geoespaciales. En el siguiente proceso, se detallarán los pasos para configurar el entorno de trabajo, establecer la conexión con GEE, y realizar la descarga de las imágenes satelitales requeridas.

## Configurar el entorno de *R*

El código configura el entorno de trabajo en R para el análisis de datos geográficos y la conexión con Google Earth Engine. Primero, limpia el espacio de trabajo y establece un límite de memoria. Luego, carga varias bibliotecas necesarias para la manipulación de datos, el muestreo, la conexión con Google Earth Engine, y el manejo de datos geográficos. 


``` r
# rm(list = ls())

memory.limit(500000)
```

```
## [1] Inf
```

``` r
library(tidyverse)
library(sampling)
library(rgee) # Conexión con Google Earth Engine
library(sf) # Paquete para manejar datos geográficos
library(concaveman)
library(geojsonio)
library(magrittr)
library(furrr)
library(readr)
```

El siguiente código configura el entorno de Python necesario para trabajar con Google Earth Engine a través de `reticulate`, asegurando que el entorno `rgee_py` esté correctamente instalado y configurado. Finalmente, inicializa la conexión con Google Earth Engine, habilitando el acceso a los recursos de Google Drive. Este proceso prepara el entorno para realizar análisis avanzados que combinan capacidades de R y Python.


``` r
rgee_environment_dir <-reticulate::conda_list()
rgee_environment_dir <- rgee_environment_dir[rgee_environment_dir$name == "rgee_py",2]
reticulate::use_python(rgee_environment_dir, required = T )
reticulate::py_config()
library(reticulate) # Conexión con Python

rgee::ee_install_set_pyenv(py_path = rgee_environment_dir, py_env = "rgee_py")
Sys.setenv(RETICULATE_PYTHON = rgee_environment_dir)
Sys.setenv(EARTHENGINE_PYTHON = rgee_environment_dir)
rgee::ee_Initialize(drive = T)
```

## Preparación de la **Shapefile** 

Se leen todos los archivos shapefile en el directorio especificado, seleccionando las columnas relevantes y utilizando procesamiento paralelo para mejorar la velocidad.


``` r
# Leer y seleccionar shapefiles
MEX <- list.files("../shapefile/2020/", pattern = "shp$", full.names = TRUE) %>%
  furrr::future_map(~read_sf(.x) %>%
                      select(CVEGEO, NOM_ENT, NOMGEO, geometry),
                    .progress = TRUE)
```


### Definición del Sistema de Referencia Espacial

Se define el sistema de referencia espacial WGS84 para la transformación de coordenadas.


``` r
mi_crs <- "+proj=longlat +datum=WGS84"
```

### Transformación y Combinación de Shapefiles
Se transforman las coordenadas de los shapefiles al sistema de referencia espacial WGS84 y se asegura que las geometrías sean de tipo `MULTIPOLYGON`. Luego, se combinan todos los shapefiles transformados en un único dataframe espacial.


``` r
# Transformar y combinar shapefiles
MEX_trans <- map(MEX,  ~st_transform(.x, crs = mi_crs))
MEX_trans <- map(MEX_trans,  ~st_cast(.x, "MULTIPOLYGON"))
MEX_trans <- bind_rows(MEX_trans)
```


### Corrección de la Codificación de los Nombres Geográficos

Se convierte la codificación de los nombres geográficos a UTF-8 para asegurar la correcta visualización de caracteres especiales.



``` r
# Corregir la codificación de los nombres geográficos
MEX_trans$NOMGEO <- iconv(MEX_trans$NOMGEO, to = "UTF-8", from = "ISO-8859-1")
```


### Guardado del Shapefile Transformado

Se escribe el shapefile transformado y unificado en un nuevo archivo en el directorio especificado.


``` r
# Guardar el shapefile transformado
st_write(MEX_trans, "shapefile/2020/MEX_2020.shp", append = TRUE)
```


### Limpieza del Espacio de Trabajo
Se limpia el espacio de trabajo eliminando todos los objetos para liberar memoria.


``` r
# Limpiar el espacio de trabajo
rm(list = ls())
```



### Lectura de Shapefile Transformado

Este script carga un shapefile transformado que contiene información geográfica de México, añade una columna con los códigos de entidad, y extrae datos de luminosidad nocturna del conjunto de datos NOAA/VIIRS para el año 2020, transformando las imágenes seleccionadas en una banda que representa la luminosidad promedio anual. Los datos extraídos se combinan en un solo dataframe, se filtran para identificar y manejar valores faltantes, y finalmente se guardan en un archivo RDS para su uso posterior.



``` r
# Leer el shapefile transformado
MEX <- read_sf("shapefile/2020/MEX_2020.shp")

# Añadir una nueva columna con la entidad (los primeros dos caracteres de CVEGEO)
MEX %<>% mutate(ent = substr(CVEGEO, 1, 2))

# Selección de datos de luminosidad nocturna de la colección de imágenes NOAA/VIIRS
# Enlace: https://developers.google.com/earth-engine/datasets/catalog/NOAA_VIIRS_DNB_ANNUAL_V21#bands

# Cargar la colección de imágenes VIIRS-DNB y filtrar por el año 2020
luces = ee$ImageCollection("NOAA/VIIRS/DNB/ANNUAL_V21") %>%
  ee$ImageCollection$filterDate("2020-01-01", "2021-01-01") %>%
  ee$ImageCollection$map(function(x) x$select("average")) %>%
  ee$ImageCollection$toBands()

# Imprimir la estructura de la colección de imágenes para ver detalles de las bandas
ee_print(luces)

# Extraer valores de luminosidad nocturna para cada entidad en MEX
MEX_luces <- map(unique(MEX$ent),
                 ~tryCatch(ee_extract(
                   x = luces,
                   y = MEX[c("ent", "CVEGEO")] %>%
                     filter(ent == .x),
                   ee$Reducer$mean(),
                   sf = FALSE
                 ), 
                 error = function(e) data.frame(ent = .x))
)

# Combinar los resultados en un solo dataframe
MEX_luces %<>% bind_rows()

# Filtrar y mostrar las filas con valores faltantes en la columna de luminosidad promedio
MEX_luces %>% filter(is.na(X20200101_average)) %>% select(CVEGEO)

# Guardar los datos de luminosidad en un archivo RDS
saveRDS(MEX_luces, "output/2020/Satelital/MEX_luces.rds")
```

### Cobertura urbana y uso del suelo

El código realiza el procesamiento de datos relacionados con el uso del suelo y la cobertura urbana y de cultivos en México para el año 2019. Primero, carga la colección de imágenes de la serie Proba-V Landcover, que proporciona datos sobre fracciones de cobertura urbana y de cultivos. Luego, se filtra esta colección para el año 2019 y se seleccionan las bandas relevantes para analizar estas fracciones. Posteriormente, el código extrae estas fracciones para cada entidad geográfica en México, calculando la media de las bandas seleccionadas y manejando errores que puedan surgir durante el proceso. Los resultados se combinan en un solo dataframe y se revisan los datos para identificar cualquier valor faltante en las fracciones de cultivos y cobertura urbana. Finalmente, el dataframe resultante se guarda en un archivo RDS para su uso futuro.


``` r
tiposuelo = ee$ImageCollection("COPERNICUS/Landcover/100m/Proba-V-C3/Global") %>%
  ee$ImageCollection$filterDate("2019-01-01", "2019-12-31") %>%
  ee$ImageCollection$map(function(x) x$select("urban-coverfraction", "crops-coverfraction")) %>% 
  ee$ImageCollection$toBands()
ee_print(tiposuelo)

MEX_urbano_cultivo <- map(unique(MEX$CVEGEO),
                          ~tryCatch(ee_extract(
                            x = tiposuelo,
                            y = MEX[c("ent","CVEGEO")] %>%
                              filter(CVEGEO == .x),
                            ee$Reducer$mean(),
                            sf = FALSE
                          ) , 
                          error = function(e)data.frame(CVEGEO = .x)) 
)


MEX_urbano_cultivo %<>% bind_rows()

MEX_urbano_cultivo %>% filter(is.na(X2019_crops.coverfraction)) %>% select(CVEGEO) 
MEX_urbano_cultivo %>% filter(is.na(X2019_urban.coverfraction)) %>% select(CVEGEO) 

saveRDS(MEX_urbano_cultivo, "output/2020/Satelital/MEX_urbano_cultivo.rds")
```

### Distancia a hospitales 

El código realiza el análisis de la distancia a los hospitales utilizando datos satelitales para el año 2019. Primero, se carga la imagen que representa la accesibilidad a la atención médica, proporcionada por el dataset de Oxford. Luego, se extraen datos de accesibilidad para cada entidad geográfica en México, calculando la media de los valores de accesibilidad para cada área. Se maneja cualquier error potencial durante este proceso, garantizando que, en caso de problemas, se registre la entidad correspondiente sin datos. Posteriormente, los resultados se combinan en un solo dataframe y se revisa si hay valores faltantes en las medidas de accesibilidad. Finalmente, el dataframe con los datos de accesibilidad se guarda en un archivo RDS para su análisis posterior.


``` r
dist_salud = ee$Image('Oxford/MAP/accessibility_to_healthcare_2019') 
ee_print(dist_salud)


MEX_dist_salud <- map(unique(MEX$CVEGEO),
                          ~tryCatch(ee_extract(
                            x = dist_salud,
                            y = MEX[c("ent","CVEGEO")] %>%
                              filter(CVEGEO == .x),
                            ee$Reducer$mean(),
                            sf = FALSE
                          ) , 
                          error = function(e)data.frame(CVEGEO = .x)) 
)


MEX_dist_salud %<>% bind_rows()

MEX_dist_salud %>% filter(is.na(accessibility )) %>% select(CVEGEO) 
MEX_dist_salud %>% filter(is.na(accessibility_walking_only)) %>% select(CVEGEO) 

saveRDS(MEX_dist_salud, "output/2020/Satelital/MEX_dist_salud.rds")
```


### Global Human Modification (GHM)

El código realiza el análisis de la modificación humana global (gHM) utilizando datos de satélites para el año 2016. Primero, se carga la colección de imágenes correspondiente al índice de Modificación Humana Global, que proporciona información sobre la alteración del entorno por actividades humanas. Luego, se extraen estos datos para cada entidad geográfica en México, calculando la media de los valores para cada área. Se maneja cualquier error que pueda surgir durante la extracción de datos, asegurando que, en caso de problemas, se registre la entidad sin datos. Después de combinar los resultados en un solo dataframe, se revisa si hay valores faltantes en el índice de modificación humana. Finalmente, se guarda el dataframe con los datos de modificación humana en un archivo RDS para su análisis futuro.


``` r
CSP_gHM = ee$ImageCollection('CSP/HM/GlobalHumanModification') 
ee_print(CSP_gHM)

MEX_GHM <- map(unique(MEX$CVEGEO),
                      ~tryCatch(ee_extract(
                        x = CSP_gHM,
                        y = MEX[c("ent","CVEGEO")] %>%
                          filter(CVEGEO == .x),
                        ee$Reducer$mean(),
                        sf = FALSE
                      ) , 
                      error = function(e)data.frame(CVEGEO = .x)) 
)


MEX_GHM %<>% bind_rows()

MEX_GHM %>% filter(is.na(X2016_gHM )) %>% select(CVEGEO) 

saveRDS(MEX_GHM, "output/2020/Satelital/MEX_GHM.rds")
```

## Conslidar y  guardar las variables satelitales    

El código se encarga de consolidar y guardar los datos de predictores a nivel estatal obtenidos a partir de diferentes archivos satelitales. Primero, lee todos los archivos `.rds` generados previamente desde un directorio específico y los combina en un solo dataframe mediante una unión completa (`full_join`). Luego, normaliza las variables numéricas escalándolas para asegurar que estén en una escala comparable. A continuación, renombra las columnas del dataframe para hacerlas más descriptivas y comprensibles, como "modifica_humana" para el índice de modificación humana y "luces_nocturnas" para la luminosidad nocturna. Se verifica si hay valores faltantes en el dataframe y se visualizan las filas con datos faltantes. Finalmente, guarda el dataframe consolidado y limpio en un archivo `.rds` para su uso posterior.


``` r
statelevel_predictors_df <-
    list.files("../output/2020/Satelital/", full.names = TRUE) %>%
  map( ~ readRDS(file = .x)) %>%
  reduce(., full_join) %>%
  mutate_if(is.numeric, function(x)
    as.numeric(scale(x))) %>%
  rename(
    cve_mun = CVEGEO,
    "modifica_humana" = X2016_gHM,
    "acceso_hosp" = accessibility,
    "acceso_hosp_caminando" = accessibility_walking_only,
    cubrimiento_urbano = X2019_urban.coverfraction,
    cubrimiento_cultivo = X2019_crops.coverfraction,
    luces_nocturnas = X20200101_average
  )

statelevel_predictors_df[apply(statelevel_predictors_df,1,function(x)any(is.na(x))),] %>% view()
```

<table class="table table-striped lightable-classic" style="width: auto !important; margin-left: auto; margin-right: auto; font-family: Arial Narrow; width: auto !important; margin-left: auto; margin-right: auto;">
<caption>(\#tab:unnamed-chunk-14)Variables satelitales descargadas</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> ent </th>
   <th style="text-align:left;"> cve_mun </th>
   <th style="text-align:right;"> acceso_hosp </th>
   <th style="text-align:right;"> acceso_hosp_caminando </th>
   <th style="text-align:right;"> modifica_humana </th>
   <th style="text-align:right;"> luces_nocturnas </th>
   <th style="text-align:right;"> cubrimiento_cultivo </th>
   <th style="text-align:right;"> cubrimiento_urbano </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01001 </td>
   <td style="text-align:right;"> -0.6813 </td>
   <td style="text-align:right;"> -0.5599 </td>
   <td style="text-align:right;"> 0.4796 </td>
   <td style="text-align:right;"> 0.9925 </td>
   <td style="text-align:right;"> 0.7079 </td>
   <td style="text-align:right;"> 0.6553 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01002 </td>
   <td style="text-align:right;"> -0.5563 </td>
   <td style="text-align:right;"> -0.3965 </td>
   <td style="text-align:right;"> 0.1182 </td>
   <td style="text-align:right;"> -0.1495 </td>
   <td style="text-align:right;"> 0.4772 </td>
   <td style="text-align:right;"> -0.1273 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01003 </td>
   <td style="text-align:right;"> -0.2991 </td>
   <td style="text-align:right;"> -0.3885 </td>
   <td style="text-align:right;"> -0.4109 </td>
   <td style="text-align:right;"> -0.2306 </td>
   <td style="text-align:right;"> -0.2745 </td>
   <td style="text-align:right;"> -0.2943 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01004 </td>
   <td style="text-align:right;"> -0.6035 </td>
   <td style="text-align:right;"> -0.3140 </td>
   <td style="text-align:right;"> 0.4977 </td>
   <td style="text-align:right;"> -0.1221 </td>
   <td style="text-align:right;"> 0.7296 </td>
   <td style="text-align:right;"> -0.0922 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01005 </td>
   <td style="text-align:right;"> -0.5637 </td>
   <td style="text-align:right;"> -0.4701 </td>
   <td style="text-align:right;"> 0.2252 </td>
   <td style="text-align:right;"> 0.1767 </td>
   <td style="text-align:right;"> 0.3310 </td>
   <td style="text-align:right;"> 0.1532 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01006 </td>
   <td style="text-align:right;"> -0.8050 </td>
   <td style="text-align:right;"> -0.7857 </td>
   <td style="text-align:right;"> 0.7677 </td>
   <td style="text-align:right;"> 0.1072 </td>
   <td style="text-align:right;"> 0.9624 </td>
   <td style="text-align:right;"> -0.0263 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01007 </td>
   <td style="text-align:right;"> -0.5964 </td>
   <td style="text-align:right;"> -0.6407 </td>
   <td style="text-align:right;"> 0.1519 </td>
   <td style="text-align:right;"> -0.0469 </td>
   <td style="text-align:right;"> 0.8441 </td>
   <td style="text-align:right;"> -0.1150 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01008 </td>
   <td style="text-align:right;"> 0.4619 </td>
   <td style="text-align:right;"> 0.2202 </td>
   <td style="text-align:right;"> -1.0825 </td>
   <td style="text-align:right;"> -0.2670 </td>
   <td style="text-align:right;"> -0.4528 </td>
   <td style="text-align:right;"> -0.3755 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01009 </td>
   <td style="text-align:right;"> -0.6636 </td>
   <td style="text-align:right;"> -0.6723 </td>
   <td style="text-align:right;"> 0.3384 </td>
   <td style="text-align:right;"> -0.0860 </td>
   <td style="text-align:right;"> 0.1691 </td>
   <td style="text-align:right;"> -0.0672 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 01 </td>
   <td style="text-align:left;"> 01010 </td>
   <td style="text-align:right;"> -0.5932 </td>
   <td style="text-align:right;"> -0.5854 </td>
   <td style="text-align:right;"> -0.2819 </td>
   <td style="text-align:right;"> -0.1655 </td>
   <td style="text-align:right;"> 1.1342 </td>
   <td style="text-align:right;"> -0.2584 </td>
  </tr>
</tbody>
</table>


``` r
saveRDS(statelevel_predictors_df, "input/2020/predictores/statelevel_predictors_satelite.rds")
```

