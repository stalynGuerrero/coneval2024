# 18_estimacion_municipios_carencias_error.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **18_estimacion_municipios_carencias_error.R** disponible en la ruta *Rcodes/2020/18_estimacion_municipios_carencias_error.R*.

Este script en R está diseñado para procesar y analizar datos de encuestas sobre pobreza en México para el año 2020. El código comienza limpiando el entorno de trabajo con `rm(list = ls())` y cargando las bibliotecas necesarias (`tidyverse`, `magrittr`, `survey`, `srvyr`, `tidyselect`) para la manipulación de datos y análisis estadístico.

La primera sección del script se enfoca en la lectura de datos. Se cargan dos conjuntos de datos usando `readRDS()`: `encuesta_ampliada` y `encuesta_enigh`, este último con una modificación en la columna `ingreso`. También se lee un archivo CSV con `read.delim()`, que contiene datos sobre las líneas de bienestar, y se convierte la columna `area` a formato de carácter.

En la sección del Índice de Pobreza Multidimensional en la encuesta ENIGH, se realiza una combinación interna (`inner_join()`) entre `encuesta_enigh` y `LB` para añadir variables relacionadas con carencias sociales y pobreza. Se crean nuevas variables en `encuesta_enigh`, tales como `tol_ic` (suma de indicadores de carencia social), `pobrea_lp` (indicador de pobreza por ingresos) y `pobrea_li` (indicador de pobreza extrema por ingresos). Además, se genera un resumen del número de personas por municipio en `N_mpio`. A continuación, el script lee una lista de archivos de estimación de carencia desde un directorio específico, y procesa estos archivos para calcular diversas estadísticas como media, varianza y errores estándar de las estimaciones. Utiliza la función `map_df()` para combinar datos de múltiples archivos, y realiza cálculos de varianza y errores estándar para cada indicador de pobreza. Los resultados se almacenan en una lista y se combinan en un único marco de datos.

Los resultados combinados (`temp`) se enriquecen con información de población de `N_mpio` y se guardan en archivos RDS y Excel utilizando `saveRDS()` y `write.xlsx()`. Se calcula también un resumen a nivel estatal, agregando los resultados por entidad. En la última sección del script, se realiza un análisis gráfico de las estimaciones de pobreza. Se convierte `encuesta_enigh` en un diseño de encuesta usando `as_survey_design()`, y se calculan las estimaciones de pobreza por entidad. Luego, se generan gráficos con `ggplot2` para comparar las estimaciones de carencia entre los datos de la encuesta ENIGH y las estimaciones de CEPAL. Los gráficos se guardan como archivos PNG utilizando `ggsave()`.

#### Limpieza del Entorno y Carga de Bibliotecas{-}

En esta sección, el código se ocupa de limpiar el entorno de trabajo y cargar las bibliotecas necesarias para la manipulación y análisis de datos, así como para la visualización. Este proceso garantiza que el entorno esté libre de interferencias de objetos previos y que las herramientas requeridas estén disponibles para ejecutar el resto del script de manera eficiente.


``` r
   rm(list = ls())
   library(tidyverse)
   library(magrittr)
   library(survey)
   library(srvyr)
   library(tidyselect)
```

- **Limpieza del entorno**: El comando `rm(list = ls())` se utiliza para eliminar todos los objetos en el entorno de trabajo de R. Esto asegura que no haya residuos de datos anteriores que puedan afectar el nuevo análisis.
- **Carga de bibliotecas**:
  - `tidyverse`: Conjunto de paquetes que incluye `ggplot2`, `dplyr`, `tidyr`, `readr`, `purrr`, `tibble`, entre otros. Estos paquetes son esenciales para la manipulación de datos, análisis y visualización.
  - `magrittr`: Proporciona el operador de pipe (`%>%`), facilitando la escritura de código legible y conciso al encadenar funciones.
  - `survey`: Ofrece herramientas para el análisis de datos de encuestas complejas, permitiendo el diseño de muestras y la estimación de estadísticas ponderadas.
  - `srvyr`: Extiende las funcionalidades del paquete `survey` utilizando la sintaxis de `dplyr`, lo que facilita la manipulación y resumen de datos de encuestas.
  - `tidyselect`: Utilizado para facilitar la selección de columnas en operaciones de manipulación de datos dentro del ecosistema de tidyverse.

