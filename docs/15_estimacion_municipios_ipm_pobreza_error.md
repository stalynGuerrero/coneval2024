# 15_estimacion_municipios_ipm_pobreza_error.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **15_estimacion_municipios_ipm_pobreza_error.R** disponible en la ruta **Rcodes/2020/15_estimacion_municipios_ipm_pobreza_error.R**.


El script comienza con la limpieza del entorno de trabajo y la carga de librerías necesarias, como `tidyverse` y `survey`. A continuación, se cargan y preparan los datos de `muestra_ampliada`, `encuesta_enigh`, y `LB`. Se calculan las variables relacionadas con la pobreza multidimensional (IPM) en la encuesta, incluyendo la clasificación en categorías como "I", "II", "III", y "IV", y se identifican los casos de pobreza moderada y extrema.

Luego, se procesa la información de los resultados de estimación por municipio, que se leen desde archivos generados en iteraciones previas. Se calculan y combinan las estimaciones, varianzas y medias, y se consolidan en un solo conjunto de datos. Estos resultados se guardan en archivos RDS y Excel para su posterior análisis.

Finalmente, se realiza una visualización comparativa de las estimaciones de pobreza multidimensional. Utilizando `ggplot2`, se crean gráficos que comparan las estimaciones obtenidas de la ENIGH con las de CEPAL para cada estado, y estos gráficos se exportan en formato PNG. Este paso permite la evaluación visual de la precisión y consistencia de las estimaciones realizadas.


El código que compartiste está orientado a procesar y analizar datos de una encuesta para calcular y comparar las estimaciones de pobreza y el Índice de Pobreza Multidimensional (IPM). A continuación, te proporciono un desglose de las principales secciones y acciones del código:

####  Preparación del Entorno{-}

En esta etapa se realiza la limpieza del entorno de trabajo y se cargan las bibliotecas necesarias para llevar a cabo el análisis. Primero, se elimina cualquier objeto que pueda haber quedado en el entorno global, utilizando el comando `rm(list = ls())`. Este paso es crucial para evitar interferencias de variables o datos residuales de análisis previos.

A continuación, se cargan las bibliotecas necesarias para el análisis. Las bibliotecas incluyen `tidyverse`, un conjunto de paquetes diseñado para facilitar la manipulación y visualización de datos, `magrittr`, que proporciona el operador de encadenamiento `%>%` para escribir código más legible y fluido, `survey`, que ofrece herramientas para el análisis de datos de encuestas complejas, y `srvyr`, que permite realizar operaciones sobre datos de encuestas utilizando una sintaxis similar a `dplyr`. La carga de estas bibliotecas asegura que se disponga de las herramientas adecuadas para manejar y analizar los datos de manera efectiva.



``` r
rm(list = ls())
library(tidyverse)
library(magrittr)
library(survey)
library(srvyr)
```



#### Lectura de Datos{-}

En esta sección se lleva a cabo la carga y preparación de los datos necesarios para el análisis. Se leen tres conjuntos de datos clave: `muestra_ampliada`, `encuesta_enigh`, y `LB`.

Primero, se carga el archivo de muestra ampliada, almacenado en el archivo `../output/2020/encuesta_ampliada.rds`, utilizando la función `readRDS()`. Este archivo contiene la muestra ampliada del censo, que incluye una serie de variables importantes para el análisis posterior.

Luego, se carga el archivo `encuesta_enigh`, ubicado en `../input/2020/enigh/encuesta_sta.rds`. Este archivo se lee también mediante `readRDS()`, y se le añade una columna nueva llamada `ingreso`, que toma los valores de la columna `ictpc`. Esta columna `ingreso` se usará para realizar cálculos y análisis en los pasos siguientes.

Finalmente, se lee el archivo `Lineas_Bienestar.csv` con la función `read.delim()`, especificando que el archivo está separado por punto y coma (`sep = ";"`) y que los decimales están representados por comas (`dec = ","`). Este archivo contiene datos sobre las líneas de bienestar, y se convierte la columna `area` a formato carácter para asegurar una correcta manipulación de los datos en el análisis. 

