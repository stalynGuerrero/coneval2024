# 01_Descarga_satelitales.R {-}

Para la ejecución del presente archivo, debe abrir el archivo **01_Descarga_satelitales.R** disponible en la ruta *Rcodes/2020/01_Descarga_satelitales.R*.

Este código en R se enfoca en el procesamiento y análisis de datos geoespaciales utilizando una combinación de bibliotecas y herramientas de Google Earth Engine. La primera parte del script configura el entorno de trabajo, carga las librerías necesarias, y establece la conexión con Python mediante la biblioteca `reticulate`, esencial para trabajar con `rgee`, que es la interfaz de R para Google Earth Engine.

En la segunda parte, se realiza la limpieza y transformación de datos espaciales. Se cargan y transforman archivos shapefiles de México a un sistema de referencia de coordenadas adecuado y se convierten a un formato `MULTIPOLYGON`. Luego, se extraen y procesan datos de imágenes satelitales para diversas variables como luminosidad, urbanización, y distancia a servicios de salud utilizando colecciones de imágenes de Google Earth Engine. Estos datos se agrupan por región y se guardan en archivos RDS para su análisis posterior. Finalmente, se combinan todos los datos procesados en un único dataframe y se escalan para prepararlos para su uso en modelos o análisis adicionales.

#### Lectura de Librerías {-}

Para comenzar, se realiza una limpieza del entorno de trabajo en R mediante la eliminación de todos los objetos existentes usando `rm(list = ls())`. Esto asegura que no haya interferencias de sesiones anteriores y que el entorno esté limpio. Luego, se establece un límite de memoria de 500,000 MB con `memory.limit(500000)`, lo que permite manejar grandes volúmenes de datos sin que R se quede sin memoria, preparando el entorno para análisis intensivos.

A continuación, se cargan varias librerías esenciales para el análisis de datos. `tidyverse` se utiliza para la manipulación y visualización de datos de manera eficiente, mientras que `sampling` proporciona herramientas para realizar muestreo estadístico. `rgee` permite la conexión con Google Earth Engine para realizar análisis geoespaciales avanzados. `sf` es fundamental para manejar datos geográficos, y `concaveman` ayuda a generar polígonos cóncavos a partir de conjuntos de puntos. Además, `geojsonio` facilita la conversión entre diversos formatos de datos geoespaciales, `magrittr` mejora la legibilidad del código con el operador `%>%`, `furrr` habilita la programación paralela utilizando `purrr`, y `readr` se emplea para la lectura eficiente de archivos de datos en formatos como CSV.


``` r
rm(list = ls())

memory.limit(500000)

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

#### configuración inicial de Python {-}

Para comenzar, se obtiene la lista de entornos conda disponibles y se selecciona el entorno llamado "rgee_py", que es específico para la conexión con Google Earth Engine mediante `rgee`. Esto asegura que se utilice el entorno de Python correcto para las operaciones geoespaciales.

Luego, se configura `reticulate` para usar el entorno de Python especificado. La función `py_config()` verifica la configuración de Python y se carga la librería `reticulate` para habilitar la conexión entre R y Python, lo que permite ejecutar código Python directamente desde R.

Finalmente, se establece el entorno de Python para `rgee` mediante `ee_install_set_pyenv()`, y se configuran las variables de entorno para asegurar que `reticulate` y `rgee` usen el entorno de Python correcto. La función `ee_Initialize(drive = T)` inicializa la conexión con Google Earth Engine, permitiendo el acceso y manipulación de datos geoespaciales desde R.


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

#### Arreglar la shapefile {-}

Este bloque de código está diseñado para procesar y transformar archivos shapefile (archivos de formato geoespacial) de México para asegurarse de que estén en el formato correcto y preparados para su análisis. Incluye la lectura de archivos, la transformación del sistema de referencia de coordenadas (CRS), la combinación de los archivos shapefile y la corrección de la codificación de caracteres.

Primero, se descomenta la configuración para ejecutar en paralelo con dos trabajadores usando `plan(multisession, workers = 2)`, lo que mejora la eficiencia del procesamiento. Luego, se listan los archivos shapefile en el directorio "shapefile/2020/" y se leen en una lista `MEX` utilizando `furrr::future_map()`, que permite el procesamiento en paralelo. Cada archivo se transforma para seleccionar columnas específicas (CVEGEO, NOM_ENT, NOMGEO, y geometry). La función `st_transform()` se utiliza para cambiar el sistema de referencia de coordenadas a WGS84, y `st_cast()` convierte los objetos geométricos a "MULTIPOLYGON".

Después de la transformación y combinación, se unen todos los shapefiles en un solo objeto usando `bind_rows()`. Para asegurar que los nombres geográficos estén en la codificación correcta, se usa `iconv()` para convertir `NOMGEO` a UTF-8 desde ISO-8859-1. El archivo shapefile transformado y combinado se guarda con `st_write()` en la ubicación "shapefile/2020/MEX_2020.shp".

Finalmente, se limpia el entorno con `rm(list = ls())`, y el archivo shapefile resultante se lee nuevamente con `read_sf()`. Se agrega una nueva columna `ent` mediante `mutate()` para extraer los dos primeros caracteres de `CVEGEO`, que representan la entidad federativa.



``` r
# plan(multisession, workers = 2)
# 
# MEX <- list.files("shapefile/2020/", pattern = "shp$",
#            full.names = TRUE) %>%
#   furrr::future_map(~read_sf(.x) %>%
#                           select(CVEGEO, NOM_ENT, NOMGEO,geometry),
#                     .progress = TRUE)
# 
# mi_crs <- "+proj=longlat +datum=WGS84"
# 
# # Transforma el polígono al nuevo CRS
# MEX_trans <- map(MEX,  ~st_transform(.x, crs = mi_crs))
# MEX_trans <- map(MEX_trans,  ~st_cast(.x, "MULTIPOLYGON"))
# 
# MEX_trans <- bind_rows(MEX_trans)
# 
# 
# # stringi::stri_enc_detect(MEX_trans$NOMGEO)
# 
# MEX_trans$NOMGEO <- iconv(MEX_trans$NOMGEO,
#                           to = "UTF-8", from = "ISO-8859-1")
# 
# 
# st_write(MEX_trans,
#          "shapefile/2020/MEX_2020.shp",
#          append = TRUE)
# 
# rm(list = ls())