Este paso es crucial para asegurar un entorno limpio y preparado para realizar análisis de datos precisos y eficientes.

##### Lectura de Datos{-}

En esta sección, el código se encarga de leer y cargar los datos necesarios para el análisis desde archivos RDS y CSV. Estos datos incluyen encuestas ampliadas, encuestas ENIGH y líneas de bienestar.


``` r
encuesta_ampliada <- readRDS("../../output/2020/encuesta_ampliada.rds")
encuesta_enigh <-
  readRDS("../input/2020/enigh/encuesta_sta.rds") %>%
  mutate(ingreso = ictpc)

LB <-
  read.delim(
    "../input/2020/Lineas_Bienestar.csv",
    header = TRUE,
    sep = ";",
    dec = ","
  ) %>%
  mutate(area = as.character(area))
```

- **Lectura de encuesta ampliada**:
  - `encuesta_ampliada <- readRDS("../../output/2020/encuesta_ampliada.rds")`: Carga el archivo RDS que contiene la encuesta ampliada. Este archivo se encuentra en la carpeta de salida `output/2020` y contiene datos detallados sobre las encuestas realizadas en 2020.

- **Lectura de encuesta ENIGH**:
  - `encuesta_enigh <- readRDS("../input/2020/enigh/encuesta_sta.rds")`: Carga el archivo RDS que contiene los datos de la Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH) de 2020.
  - `mutate(ingreso = ictpc)`: Añade una nueva columna `ingreso` a la encuesta ENIGH, utilizando la columna `ictpc` que representa el ingreso total per cápita.

- **Lectura de líneas de bienestar**:
  - `LB <- read.delim("../input/2020/Lineas_Bienestar.csv", header = TRUE, sep = ";", dec = ",")`: Lee el archivo CSV que contiene las líneas de bienestar. Este archivo se encuentra en la carpeta `input/2020` y contiene información sobre las líneas de pobreza y bienestar.
  - `mutate(area = as.character(area))`: Convierte la columna `area` en tipo de dato carácter, asegurando una correcta manipulación de los datos en análisis posteriores.

Este paso es esencial para preparar y estructurar los datos que se utilizarán en el análisis, asegurando que toda la información necesaria esté disponible y correctamente formateada.

#### Cálculo de Indicadores de Pobreza{-}

En esta sección, se calculan diversos indicadores de pobreza utilizando los datos de la encuesta ENIGH y las líneas de bienestar. El código une los datos de la encuesta ENIGH con las líneas de bienestar (`LB`) mediante una unión interna, asegurando que solo se conserven las observaciones presentes en ambos conjuntos de datos. A continuación, se calcula el total de carencias sociales (`tol_ic`) sumando las diferentes carencias individuales (`ic_segsoc`, `ic_ali_nc`, `ic_asalud`, `ic_cv`, `ic_sbv`, `ic_rezedu`). Se determinan indicadores binarios de pobreza basados en el ingreso, como `pobrea_lp`, que indica si un hogar está por debajo de la línea de pobreza (`lp`), y `pobrea_li`, que indica si un hogar está por debajo de la línea de indigencia (`li`). Asimismo, se calculan indicadores de pobreza basados en el número de carencias sociales, como `tol_ic_1`, que indica si un hogar tiene al menos una carencia social, y `tol_ic_2`, que indica si un hogar tiene más de dos carencias sociales. Estos cálculos permiten evaluar diferentes dimensiones de la pobreza, combinando tanto el ingreso como las carencias sociales para ofrecer una visión más completa de la situación socioeconómica de los hogares.