Estos datos, una vez cargados, se preparan para el análisis mediante la adición de columnas y el ajuste de formatos necesarios.


``` r
muestra_ampliada <- readRDS("../output/2020/encuesta_ampliada.rds")
encuesta_enigh <- readRDS("../input/2020/enigh/encuesta_sta.rds") %>%
  mutate(ingreso = ictpc)

LB <- read.delim(
    "input/2020/Lineas_Bienestar.csv",
    header = TRUE,
    sep = ";",
    dec = ","
  ) %>% mutate(area = as.character(area))
```


#### Cálculo del Índice de Pobreza Multidimensional (IPM) en `encuesta_enigh` {-}

Esta sección del código realiza el cálculo del Índice de Pobreza Multidimensional (IPM) y de variables asociadas utilizando los datos de la encuesta `encuesta_enigh` y la información sobre las líneas de bienestar `LB`. El objetivo es integrar la información de las carencias y el ingreso para clasificar a los individuos en diferentes categorías de pobreza y vulnerabilidad.

1. **Integración de Datos**: Primero, se realiza una unión interna (`inner_join()`) entre `encuesta_enigh` y `LB`, añadiendo las variables de las líneas de bienestar (`LB`) a los datos de la encuesta.

2. **Cálculo de `tol_ic`**: Se calcula la variable `tol_ic`, que representa el total de las carencias sociales que un individuo enfrenta. Esta variable es la suma de varias carencias (`ic_segsoc`, `ic_ali_nc`, `ic_asalud`, `ic_cv`, `ic_sbv`, y `ic_rezedu`).

3. **Determinación del IPM**:
   - **Pobreza Multidimensional (`ipm`)**: Se clasifica a cada individuo en una de las cuatro categorías de pobreza multidimensional basadas en el ingreso (`ingreso`) y el total de carencias (`tol_ic`).

4. **Variables de Pobreza Moderada y Extrema**:
   - **Pobreza Moderada (`pobre_moderada`)**: Se define como 1 si el ingreso está entre la línea de pobreza extrema (`li`) y la línea de pobreza (`lp`), y el total de carencias sociales (`tol_ic`) es mayor que 2; en caso contrario, se define como 0.
   - **Pobreza Extrema (`pobre_extrema`)**: Se define como 1 si el individuo está en la categoría de pobreza multidimensional "I" y no es considerado pobre moderado; en caso contrario, se define como 0.

Estas transformaciones y cálculos permiten clasificar y analizar el nivel de pobreza y las condiciones de vulnerabilidad de los individuos en la encuesta `encuesta_enigh`.


``` r
encuesta_enigh <-
  encuesta_enigh %>% inner_join(LB)  %>% 
  mutate(
    tol_ic = ic_segsoc + ic_ali_nc + ic_asalud + ic_cv +  ic_sbv + ic_rezedu,
    ipm   = case_when(
      ingreso < lp  &  tol_ic >= 1 ~ "I",
      ingreso >= lp & tol_ic >= 1 ~ "II",
      ingreso <= lp & tol_ic < 1 ~ "III",
      ingreso >= lp & tol_ic < 1 ~ "IV"
    ),
    pobre_moderada = ifelse(c(ingreso > li & ingreso < lp) &
                              tol_ic > 2, 1, 0),
    pobre_extrema = ifelse(ipm == "I" & pobre_moderada == 0, 1, 0)
  )
```



#### Resumen de Datos por Municipio{-}

El análisis de los datos de la `muestra_ampliada` se centra en calcular la cantidad total de personas en cada municipio. Para lograr esto, se agrupan los datos por entidad federativa y por clave de municipio. Luego, se suma el número de personas, representado por la variable `factor`, para cada municipio específico. Esta operación proporciona un resumen detallado de la población en cada municipio, generando un `data frame` que incluye las columnas `ent` (entidad federativa), `cve_mun` (clave del municipio) y `N_pers_mpio` (número total de personas en el municipio). Este resumen es fundamental para la correcta calibración de los modelos y la interpretación de los resultados a nivel municipal.



