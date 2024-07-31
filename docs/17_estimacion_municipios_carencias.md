# 17_estimacion_municipios_carencias.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **17_estimacion_municipios_carencias.R** disponible en la ruta *Rcodes/2020/17_estimacion_municipios_carencias.R*.

El código proporcionado realiza una serie de procesos para la estimación y calibración de indicadores de carencias a nivel municipal en base a datos de encuestas. Comienza limpiando el entorno de trabajo y cargando las librerías necesarias, como `tidyverse`, `data.table`, y `survey`, además de leer los datos de predicciones y encuestas. Se preparan las bases de datos con variables calculadas para las carencias, incluyendo indicadores de carencias sociales y líneas de pobreza.

Posteriormente, el script filtra y prepara los datos de la encuesta para cada entidad (municipio) específica. Para cada municipio, se realiza un proceso iterativo de simulación donde se actualizan las variables del modelo (como ingreso y carencias sociales) con base en predicciones previas. Se calcula la población total y se ajusta la muestra para realizar estimaciones calibradas utilizando el método de "raking", que ajusta los pesos de la muestra para que coincidan con las estimaciones de la población.

Finalmente, se guarda cada iteración de los resultados calibrados en archivos RDS y se calcula el tiempo total de ejecución. Cada archivo contiene estimaciones para los indicadores de pobreza en cada municipio. El script utiliza técnicas de muestreo y calibración para asegurar que las estimaciones sean consistentes con las características de la población y se gestionan errores potenciales durante el proceso de calibración.


#### Configuración y carga de librerías{-}

El código comienza limpiando el entorno de trabajo en R para eliminar cualquier objeto residual que pudiera interferir con el análisis, utilizando la función `rm(list = ls())`. A continuación, carga una serie de librerías esenciales: `tidyverse` para manipulación de datos y visualización, `data.table` para el manejo eficiente de grandes conjuntos de datos, `openxlsx` para trabajar con archivos Excel, `magrittr` para el uso del operador `%>%` en la encadenación de operaciones, `haven` para leer y escribir datos en formatos de software estadístico, `labelled` para manejar etiquetas de variables, `sampling` para técnicas de muestreo, `lme4` para modelos lineales y no lineales con efectos mixtos, `survey` para el análisis de datos de encuestas, y `srvyr` para la integración de datos de encuestas con `dplyr`. Finalmente, el script `benchmarking_indicador.R` se carga usando la función `source()` para incluir funciones adicionales o configuraciones necesarias para el análisis.


``` r
### Cleaning R environment ###
rm(list = ls())
library(tidyverse)
library(data.table)
library(openxlsx)
library(magrittr)
library(haven)
library(labelled)
library(sampling)
library(lme4)
library(survey)
library(srvyr)
source("../source/benchmarking_indicador.R")
```

#### Lectura de datos{-}

El código comienza con la lectura de varios archivos de datos necesarios para el análisis. Primero, se cargan las predicciones guardadas en el archivo `predicciones.rds` ubicado en la carpeta de modelos de 2020. Luego, se lee el archivo `encuesta_ampliada.rds`, que contiene datos ampliados de encuestas, y el archivo `encuesta_sta.rds`, que se transforma para incluir la variable `ingreso`, calculada a partir de `ictpc`. Finalmente, se importa el archivo `Lineas_Bienestar.csv` con delimitadores específicos y se convierte la columna `area` en formato de texto. Estos pasos preparan los datos necesarios para realizar el análisis posterior.


``` r
predicciones <- readRDS("../output/2020/modelos/predicciones.rds")
encuesta_ampliada <- readRDS("../output/2020/encuesta_ampliada.rds")
encuesta_enigh <- readRDS("../input/2020/enigh/encuesta_sta.rds") %>%
  mutate(ingreso = ictpc)

LB <- read.delim(
  "../input/2020/Lineas_Bienestar.csv",
  header = TRUE,
  sep = ";",
  dec = ","
) %>% mutate(area = as.character(area))
```