``` r
encuesta_enigh <- encuesta_enigh %>% inner_join(LB) %>%
mutate(
tol_ic = ic_segsoc + ic_ali_nc + ic_asalud + ic_cv + ic_sbv + ic_rezedu,
pobrea_lp = ifelse(ingreso < lp, 1, 0),
pobrea_li = ifelse(ingreso < li, 1, 0),
tol_ic_1 = ifelse(tol_ic > 0, 1, 0),
tol_ic_2 = ifelse(tol_ic > 2, 1, 0)
)
```


#### Procesamiento de Resultados Iterativos{-}

Esta sección se encarga de listar y procesar los directorios que contienen los resultados de iteraciones previas, almacenados en archivos. Primero, se listan todos los directorios dentro de la ruta especificada (`../output/2020/iteraciones/mpio_calib_carencia//`) utilizando la función `list.files`, incluyendo sus rutas completas. Luego, se crea un data frame (`list_estiacion`) que contiene las rutas de los directorios y la cantidad de archivos en cada uno de ellos, que se determina aplicando la función `length` a la lista de archivos en cada directorio. Se filtran los directorios para conservar solo aquellos que contienen al menos un archivo de iteración (es decir, aquellos con `iter > 0`). Finalmente, se inicializa una lista vacía (`resul_ent_ipm`) para almacenar los resultados procesados de estas iteraciones.


``` r
   list_estiacion <- list.files("../output/2020/iteraciones/mpio_calib_carencia//", full.names = TRUE)
   list_estiacion <- data.frame(list_estiacion, iter = list_estiacion %>% map_dbl(~list.files(.x) %>% length())) %>%
     filter(iter > 0)

   resul_ent_ipm <- list()
```


#### Combinación de Resultados Iterativos{-}

Esta sección combina los resultados de las iteraciones en un único conjunto de datos. Primero, se recorre cada directorio listado en `list_estiacion$list_estiacion`. Para cada directorio, se obtiene la lista de archivos dentro de él y se leen los datos contenidos en cada archivo utilizando la función `readRDS`, combinando los datos en un solo data frame con `map_df`. Luego, se agrupan los datos por `cve_mun` y se calculan diversas estadísticas: las medias (`summarise_at(vars(!matches("var")), mean)`), el conteo (`tally()`), las varianzas (`summarise_at(vars(!matches("var")), var)`), y las medias de las varianzas (`summarise_at(vars(matches("var")), mean)`). 

Posteriormente, los datos se transforman y combinan para calcular las varianzas totales y los errores estándar (`transmute(cve_mun, Indicador, var = Ubar + (1 + 1 / n) * B, ee = sqrt(var))`). Los resultados se reestructuran utilizando `pivot_wider` para obtener el formato deseado y se guardan en la lista `resul_ent_ipm` para cada entidad procesada. Finalmente, se unen los diversos data frames generados (`dat_n`, `dat_estima`, `dat_var`, `var_est`) en un único data frame por entidad y se almacenan en la lista de resultados.

 

``` r
for (ii_ent in list_estiacion$list_estiacion) {
  archivos <- list.files(ii_ent, full.names = TRUE)
  datos <- map_df(archivos, ~ readRDS(.x)$estima_calib)
  
  dat_estima <-
    datos %>% group_by(cve_mun) %>% summarise_at(vars(!matches("var")), mean) %>% data.frame()
  dat_n <- datos %>% group_by(cve_mun) %>% tally()
  dat_B <-
    datos %>% group_by(cve_mun) %>% summarise_at(vars(!matches("var")), var) %>% data.frame()
  
  dat_Ubar <-
    datos %>% group_by(cve_mun) %>% summarise_at(vars(matches("var")), mean) %>% data.frame()
  names(dat_Ubar) <- gsub("_var", "", names(dat_Ubar))
  
  dat_var <-
    dat_B %>% gather(key = "Indicador", value = "B", -cve_mun) %>%
    inner_join(dat_Ubar %>% gather(key = "Indicador", value = "Ubar", -cve_mun)) %>%
    inner_join(dat_n)
  
  var_est <-
    dat_var %>% transmute(cve_mun,
                          Indicador,
                          var = Ubar + (1 + 1 / n) * B,
                          ee = sqrt(var)) %>%
    pivot_wider(
      id_cols = "cve_mun",
      names_from = "Indicador",
      values_from = c("var", "ee"),
      values_fill = 0
    )
  
  dat_var <-
    pivot_wider(
      data = dat_var,
      id_cols = "cve_mun",
      names_from = "Indicador",
      values_from = c("B", "Ubar"),
      values_fill = 0
    )
  
resul_ent_ipm[[ii_ent]] <-
    dat_n %>% inner_join(dat_estima) %>% inner_join(dat_var) %>% inner_join(var_est)
}
```