``` r
N_mpio <- muestra_ampliada %>% group_by(ent, cve_mun) %>%
  summarise(N_pers_mpio = sum(factor))
```


#### Lectura de Archivos de Iteraciones y Cálculo de Estadísticas{-}

Se procede a la lectura de los archivos generados durante las iteraciones del ajuste de modelos y calibración. Primero, se obtienen todos los archivos ubicados en el directorio `../output/2020/iteraciones/mpio_calib/`, que contienen los resultados de las iteraciones realizadas para cada entidad federativa. A continuación, se crea un `data frame` que incluye el nombre completo de cada directorio de iteración y la cantidad de archivos presentes en cada uno. Este `data frame` se filtra para mantener únicamente aquellos directorios que contienen al menos un archivo, lo que indica que se han generado resultados en esas iteraciones. Este proceso facilita el análisis posterior al asegurar que solo se consideren las iteraciones que tienen datos válidos para el análisis de las estimaciones.


``` r
list_estiacion <- list.files("../output/2020/iteraciones/mpio_calib/", full.names = TRUE)

list_estiacion <- data.frame(list_estiacion,
iter = list_estiacion %>% map_dbl(~list.files(.x) %>% length())
) %>% filter(iter > 0)
```


#### Proceso Iterativo para Cada estado {-}

Para cada entidad federativa, se lleva a cabo un análisis iterativo para calcular estimaciones promedio, varianza y otros indicadores estadísticos relevantes. Este proceso comienza con la lectura de los archivos de iteraciones para cada estado, que contienen resultados de las estimaciones calibradas. Los archivos se cargan y se combinan en un solo `data frame` para su procesamiento.

Para cada estado, se realiza una serie de cálculos estadísticos. Primero, se obtiene la estimación promedio para cada indicador por municipio, calculando la media de las estimaciones en los datos. Posteriormente, se calcula el tamaño de la muestra para cada municipio y la varianza de las estimaciones. La varianza se descompone en la varianza entre las iteraciones (B) y la varianza promedio (Ubar) dentro de las iteraciones.

Se realiza una combinación de estos datos para calcular la varianza total de las estimaciones, sumando la varianza promedio (Ubar) con la varianza entre las iteraciones ajustada por el tamaño de la muestra. La estimación del error estándar se obtiene como la raíz cuadrada de la varianza.

Finalmente, se organiza toda la información en un `data frame` que incluye el número de observaciones, las estimaciones promedio, las varianzas y los errores estándar para cada indicador por municipio. Todos estos resultados se guardan en una lista para su posterior análisis y evaluación.


``` r
resul_ent_ipm <- list()

for(ii_ent in list_estiacion$list_estiacion){
  
  archivos <- list.files(ii_ent, full.names = TRUE)
  datos <- map_df(archivos, ~ readRDS(.x)$estima_calib)

  dat_estima <- datos %>% group_by(cve_mun) %>% 
    summarise_at(vars(!matches("var")), mean) %>% data.frame()

  dat_n <- datos %>% group_by(cve_mun) %>% tally()
  dat_B <- datos %>% group_by(cve_mun) %>% 
    summarise_at(vars(!matches("var")), var) %>% data.frame()

  dat_Ubar <- datos %>% group_by(cve_mun) %>% 
    summarise_at(vars(matches("var")), mean) %>% data.frame()

  names(dat_Ubar) <- gsub("_var", "", names(dat_Ubar)) 

  dat_var <- dat_B %>% 
    gather(key = "Indicador", value = "B", -cve_mun) %>%
    inner_join(
      dat_Ubar %>% gather(key = "Indicador", value = "Ubar", -cve_mun)
    ) %>% inner_join(dat_n)

  var_est <-  dat_var %>% 
    transmute(cve_mun, Indicador,
              var = Ubar + (1 +1/n)*B,
              ee = sqrt(var)) %>% 
    pivot_wider(
      data = .,
      id_cols = "cve_mun",
      names_from = "Indicador",
      values_from = c("var", "ee"), values_fill = 0
    ) 

  dat_var <- pivot_wider(
    data = dat_var,
    id_cols = "cve_mun",
    names_from = "Indicador",
    values_from = c("B", "Ubar"), values_fill = 0
  ) 

  resul_ent_ipm[[ii_ent]] <- dat_n %>% inner_join(dat_estima) %>%
    inner_join(dat_var) %>%
    inner_join(var_est)
}
```



