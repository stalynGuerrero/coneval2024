# 19_Mapas_carencias.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **19_Mapas_carencias.R** disponible en la ruta *Rcodes/2020/19_Mapas_carencias.R*.

El código proporciona un procedimiento para la visualización de estimaciones de carencias y errores de estimación en un formato de mapa utilizando `tmap` y datos geoespaciales. Primero, limpia el entorno de trabajo en R y establece un límite para el uso de memoria RAM. Luego, carga las librerías necesarias y los datasets relevantes, incluyendo los resultados de estimación de carencia y un shapefile de México que se ajusta para incluir las claves municipales y regionales.

A continuación, define los cortes para la clasificación de los datos en porcentajes y genera mapas temáticos para cada indicador de carencia (como pobreza por ingresos, carencias sociales, etc.). Utiliza el objeto `ShapeDAM` combinado con `ipm_mpios` para crear mapas que visualizan estos indicadores con una paleta de colores verde. Cada mapa se guarda como una imagen PNG con alta resolución.

Finalmente, el código realiza un proceso similar para visualizar los errores de estimación asociados a cada indicador. Los cortes para los errores se definen de manera diferente, y se generan mapas temáticos que muestran la magnitud de estos errores. Cada mapa se guarda en formato PNG con especificaciones similares.

#### Limpieza del Entorno y Carga de Bibliotecas{-}

Este fragmento de código realiza una serie de preparativos esenciales antes de ejecutar un análisis de datos. Primero, limpia el entorno de trabajo eliminando todos los objetos existentes en la memoria y fuerza la recolección de basura para liberar espacio en el sistema, asegurando que no queden residuos de ejecuciones anteriores que puedan afectar la ejecución actual. Luego, carga una serie de bibliotecas necesarias para el análisis y la visualización de datos. Entre las bibliotecas cargadas se encuentran `dplyr` y `data.table` para la manipulación de datos, `haven` para la importación de datos, `magrittr` para el uso de operadores encadenados, `stringr` para el manejo de cadenas de texto, `openxlsx` para la manipulación de archivos Excel, y `tmap` y `sf` para la creación de mapas y el manejo de datos espaciales. Además, se redefine la función `select` para asegurarse de que se utilice la versión de `dplyr`. Finalmente, el código establece un límite de memoria RAM de 250 GB para la ejecución del script, lo cual es crucial para evitar problemas de memoria durante el análisis de grandes volúmenes de datos.


``` r
rm(list = ls())
gc()

#################
### Libraries ###
#################
library(dplyr)
library(data.table)
library(haven)
library(magrittr)
library(stringr)
library(openxlsx)
library(tmap)
library(sf)
select <- dplyr::select

###------------ Definiendo el límite de la memoria RAM a emplear ------------###

memory.limit(250000000)
```

##### Lectura de Datos{-}

Primero, carga un archivo de resultados de pobreza a nivel municipal en formato RDS, utilizando la función `readRDS`. El archivo cargado, `ipm_mpios`, contiene los resultados de los indicadores de pobreza a nivel municipal que se han generado previamente.

En segundo lugar, lee un archivo de forma (shapefile) en formato `.shp` utilizando la función `read_sf` de la biblioteca `sf`. Este archivo, ubicado en la carpeta "../shapefile/2020/", contiene datos espaciales que describen la geografía de México para el año 2020. Una vez cargado el shapefile, el código realiza dos transformaciones en el objeto `ShapeDAM`: extrae el código de entidad (`ent`) a partir de los primeros dos dígitos del campo `CVEGEO`, y elimina la columna `CVEGEO`, ya que no es necesaria para el análisis posterior.

Finalmente, define un vector `cortes` que contiene los puntos de corte para la clasificación de datos en intervalos porcentuales (0%, 15%, 30%, 50%, 80%, 100%). Estos cortes serán utilizados para categorizar o visualizar los datos de pobreza en diferentes rangos de intensidad.


``` r
ipm_mpios <- 
  readRDS("../output/Entregas/2020/result_mpios_carencia.RDS")   

ShapeDAM <- read_sf("../shapefile/2020/MEX_2020.shp")
ShapeDAM %<>% mutate(cve_mun = CVEGEO ,
                     ent = substr(cve_mun, 1, 2),
                     CVEGEO = NULL) 

cortes <- c(0,  15, 30, 50, 80, 100 )/100
```

#### Mapas de las carencias estimadas {-}

Primero, se crea un objeto `P1_mpio_norte` utilizando `tm_shape`, que carga un mapa basado en el shapefile `ShapeDAM` y une este mapa con los datos de pobreza `ipm_mpios` a través de la columna `cve_mun`. Los datos de pobreza son incorporados al mapa utilizando un `left_join`.

Se define un vector `ind` que contiene los nombres de los indicadores de pobreza que se desean visualizar en los mapas. Estos indicadores incluyen varias medidas de pobreza y carencias sociales.