#### Guardado de Resultados{-}

En esta sección se guardan los resultados combinados en archivos RDS y Excel. Primero, se unen todos los data frames contenidos en la lista `resul_ent_ipm` utilizando `bind_rows()`. A continuación, se hace un `inner_join` con el data frame `N_mpio` y se seleccionan las columnas relevantes: `ent`, `cve_mun`, los indicadores de pobreza y carencias (`pobrea_lp`, `pobrea_li`, `tol_ic_1`, `tol_ic_2`, `ic_segsoc`, `ic_ali_nc`), `N_pers_mpio` y las columnas que coinciden con `ee`. Este conjunto de datos se guarda en un archivo RDS con la función `saveRDS`.

Luego, se guarda el mismo conjunto de datos en un archivo Excel utilizando la función `write.xlsx` de la biblioteca `openxlsx`. Se genera un nombre de archivo que incluye la fecha actual obtenida con `Sys.Date()`.

Para obtener un resumen a nivel de entidad, se seleccionan las columnas relevantes (`ent`, `pobrea_lp`, `pobrea_li`, `tol_ic_1`, `tol_ic_2`, `ic_segsoc`, `ic_ali_nc`, `N_pers_mpio`) y se agrupan por `ent`. Se calculan las medias ponderadas de los indicadores de pobreza y carencias, ponderadas por el número de personas en cada municipio (`N_pers_mpio`). Este resumen se guarda en otro archivo Excel, también con un nombre que incluye la fecha actual.



``` r
temp <- resul_ent_ipm %>% bind_rows()
temp %<>% inner_join(N_mpio) %>%
  select(
    ent,
    cve_mun,
    "pobrea_lp",
    "pobrea_li",
    "tol_ic_1",
    "tol_ic_2",
    "ic_segsoc",
    "ic_ali_nc",
    N_pers_mpio,
    matches("ee")
  )

temp %>% saveRDS("../output/Entregas/2020/result_mpios_carencia.RDS")

openxlsx::write.xlsx(
  temp,
  paste0(
    "../output/Entregas/2020/estimacion_numicipal_carencia_",
    Sys.Date(),
    ".xlsx"
  )
)

temp2 <- temp %>%
  select(ent, pobrea_lp:ic_ali_nc, N_pers_mpio) %>%
  group_by(ent) %>%
  summarise(across(
    c(
      "pobrea_lp",
      "pobrea_li",
      "tol_ic_1",
      "tol_ic_2",
      "ic_segsoc",
      "ic_ali_nc"
    ),
    ~ sum(.x * N_pers_mpio) / sum(N_pers_mpio)
  ))

openxlsx::write.xlsx(
  temp2,
  paste0(
    "../output/Entregas/2020/estimacion_estado_carencia_",
    Sys.Date(),
    ".xlsx"
  )
)
```


#### Visualización de Resultados{-}

En esta sección se generan y guardan gráficos que comparan los indicadores de pobreza entre diferentes fuentes de datos. Primero, se crea un diseño de encuesta utilizando la base de datos `encuesta_enigh` con la función `as_survey_design`, eliminando previamente los valores faltantes. Con este diseño, se calculan estimaciones directas de los indicadores de pobreza y carencias (`pobrea_lp`, `pobrea_li`, `tol_ic_1`, `tol_ic_2`, `ic_segsoc`, `ic_ali_nc`) para cada entidad (`ent`), incluyendo intervalos de confianza (`ci`).