####  Índice de Pobreza Multidimensional en la ENIGH  {-}

En esta sección del código se calcula el Índice de Pobreza Multidimensional (IPM) utilizando la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH). 

Primero, se realiza una combinación de la base de datos `encuesta_enigh` con el archivo `LB` que contiene las líneas de bienestar. Luego, se crean nuevas variables para evaluar la pobreza multidimensional:

1. **`tol_ic`**: Suma de los indicadores de bienestar social y económico (`ic_segsoc`, `ic_ali_nc`, `ic_asalud`, `ic_cv`, `ic_sbv`, `ic_rezedu`).
2. **`pobrea_lp`**: Variable binaria que indica si el ingreso es menor que el umbral de pobreza (lp), asignando un valor de 1 si es el caso, y 0 en caso contrario.
3. **`pobrea_li`**: Variable binaria que indica si el ingreso es menor que el umbral de pobreza extrema (li), asignando un valor de 1 si es el caso, y 0 en caso contrario.
4. **`tol_ic_1`**: Variable binaria que indica si la suma de los indicadores de bienestar (`tol_ic`) es mayor que 0, asignando un valor de 1 si es el caso, y 0 en caso contrario.
5. **`tol_ic_2`**: Variable binaria que indica si la suma de los indicadores de bienestar (`tol_ic`) es mayor que 2, asignando un valor de 1 si es el caso, y 0 en caso contrario.

Estas transformaciones permiten evaluar la pobreza multidimensional en función de los umbrales establecidos y los indicadores de bienestar.


``` r
################################################################################
# IPM en la enigh
################################################################################
encuesta_enigh <- encuesta_enigh %>%
  inner_join(LB) %>%
  mutate(
    tol_ic = ic_segsoc + ic_ali_nc + ic_asalud + ic_cv + ic_sbv + ic_rezedu,
    pobrea_lp = ifelse(ingreso < lp, 1, 0),
    pobrea_li = ifelse(ingreso < li, 1, 0),
    tol_ic_1 = ifelse(tol_ic > 0, 1, 0),
    tol_ic_2 = ifelse(tol_ic > 2, 1, 0)
  )
```

#### Preparación de encuesta ampliada {-}

En esta sección, se lleva a cabo la preparación de la base de datos `encuesta_ampliada` para el análisis de pobreza multidimensional. En primer lugar, la base `encuesta_ampliada` se une con el archivo `LB`, que contiene las líneas de bienestar. A continuación, se realizan varias transformaciones importantes en las variables de la encuesta. Se transforman los indicadores relacionados con salud (`ic_asalud`), cobertura de vivienda (`ic_cv`), servicios básicos (`ic_sbv`), y educación (`ic_rezedu`) en variables binarias, donde un valor de 1 se mantiene como 1 y un valor distinto se convierte en 0. Además, se calcula `tol_ic4`, que representa la suma de estos indicadores de bienestar.

Se incorporan también las predicciones generadas por modelos previos, incluyendo las predicciones de seguridad social (`pred_segsoc`), ingresos (`pred_ingreso`), y el indicador de alimentos no consumidos (`pred_ic_ali_nc`), junto con el error estándar residual (`desv_estandar_residual`). Aunque se ha planificado crear una estructura de directorios para almacenar los resultados de las iteraciones de calibración de carencia por municipio, esta parte del código está comentada y no se ejecuta en este fragmento.

Finalmente, se realiza una limpieza de memoria al eliminar el objeto `predicciones`, y se configuran variables adicionales. Se define un código de entidad específico (`ii_ent`) y se prepara una lista (`list_yks`) que incluye las variables clave para el análisis, tales como indicadores de pobreza y bienestar. Esta preparación es esencial para facilitar el análisis posterior y la organización de los resultados en directorios específicos.


