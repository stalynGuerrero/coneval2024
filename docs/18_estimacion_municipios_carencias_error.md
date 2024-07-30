# 18_estimacion_municipios_carencias_error.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **18_estimacion_municipios_carencias_error.R** disponible en la ruta *Rcodes/2020/18_estimacion_municipios_carencias_error.R*.

Este script en R está diseñado para procesar y analizar datos de encuestas sobre pobreza en México para el año 2020. El código comienza limpiando el entorno de trabajo con `rm(list = ls())` y cargando las bibliotecas necesarias (`tidyverse`, `magrittr`, `survey`, `srvyr`, `tidyselect`) para la manipulación de datos y análisis estadístico.

La primera sección del script se enfoca en la lectura de datos. Se cargan dos conjuntos de datos usando `readRDS()`: `encuesta_ampliada` y `encuesta_enigh`, este último con una modificación en la columna `ingreso`. También se lee un archivo CSV con `read.delim()`, que contiene datos sobre las líneas de bienestar, y se convierte la columna `area` a formato de carácter.

En la sección del Índice de Pobreza Multidimensional en la encuesta ENIGH, se realiza una combinación interna (`inner_join()`) entre `encuesta_enigh` y `LB` para añadir variables relacionadas con carencias sociales y pobreza. Se crean nuevas variables en `encuesta_enigh`, tales como `tol_ic` (suma de indicadores de carencia social), `pobrea_lp` (indicador de pobreza por ingresos) y `pobrea_li` (indicador de pobreza extrema por ingresos). Además, se genera un resumen del número de personas por municipio en `N_mpio`. A continuación, el script lee una lista de archivos de estimación de carencia desde un directorio específico, y procesa estos archivos para calcular diversas estadísticas como media, varianza y errores estándar de las estimaciones. Utiliza la función `map_df()` para combinar datos de múltiples archivos, y realiza cálculos de varianza y errores estándar para cada indicador de pobreza. Los resultados se almacenan en una lista y se combinan en un único marco de datos.

Los resultados combinados (`temp`) se enriquecen con información de población de `N_mpio` y se guardan en archivos RDS y Excel utilizando `saveRDS()` y `write.xlsx()`. Se calcula también un resumen a nivel estatal, agregando los resultados por entidad. En la última sección del script, se realiza un análisis gráfico de las estimaciones de pobreza. Se convierte `encuesta_enigh` en un diseño de encuesta usando `as_survey_design()`, y se calculan las estimaciones de pobreza por entidad. Luego, se generan gráficos con `ggplot2` para comparar las estimaciones de carencia entre los datos de la encuesta ENIGH y las estimaciones de CEPAL. Los gráficos se guardan como archivos PNG utilizando `ggsave()`.

#### Limpieza del Entorno y Carga de Bibliotecas{-}

Se limpia el entorno de trabajo y se cargan las bibliotecas necesarias para la manipulación de datos y la visualización.


``` r
   rm(list = ls())
   library(tidyverse)
   library(magrittr)
   library(survey)
   library(srvyr)
   library(tidyselect)
```


#### Lectura de Datos{-}

Se cargan los datos de las encuestas y las líneas de bienestar desde archivos RDS y CSV.



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

#### Cálculo de Indicadores de Pobreza{-}

Se calculan varios indicadores de pobreza basados en el ingreso y las carencias sociales.


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

Se listan y procesan los directorios que contienen resultados iterativos.


``` r
   list_estiacion <- list.files("../../output/2020/iteraciones/mpio_calib_carencia//", full.names = TRUE)
   list_estiacion <- data.frame(list_estiacion, iter = list_estiacion %>% map_dbl(~list.files(.x) %>% length())) %>%
     filter(iter > 0)

   resul_ent_ipm <- list()
```


#### Combinación de Resultados Iterativos{-}
Se combinan los resultados de las iteraciones en un único conjunto de datos.
 

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

Se guardan los resultados combinados en archivos RDS y Excel.


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

Se generan y guardan gráficos que comparan los indicadores de pobreza entre diferentes fuentes de datos.


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