MEX <- read_sf("../shapefile/2020/MEX_2020.shp")

MEX %<>% mutate(ent = substr(CVEGEO,1,2))
```

#### Descargar las imegenes satelitales  {-}

Este bloque de código se centra en la descarga y procesamiento de imágenes satelitales utilizando Google Earth Engine (GEE) en R. Se recopilan datos de diferentes fuentes satelitales para diversas características geoespaciales de México, incluyendo luces nocturnas, cobertura urbana y agrícola, accesibilidad a servicios de salud y modificación humana global. Cada conjunto de datos se extrae, procesa y guarda para su uso posterior en análisis.

##### Luces Nocturnas {-}

Se utiliza la colección de imágenes "NOAA/VIIRS/DNB/ANNUAL_V21" para obtener datos de luces nocturnas del año 2020. Las imágenes se filtran por fecha y se selecciona la banda "average". Posteriormente, se extraen los datos de luces nocturnas promedio para cada entidad federativa en México, y se combinan los resultados en un solo dataframe. Finalmente, se guardan los datos en un archivo `.rds`.

``` r
#https://developers.google.com/earth-engine/datasets/catalog/NOAA_VIIRS_DNB_ANNUAL_V21#bands

luces = ee$ImageCollection("NOAA/VIIRS/DNB/ANNUAL_V21") %>%
  ee$ImageCollection$filterDate("2020-01-01", "2021-01-01") %>%
  ee$ImageCollection$map(function(x) x$select("average")) %>%
  ee$ImageCollection$toBands()
ee_print(luces)

MEX_luces <- map(unique(MEX$ent),
                 ~tryCatch(ee_extract(
                   x = luces,
                   y = MEX[c("ent","CVEGEO")] %>%
                     filter(ent == .x),
                   ee$Reducer$mean(),
                   sf = FALSE
                 ) , 
                 error = function(e)data.frame(ent = .x)) 
)


MEX_luces %<>% bind_rows()

MEX_luces %>% filter(is.na(X20200101_average)) %>% select(CVEGEO) 


saveRDS(MEX_luces, "../output/2020/Satelital/MEX_luces.rds")
```


##### Urbanismo y Cultivos {-}

Se extraen datos de cobertura urbana y de cultivos de la colección "COPERNICUS/Landcover/100m/Proba-V-C3/Global" para el año 2019. Se seleccionan las bandas "urban-coverfraction" y "crops-coverfraction", y se calculan los promedios de estas coberturas para cada entidad federativa. Los resultados se combinan y se guardan en un archivo `.rds`.



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

saveRDS(MEX_urbano_cultivo, "../output/2020/Satelital/MEX_urbano_cultivo.rds")
```

##### Distancia a Hospitales {-}

Se utilizan datos de la imagen "Oxford/MAP/accessibility_to_healthcare_2019" para calcular la accesibilidad a servicios de salud. Se extraen los datos de accesibilidad para cada entidad federativa y se combinan en un solo dataframe. Los resultados se guardan en un archivo `.rds`.



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

saveRDS(MEX_dist_salud, "../output/2020/Satelital/MEX_dist_salud.rds")
```

##### Modificación Humana Global{-}

Se recopilan datos de la colección "CSP/HM/GlobalHumanModification" para medir la modificación humana global en el año 2016. Se extraen los datos para cada entidad federativa y se calculan los promedios, combinándolos en un dataframe. Finalmente, se guardan los resultados en un archivo `.rds`.


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

saveRDS(MEX_GHM, "../output/2020/Satelital/MEX_GHM.rds")
```

#### Unir y guardar los resultados de las descargas realizadas {-}

En este bloque de código, se realiza la integración y almacenamiento de los datos satelitales descargados en archivos `.rds`. El objetivo es combinar los datos de diferentes tipos de imágenes satelitales en un único dataframe para facilitar el análisis posterior.

Primero, se generan una lista de archivos `.rds` ubicados en el directorio `"../output/2020/Satelital/"`, que contienen los datos procesados de las imágenes satelitales. Utilizando `map()` y `readRDS()`, se leen todos los archivos y se combinan en un solo dataframe mediante la función `reduce()` con `full_join`, que une los datos basándose en todas las claves presentes.

Luego, se estandarizan las columnas numéricas en el dataframe usando `scale()` para normalizar los valores y se renombran las columnas para asegurar que los nombres sean descriptivos y consistentes. Los nuevos nombres de columna incluyen información sobre modificación humana, acceso a servicios de salud, cobertura urbana y de cultivos, y luces nocturnas.

Se identifican y visualizan las filas que contienen valores faltantes (`NA`) en cualquier columna para realizar una revisión o limpieza adicional. Finalmente, se guarda el dataframe combinado en un archivo `.rds` con el nombre `"statelevel_predictors_satelite.rds"` en el directorio `"../input/2020/predictores/"`, lo que facilita el acceso y análisis de los datos integrados en futuras etapas del proyecto.


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

saveRDS(statelevel_predictors_df, "../input/2020/predictores/statelevel_predictors_satelite.rds")
```