``` r
encuesta_ampliada <- encuesta_ampliada %>%
  inner_join(LB) %>%
  mutate(
    ic_asalud = ifelse(ic_asalud == 1, 1, 0),
    ic_cv = ifelse(ic_cv == 1, 1, 0),
    ic_sbv = ifelse(ic_sbv == 1, 1, 0),
    ic_rezedu = ifelse(ic_rezedu == 1, 1, 0),
    tol_ic4 = ic_asalud + ic_cv + ic_sbv + ic_rezedu,
    pred_segsoc = predicciones$pred_segsoc,
    pred_ingreso = predicciones$pred_ingreso,
    desv_estandar_residual = predicciones$desv_estandar_residual,
    pred_ic_ali_nc = predicciones$pred_ic_ali_nc
  )

# dir.create("output/2020/iteraciones/mpio_calib_carencia")
# map(
#   paste0(
#     "output/2020/iteraciones/mpio_calib_carencia/",
#     unique(encuesta_enigh$ent)
#   ),
#   ~ dir.create(path = .x)
# )

rm(predicciones)

ii_ent <- "20"

list_yks <- list(
  c("pobrea_lp"),
  c("pobrea_li"),
  c("tol_ic_1", "tol_ic_2", "ic_segsoc", "ic_ali_nc")
)
```


#### Iteraciones y calibración {-}

En esta sección del código, se realiza un proceso iterativo para la calibración de estimaciones de pobreza multidimensional a nivel municipal. Este proceso se ejecuta para una lista específica de entidades, representadas por sus códigos (`ii_ent`).

### Descripción General

1. **Inicio del Ciclo de Iteración:**
   - Se comienza un ciclo `for` que itera sobre una lista de códigos de entidades (`ii_ent`), que representan diferentes áreas geográficas para las cuales se calibrarán los modelos.

2. **Configuración y Preparación de Datos:**
   - Para cada entidad (`ii_ent`), se registra el tiempo de inicio y se filtran los datos correspondientes a la entidad actual en `encuesta_ampliada` y `encuesta_enigh`.
   - Se agrupan los datos por municipio y se calcula la población total de cada municipio (`total_mpio`) y la población total a nivel estatal (`tot_pob`).
   - Se crean los vectores `top_caren` y `tx_mun`, que se combinan en `Tx_hat` para la calibración de los modelos.

3. **Iteraciones de Calibración:**
   - Se inicia un ciclo `for` para realizar 200 iteraciones de calibración.
   - En cada iteración:
     - Se simulan nuevas observaciones para las variables `ic_segsoc`, `ic_ali_nc`, y `ingreso` utilizando distribuciones aleatorias basadas en las predicciones previas.
     - Se actualizan las variables de pobreza y bienestar (`pobrea_lp`, `pobrea_li`, `tol_ic_1`, `tol_ic_2`) en función de los nuevos valores generados.
   
4. **Calibración del Modelo:**
   - Para cada conjunto de variables de interés (`list_yks`), se preparan los datos para la calibración:
     - Se generan variables dummy para los códigos de municipios y se construye un diseño de encuesta usando `as_survey_design`.
     - Se define una fórmula para la calibración y se intenta ajustar el modelo con el método de `raking`.
     - Si ocurre un error durante la calibración, se captura el error y se continúa con la siguiente iteración.

5. **Resumen y Guardado de Resultados:**
   - Los resultados calibrados se agrupan por municipio y se guardan en un archivo `.rds` en un directorio específico para cada entidad y cada iteración.
   - Se limpia la memoria (`gc()`) para liberar espacio.
   
6. **Registro de Tiempo:**
   - Al final de cada iteración de entidad, se calcula y se imprime el tiempo total transcurrido.


Los resultados de cada iteración se guardan en archivos que permiten la evaluación de la calibración en diferentes áreas geográficas y en diferentes iteraciones, lo que facilita el análisis de la variabilidad y la precisión de las estimaciones de pobreza multidimensional. 