#### Guardar Resultados y Crear Archivos Excel{-}

Una vez calculadas todas las estimaciones y estadísticas para cada entidad federativa, los resultados se consolidan y se guardan en varios formatos para su posterior análisis y distribución. 

Primero, los resultados de todas las entidades federativas se combinan en un solo `data frame`. Este `data frame` se enriquece con el número de personas por municipio, lo que permite contextualizar las estimaciones. Se seleccionan las columnas relevantes, que incluyen estimaciones de diversos indicadores, errores estándar y número de personas por municipio.

El `data frame` consolidado se guarda en dos formatos. Se utiliza `saveRDS()` para guardar los datos en un archivo RDS, que es adecuado para el almacenamiento y recuperación eficiente en R. Además, se utiliza `openxlsx::write.xlsx()` para exportar los resultados a un archivo Excel, facilitando su revisión y análisis en herramientas de hojas de cálculo.

Además, se calcula una estimación agregada a nivel estatal. Para esto, se agrupan los resultados por entidad federativa y se calculan las estimaciones totales para cada indicador, ponderadas por el número de personas en cada municipio. Este resumen estatal se guarda en otro archivo Excel separado, proporcionando una visión general de las estimaciones a nivel estatal.

Estos pasos garantizan que tanto los datos detallados por municipio como las estimaciones agregadas a nivel estatal estén disponibles en formatos accesibles y útiles para la toma de decisiones y la presentación de resultados.


``` r
temp <- resul_ent_ipm %>% bind_rows()

temp %<>% inner_join(N_mpio) %>% 
  select(ent, cve_mun, N_pers_mpio, est_ipm_I, 
         est_pob_mod, est_pob_ext, est_ipm_II:est_ipm_IV,
         ee_ipm_I = ee_est_ipm_I,
         ee_pob_mod = ee_est_pob_mod,
         ee_pob_ext = ee_est_pob_ext,
         ee_ipm_II = ee_est_ipm_II,
         ee_ipm_III = ee_est_ipm_III,
         ee_ipm_IV = ee_est_ipm_IV)

temp %>% 
  saveRDS("../output/Entregas/2020/result_mpios.RDS")

openxlsx::write.xlsx(temp,
                     paste0(
                       "../output/Entregas/2020/estimacion_numicipal_",
                       Sys.Date() ,
                       ".xlsx"
                     ))

temp2 <- temp %>%
  select(ent, est_ipm_I:est_ipm_IV, N_pers_mpio) %>%
  group_by(ent) %>%
  summarise(across(starts_with("est_"),
                   ~ sum(.x * N_pers_mpio) / sum(N_pers_mpio)))

openxlsx::write.xlsx(temp2,
                     paste0(
                       "../output/Entregas/2020/estimacion_estado_",
                       Sys.Date() ,
                       ".xlsx"
                     "))
```



#### Generación de Gráficos{-}

El proceso de generación de gráficos se enfoca en visualizar las estimaciones de pobreza multidimensional por entidad federativa para el año 2020. Los gráficos se crean para diferentes tipos de Índice de Pobreza Multidimensional (IPM) y se guardan en archivos PNG.

1. **Preparación de los Datos para Graficar:**
   Se utiliza el diseño muestral (`diseno`) de la encuesta ENIGH para calcular las estimaciones directas de los indicadores de pobreza multidimensional y sus intervalos de confianza. Estos cálculos se realizan para cada entidad federativa y se organizan en un `data frame` (`estimad_dir`), que se amplía para incluir las estimaciones directas de pobreza moderada y extrema.