Luego, se ejecuta un bucle `for` que recorre cada indicador en `ind`. Para cada indicador (`yks_ind`), se crea un mapa temático (`Mapa_I`) usando `tm_polygons`. Este mapa visualiza los datos de pobreza con una paleta de colores verde (`palette = "Greens"`) y clasifica los datos en intervalos definidos por el vector `cortes`. El título del mapa incluye el nombre del indicador actual.

La función `tm_layout` ajusta la presentación del mapa, configurando la leyenda para que sea visible, con un tamaño de texto específico, y colocada en la posición izquierda. La leyenda se presenta en una disposición vertical y con estilos de fuente en negrita.

Finalmente, el mapa se guarda en un archivo PNG utilizando `tmap_save`, con un tamaño de imagen de 4000x3000 píxeles y una relación de aspecto de 0. El archivo se nombra de acuerdo con el indicador de pobreza actual y se guarda en la carpeta "../output/Entregas/2020/mapas/".


``` r
P1_mpio_norte <-
  tm_shape(ShapeDAM %>%
             left_join(ipm_mpios,   by = "cve_mun"))
names(ipm_mpios)

ind <- c("pobrea_lp", "pobrea_li","tol_ic_1","tol_ic_2","ic_segsoc", 
         "ic_ali_nc")


for(yks_ind in ind){

  Mapa_I <-
    P1_mpio_norte + tm_polygons(
      breaks  = cortes,
      yks_ind,
      # style = "quantile",
      title =  paste0("Estimación de la carencia 2020 -", yks_ind ),
      palette = "Greens",
      colorNA = "white"
    ) + tm_layout(legend.show = TRUE,
                  #legend.outside = TRUE,
                  legend.text.size =  1.5, 
                  legend.outside.position = 'left',
                  legend.hist.width = 1,
                  legend.hist.height = 3,
                  legend.stack = 'vertical',
                  legend.title.fontface = 'bold',
                  legend.text.fontface = 'bold') 
  
  
  tmap_save(
    Mapa_I,
    filename =  paste0("../output/Entregas/2020/mapas/carencia_", yks_ind, ".png"),
    width = 4000,
    height = 3000,
    asp = 0
  )
  
    
}
```

#### Mapas del error de la estimación de las carencias {-}

Primero, se redefine el vector `cortes` para establecer los intervalos en los que se clasificarán los datos de error de estimación. Los cortes incluyen valores como 0%, 1.5%, 10%, 25%, 50% y 100%.

Luego, se genera un vector `ind_ee` que contiene los nombres de las variables de error de estimación, obtenidos al anteponer "ee_" a cada indicador de pobreza en `ind`. Por ejemplo, si `ind` incluye "pobrea_lp", entonces `ind_ee` incluirá "ee_pobrea_lp".

A continuación, se ejecuta un bucle `for` que recorre cada indicador de error de estimación en `ind_ee`. Para cada indicador (`yks_ind`), se crea un mapa temático (`Mapa_I_ee`) usando `tm_polygons`. Este mapa visualiza el error de estimación con la paleta de colores verde (`palette = "Greens"`) y clasifica los datos en intervalos definidos por el vector `cortes`. El título del mapa incluye el nombre del indicador de error de estimación actual.

La función `tm_layout` ajusta la presentación del mapa, configurando la leyenda para que sea visible, con un tamaño de texto específico y ubicada en la posición izquierda. La leyenda se presenta en una disposición vertical y con estilos de fuente en negrita.

Finalmente, el mapa se guarda en un archivo PNG usando `tmap_save`, con un tamaño de imagen de 4000x3000 píxeles y una relación de aspecto de 0. El archivo se nombra según el indicador de error de estimación actual y se guarda en la carpeta "../output/Entregas/2020/mapas/".


``` r
cortes <- c(0,  1,5,10, 25,  50,  100 )/100

ind_ee <- paste0("ee_", ind)

for(yks_ind in ind_ee){

  Mapa_I_ee <-
  P1_mpio_norte + tm_polygons(
    breaks  = cortes,
    yks_ind,
    # style = "quantile",
    title = paste0("Error de estimación de la carencia 2020 - ", yks_ind),
    palette = "Greens",
    colorNA = "white"
  ) + tm_layout(legend.show = TRUE,
                #legend.outside = TRUE,
                legend.text.size =  1.5, 
                legend.outside.position = 'left',
                legend.hist.width = 1,
                legend.hist.height = 3,
                legend.stack = 'vertical',
                legend.title.fontface = 'bold',
                legend.text.fontface = 'bold') 


tmap_save(
  Mapa_I_ee,
  filename =  paste0("../output/Entregas/2020/mapas/carencia_", yks_ind, ".png"),
  width = 4000,
  height = 3000,
  asp = 0
)
}
```

