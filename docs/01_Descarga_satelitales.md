# 01_Descarga_satelitales.R

Para la ejecución del presente archivo, debe abrir el archivo **01_Descarga_satelitales.R** disponible en la ruta *Rcodes/2020/01_Descarga_satelitales.R*.

Este código en R se enfoca en el procesamiento y análisis de datos geoespaciales utilizando una combinación de bibliotecas y herramientas de Google Earth Engine. La primera parte del script configura el entorno de trabajo, carga las librerías necesarias, y establece la conexión con Python mediante la biblioteca `reticulate`, esencial para trabajar con `rgee`, que es la interfaz de R para Google Earth Engine.

En la segunda parte, se realiza la limpieza y transformación de datos espaciales. Se cargan y transforman archivos shapefiles de México a un sistema de referencia de coordenadas adecuado y se convierten a un formato `MULTIPOLYGON`. Luego, se extraen y procesan datos de imágenes satelitales para diversas variables como luminosidad, urbanización, y distancia a servicios de salud utilizando colecciones de imágenes de Google Earth Engine. Estos datos se agrupan por región y se guardan en archivos RDS para su análisis posterior. Finalmente, se combinan todos los datos procesados en un único dataframe y se escalan para prepararlos para su uso en modelos o análisis adicionales.

#### Lectura de libreías {-}


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

#### Arreglar la shape {-}


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

#################
### Urbanismo ###
#################

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

#################
### Distancia a hospitales ###
#################

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


#################
# CSP gHM: Global Human Modification
#################

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

saveRDS(statelevel_predictors_df, "input/2020/predictores/statelevel_predictors_satelite.rds")
```