Los indicadores de interés se almacenan en el vector `ind`, y se itera sobre cada uno de ellos para generar los gráficos correspondientes. Para cada indicador, se filtran y preparan los datos de comparación entre las estimaciones directas (ENIGH) y las estimaciones calculadas (CEPAL), utilizando `inner_join` y funciones de manipulación de datos de `dplyr` y `tidyr`. Se calculan los límites inferior y superior de los intervalos de confianza y se reorganizan los datos para su visualización.

Se genera un gráfico utilizando `ggplot2`, con las entidades en el eje x y las proporciones estimadas en el eje y. Las estimaciones de ENIGH y CEPAL se diferencian por colores. Se añaden barras de error para los intervalos de confianza de ENIGH y se ajustan los elementos visuales del gráfico para mejorar la presentación. Finalmente, se guarda cada gráfico como un archivo PNG con un nombre que incluye el indicador correspondiente.


``` r
diseno <- encuesta_enigh %>% na.omit() %>%
  as_survey_design(ids = upm,
                   weights = fep,
                   nest = TRUE)

estimad_dir <- diseno %>% group_by(ent) %>%
  summarise_at(
    .vars = c(
      "pobrea_lp",
      "pobrea_li",
      "tol_ic_1",
      "tol_ic_2",
      "ic_segsoc",
      "ic_ali_nc"
    ),
    .funs = list(dir = ~ survey_mean(., vartype = "ci", na.rm = TRUE))
  )

ind <-
  c("pobrea_lp",
    "pobrea_li",
    "tol_ic_1",
    "tol_ic_2",
    "ic_segsoc",
    "ic_ali_nc")

for (ii_ipm in 1:6) {
  ii_ipm <- ind[ii_ipm]
  paso <- paste0("(_|\\b)", ii_ipm, "(_|\\b)")
  dat_plot <- inner_join(estimad_dir, temp2) %>%
    select(ent, matches(paso))
  
  dat_lim <- dat_plot %>% select(ent, matches("upp|low"))
  names(dat_lim) <- c("ent", "Lim_Inf", "Lim_Sup")
  
  dat_plot %<>% select(-matches("upp|low")) %>%
    gather(key = "Origen", value = "Prop", -ent) %>%
    mutate(Origen = ifelse(grepl(pattern = "dir", x = Origen), "ENIGH", "Estimación CEPAL")) %>%
    inner_join(dat_lim)
  
  gg_plot <-
    ggplot(data = dat_plot, aes(x = ent, y = Prop, color = Origen)) +
    labs(
      x = "",
      y = "Estimación",
      color = "",
      title = paste0("Estimación de la carencia 2020 -", ii_ipm)
    ) +
    theme_bw(20) +
    geom_jitter(width = 0.3) +
    theme(legend.position = "bottom",
          plot.title = element_text(hjust = 0.5)) +
    scale_y_continuous(labels = scales::percent_format(scale = 100))
  
  gg_plot <- gg_plot +
    geom_errorbar(
      data = dat_plot %>% filter(Origen == "ENIGH"),
      aes(ymin = Lim_Inf, ymax = Lim_Sup, x = ent),
      width = 0.2,
      linewidth = 1
    ) +
    scale_color_manual(
      breaks = c("ENIGH", "Estimación CEPAL"),
      values = c("red", "blue3")
    ) +
    theme(
      legend.position = "bottom",
      axis.title = element_text(size = 10),
      axis.text.y = element_text(size = 10),
      axis.text.x = element_text(
        angle = 90,
        size = 8,
        vjust = 0.3
      ),
      legend.title = element_text(size = 15),
      legend.text = element_text(size = 15)
    )
  
  print(gg_plot)
  ggsave(
    plot = gg_plot,
    width = 16,
    height = 9,
    filename = paste0("../output/Entregas/2020/plot_uni/ipm_", ii_ipm, ".png")
  )
}
```