``` r
for(ii_ent in c("03", "06", "23", "04", "01", "22", "27", "25", "18",
                 "05", "17", "28", "10", "26", "09", "32", "08", "19", "29",
                 "24", "11", "31", "13", "16", "12", "14", "15", "07", "21",
                 "30", "20", "02")) {

  cat("####################################################################\n")
  inicio <- Sys.time()
  print(inicio)

  yks <- unlist(list_yks)
  muestra_post <- encuesta_ampliada %>% filter(ent == ii_ent)
  
  total_mpio <- muestra_post %>% group_by(cve_mun) %>%
    summarise(den_mpio = sum(factor), .groups = "drop") %>%
    mutate(tot_ent = sum(den_mpio))
  
  encuesta_sta <- encuesta_enigh %>% filter(ent == ii_ent) %>% na.omit()
  
  tot_pob <- encuesta_sta %>%
    summarise_at(.vars = yks, .funs = list(
      ~ weighted.mean(., w = fep, na.rm = TRUE) * sum(total_mpio$den_mpio)
    ))

  top_caren <- unlist(as.vector(tot_pob))
  tx_mun <- setNames(total_mpio$den_mpio, paste0("cve_mun_", total_mpio$cve_mun))
  Tx_hat <- c(top_caren, tx_mun)

  for(iter in 1:200) {
    cat("\n municipio = ", ii_ent, "\n\n")
    cat("\n iteracion = ", iter, "\n\n")
    
    ############################################
    muestra_post <- muestra_post %>%
      mutate(
        ic_segsoc = rbinom(n = n(), size = 1, prob = pred_segsoc),
        ic_ali_nc = rbinom(n = n(), size = 1, prob = pred_ic_ali_nc),
        ingreso = pred_ingreso + rnorm(n = n(), mean = 0, sd = desv_estandar_residual),
        tol_ic = tol_ic4 + ic_segsoc + ic_ali_nc
      ) %>%
      mutate(
        pobrea_lp = ifelse(ingreso < lp, 1, 0),
        pobrea_li = ifelse(ingreso < li, 1, 0),
        tol_ic_1 = ifelse(tol_ic > 0, 1, 0),
        tol_ic_2 = ifelse(tol_ic > 2, 1, 0)
      )

    estima_calib_ipm <- list()
    for(ii in 1:length(list_yks)) {
      yks <- list_yks[[ii]]
      
      var_calib <- c(yks, names(tx_mun))
      Xk <- muestra_post %>% select("cve_mun") %>%
        fastDummies::dummy_columns(select_columns = c("cve_mun"), remove_selected_columns = TRUE)
      
      diseno_post <- bind_cols(muestra_post, Xk) %>%
        mutate(fep = factor) %>%
        as_survey_design(
          ids = upm,
          weights = fep,
          nest = TRUE
        )
      
      mod_calib <- as.formula(paste0("~ -1 +", paste0(var_calib, collapse = " + ")))
      diseno_calib <- tryCatch({
        calibrate(diseno_post, formula = mod_calib, 
                  population = Tx_hat[var_calib], calfun = "raking",
                  maxit = 50)
      }, error = function(e) {
        message("Error en la calibración: ", yks)
        return(NULL)
      })
      
      if (is.null(diseno_calib)) {
        next
      }
      
      estima_calib_ipm[[ii]] <- diseno_calib %>% group_by(cve_mun) %>%
        summarise_at(.vars = yks, .funs = list(
          ~ survey_mean(., vartype = "var", na.rm = TRUE))
        )
    }
    
    estima_calib_ipm <- keep(estima_calib_ipm, ~ !is.null(.)) %>%
      reduce(inner_join)
    
    saveRDS(list(estima_calib = estima_calib_ipm),
            file = paste0("../output/2020/iteraciones/mpio_calib_carencia/",
                           ii_ent, "/iter", iter, ".rds"))
    gc()
  }
  
  fin <- Sys.time()
  tiempo_total <- difftime(fin, inicio, units = "mins")
  print(tiempo_total)
  cat("####################################################################\n")
}
```