2. **Configuración de los Datos para Cada IPM:**
   Se crea una lista de tipos de IPM (`ind`) que incluye tanto los tipos de IPM ("I", "II", "III", "IV") como las categorías de pobreza moderada ("mod") y extrema ("ext"). Para cada tipo, se seleccionan los datos relevantes y se preparan para la visualización.

3. **Generación de Gráficos:**
   Para cada tipo de IPM, se realiza lo siguiente:
   - Se filtran los datos para el tipo de IPM actual y se preparan las columnas para los límites de los intervalos de confianza.
   - Los datos se transforman y se preparan para graficar las estimaciones y sus intervalos de confianza.
   - Se crea un gráfico con `ggplot2` que muestra las estimaciones de pobreza por entidad federativa. Las estimaciones de la encuesta ENIGH se representan con barras de error, mientras que las estimaciones de CEPAL se muestran con puntos. Los gráficos se personalizan con colores distintos, etiquetas, y un tema limpio.
   - Los gráficos se guardan en archivos PNG con un tamaño adecuado para presentaciones y análisis.

Este proceso asegura que los resultados de las estimaciones de pobreza sean visualmente accesibles y comparables entre las distintas entidades federativas y los diferentes tipos de IPM.


``` r
diseno <- encuesta_enigh %>% na.omit() %>%
  as_survey_design(
    ids = upm,
    weights = fep,
    nest = TRUE,
    strata = estrato
  )

estimad_dir <- diseno %>% group_by(ent, ipm) %>% 
  summarise(direct = survey_mean(vartype = "ci" ))

estimad_dir_ipm <- pivot_wider(
  data = estimad_dir,
  id_cols = "ent",
  names_from = "ipm",
  values_from = c("direct", "direct_low", "direct_upp")
) 

estimad_dir <- diseno %>% group_by(ent) %>% 
  summarise(direct_mod = survey_mean(pobre_moderada, vartype = "ci"),
            direct_ext = survey_mean(pobre_extrema, vartype = "ci"))

estimad_dir %<>% inner_join(estimad_dir_ipm)

ind <- c("I", "II", "III", "IV", "mod", "ext")
ii_ipm  = 1
for (ii_ipm in 1:6){
    ii_ipm <- ind[ii_ipm]
    paso <- paste0("(_|\\b)", ii_ipm, "(_|\\b)")
    dat_plot <- inner_join(estimad_dir, temp2) %>% 
      select(ent, matches(paso))
  
    dat_lim  <- dat_plot %>% select(ent, matches("upp|low"))
   
    names(dat_lim) <- c("ent","Lim_Inf", "Lim_Sup")
   
    dat_plot %<>% select(-matches("upp|low")) %>% 
      gather(key = "Origen", value = "Prop", -ent) %>% 
      mutate(Origen = ifelse(grepl(pattern = "direct", x = Origen), 
                             "ENIGH", "Estimación CEPAL")) %>% 
      inner_join(dat_lim)
   
    gg_plot <- ggplot(data = dat_plot, aes(x = ent, y = Prop, color = Origen)) +
      labs(
        x = "",
        y = "Estimación",
        color = "",
        title = paste0("Estimación de la pobreza multidimensional 2020 - Tipo ", ii_ipm )
      )+ theme_bw(20) +
      geom_jitter(width = 0.3) +
      theme(legend.position

 = "bottom",
            plot.title = element_text(hjust = 0.5)) +
      scale_y_continuous(labels = scales::percent_format(scale = 100))
   
    gg_plot <-  gg_plot +
      geom_errorbar(data = dat_plot %>% filter(Origen   == "ENIGH"),
                    aes(ymin = Lim_Inf, ymax = Lim_Sup, x = ent),
                    width = 0.2, linewidth = 1) +
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
   
    ggsave(plot = gg_plot, width = 16, height = 9,
           filename = paste0("../output/Entregas/2020/plot_uni/ipm_", ii_ipm, ".png"))
}
```


