# 14_estimacion_municipios_ipm_pobreza.R {-}

Para la ejecución del presente análisis, se debe abrir el archivo **14_estimacion_municipios_ipm_pobreza.R** disponible en la ruta *Rcodes/2020/14_estimacion_municipios_ipm_pobreza.R*.

Este script en R realiza un análisis exhaustivo para calibrar y validar indicadores de pobreza a nivel municipal, utilizando modelos predictivos y datos de encuestas. Primero, el entorno de trabajo se limpia y se cargan las librerías necesarias para la manipulación de datos (`tidyverse`, `data.table`, `magrittr`, etc.) y para realizar cálculos estadísticos (`lme4`, `survey`, `srvyr`). Luego, se leen datos de predicciones y encuestas ampliadas, junto con información adicional sobre las líneas de bienestar.

En la sección inicial de procesamiento, se unen y mutan los datos de la encuesta ENIGH para calcular indicadores de pobreza y vulnerabilidad multidimensional. Se asignan categorías de pobreza (`ipm`) y se crean nuevas variables que reflejan la pobreza moderada y extrema, basadas en umbrales de ingreso y carencias sociales.

El script luego se centra en la preparación de datos para la calibración de modelos. La muestra ampliada se ajusta para incluir variables adicionales y predicciones de ingreso y carencias sociales. A continuación, se inicia un bucle que itera sobre un conjunto de códigos de entidad territorial. Para cada entidad, se filtran y ajustan los datos de la encuesta ampliada y se simulan nuevas variables (como ingresos y carencias sociales) utilizando distribuciones probabilísticas basadas en las predicciones anteriores. 

Dentro de cada iteración, se realiza una calibración de los datos utilizando el método de "raking", que ajusta las estimaciones de los indicadores a las estimaciones poblacionales conocidas. Los resultados de la calibración se calculan para cada municipio y se comparan con los datos originales para validar la precisión de las estimaciones. Finalmente, los resultados de cada iteración se guardan en archivos RDS separados para su análisis posterior y se reporta el tiempo total de ejecución del proceso. 

Este enfoque permite realizar un ajuste fino de los modelos predictivos y asegurar que las estimaciones de pobreza sean lo más precisas posible, utilizando simulaciones y técnicas avanzadas de calibración.


#### Limpieza del Entorno y Carga de Bibliotecas{-}

Antes de iniciar el análisis, se realiza una limpieza del entorno de trabajo para asegurar que no queden variables o datos residuales que puedan interferir con el análisis actual. Se utiliza `rm(list = ls())` para eliminar todos los objetos del entorno de R.

A continuación, se cargan las bibliotecas necesarias para el análisis. Estas bibliotecas incluyen:

- **`tidyverse`**: Para manipulación y visualización de datos.
- **`data.table`**: Para operaciones eficientes con datos en tablas.
- **`openxlsx`**: Para leer y escribir archivos de Excel.
- **`magrittr`**: Para usar el operador de tubería (`%>%`) que facilita la cadena de operaciones.
- **`haven`**: Para importar y exportar datos en formatos de software estadístico como SPSS, Stata, y SAS.
- **`labelled`**: Para gestionar y manipular etiquetas de variables.
- **`sampling`**: Para técnicas de muestreo y análisis relacionados.
- **`lme4`**: Para ajustar modelos lineales y no lineales de efectos mixtos.
- **`survey`**: Para análisis de datos de encuestas complejas.
- **`srvyr`**: Para aplicar métodos de análisis de encuestas utilizando una sintaxis similar a `dplyr`.

Finalmente, se carga un script adicional desde una ubicación específica, `benchmarking_indicador.R`, que probablemente contiene funciones y procedimientos específicos para el análisis de benchmarking que se realizará a continuación.


``` r
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

#### Lectura de Datos{-}

Se inicia el análisis cargando los datos necesarios y las predicciones de los modelos ajustados previamente. 

Primero, se leen las predicciones almacenadas en el archivo `predicciones.rds`, que contiene las predicciones para ingresos, alimentación, y seguridad social, así como la desviación estándar residual. 

Luego, se carga la muestra ampliada desde el archivo `encuesta_ampliada.rds`. Esta muestra ampliada es esencial para ajustar los valores y validar las predicciones en un contexto más amplio.

A continuación, se carga la encuesta ENIGH desde `encuesta_sta.rds`. Se realiza una mutación en los datos para agregar la variable de ingreso (`ingreso`) basada en el indicador `ictpc`.

Finalmente, se lee el archivo `Lineas_Bienestar.csv`, que contiene las líneas de bienestar. Estos datos se importan utilizando la función `read.delim` con las especificaciones adecuadas para el separador de campos y el decimal. Posteriormente, se convierte la columna `area` a formato de carácter para su uso en el análisis.


``` r
predicciones <- readRDS( "../output/2020/modelos/predicciones.rds")
muestra_ampliada <- readRDS("../output/2020/encuesta_ampliada.rds")
encuesta_enigh <- readRDS("../input/2020/enigh/encuesta_sta.rds") %>%
  mutate(ingreso = ictpc)

LB <- read.delim(
    "../input/2020/Lineas_Bienestar.csv",
    header = TRUE,
    sep = ";",
    dec = ","
  ) %>% mutate(area = as.character(area))
```

#### Cálculo del IPM en la Encuesta{-}

En esta sección, se calcula el Índice de Pobreza Multidimensional (IPM) y se definen nuevas variables relacionadas con la pobreza en la encuesta ENIGH.

Primero, se unen los datos de la encuesta ENIGH con las líneas de bienestar (`LB`) mediante la función `inner_join`. Esto permite integrar la información adicional necesaria para el cálculo del IPM.

Luego, se añade una nueva columna `tol_ic` que representa la suma de los indicadores de carencia social (`ic_segsoc`, `ic_ali_nc`, `ic_asalud`, `ic_cv`, `ic_sbv`, `ic_rezedu`). A continuación, se calcula el IPM basado en las siguientes condiciones:
- **Categoría I:** Población en situación de pobreza, donde el ingreso es menor que el umbral de pobreza (`lp`) y el total de carencias sociales (`tol_ic`) es mayor o igual a 1.
- **Categoría II:** Población vulnerable por carencias sociales, donde el ingreso es mayor o igual al umbral de pobreza y el total de carencias sociales es mayor o igual a 1.
- **Categoría III:** Población vulnerable por ingresos, donde el ingreso es menor o igual al umbral de pobreza y el total de carencias sociales es menor a 1.
- **Categoría IV:** Población no pobre multidimensional y no vulnerable, donde el ingreso es mayor o igual al umbral de pobreza y el total de carencias sociales es menor a 1.

Además, se crean dos nuevas variables:
- **`pobre_moderada`:** Indica si una persona está en situación de pobreza moderada, donde el ingreso está entre el límite inferior (`li`) y el límite de pobreza (`lp`), y el total de carencias sociales (`tol_ic`) es mayor a 2.
- **`pobre_extrema`:** Indica si una persona está en situación de pobreza extrema, definida como aquellos en la categoría "I" del IPM y que no cumplen con los criterios de pobreza moderada.

Estas variables proporcionan una visión más detallada sobre los niveles de pobreza y las carencias sociales en la muestra.


``` r
################################################################################
# IPM en la enigh
################################################################################
encuesta_enigh <-
  encuesta_enigh %>% inner_join(LB)  %>% 
  mutate(
    tol_ic = ic_segsoc + ic_ali_nc + ic_asalud + ic_cv +  ic_sbv + ic_rezedu,
    ipm   = case_when(
      # Población en situación de pobreza.
      ingreso < lp  &  tol_ic >= 1 ~ "I",
      # Población vulnerable por carencias sociales.
      ingreso >= lp & tol_ic >= 1 ~ "II",
      # Poblacion vulnerable por ingresos.
      ingreso <= lp & tol_ic < 1 ~ "III",
      # Población no pobre multidimensional y no vulnerable.
      ingreso >= lp & tol_ic < 1 ~ "IV"
    ),
    # Población en situación de pobreza moderada.
    pobre_moderada = ifelse(c(ingreso > li & ingreso < lp) &
                              tol_ic > 2, 1, 0),
    pobre_extrema = ifelse(ipm == "I" & pobre_moderada == 0, 1, 0)
  )
```

#### Preparación de la Muestra Intercensal{-}

En esta sección, se ajusta y se preparan los datos de la muestra ampliada, incorporando las predicciones obtenidas previamente y calculando nuevas variables necesarias para el análisis.

Primero, la muestra ampliada se une con los datos de las líneas de bienestar (`LB`) para asegurar que toda la información relevante esté disponible. A continuación, se ajustan las variables de carencia social en la muestra ampliada para que sean binarias, es decir, se establecen como 1 si el indicador es positivo y 0 en caso contrario. Esto incluye las variables de `ic_asalud`, `ic_cv`, `ic_sbv`, y `ic_rezedu`.

Se añade una nueva variable llamada `tol_ic4`, que es la suma de los indicadores ajustados de carencia social. Esta variable representa el total de carencias sociales en la muestra ampliada.

Además, se incorporan las predicciones de los modelos ajustados, que incluyen:
- `pred_segsoc`: Predicción para el indicador de seguridad social.
- `pred_ingreso`: Predicción para el ingreso.
- `desv_estandar_residual`: Desviación estándar residual utilizada para ajustar el modelo de ingreso.
- `pred_ic_ali_nc`: Predicción para el indicador de alimentación y necesidades básicas.

Finalmente, se elimina el objeto `predicciones` del entorno para liberar memoria, ya que los datos necesarios han sido incorporados en la muestra ampliada. 

Esta preparación asegura que la muestra intercensal esté completa y lista para el análisis posterior.


``` r
################################################################################
# preparando encuesta intercensal
################################################################################

muestra_ampliada <- muestra_ampliada %>% inner_join(LB) 
muestra_ampliada   %<>% mutate(
  ic_asalud = ifelse(ic_asalud == 1, 1,0),
  ic_cv = ifelse(ic_cv == 1, 1,0),
  ic_sbv = ifelse(ic_sbv == 1, 1,0),
  ic_rezedu = ifelse(ic_rezedu == 1, 1,0),
  tol_ic4 = ic_asalud + ic_cv +  ic_sbv + ic_rezedu,
  pred_segsoc = predicciones$pred_segsoc,
  pred_ingreso = predicciones$pred_ingreso,
  desv_estandar_residual = predicciones$desv_estandar_residual,
  pred_ic_ali_nc = predicciones$pred_ic_ali_nc
) 

rm(predicciones)
```

#### Iteración por Entidades Federativas{-}

En esta sección, se realiza un proceso iterativo para ajustar y calibrar la muestra post-calibración en cada entidad federativa. El objetivo es obtener estimaciones precisas a nivel municipal y realizar validaciones. El proceso se repite para cada entidad federal especificada.

**1. Inicialización y Configuración:**

El código comienza con una lista de entidades federativas, representadas por sus códigos (`ii_ent`). Se establece un bucle que itera sobre cada uno de estos códigos. Para cada entidad federal, se imprime la hora de inicio de la iteración y se realiza el filtrado de datos específicos para la entidad actual.

**2. Filtrado y Cálculo de Totales:**

Para la entidad actual (`ii_ent`), se filtra la muestra ampliada (`muestra_ampliada`) para incluir solo los datos correspondientes. Luego, se calcula el total de cada municipio (`total_mpio`) sumando una variable de ponderación (`factor`). También se calcula el total para la entidad (`tot_ent`).

La encuesta de referencia (`encuesta_enigh`) se filtra para la entidad actual y se eliminan las observaciones con valores faltantes (`na.omit`). A partir de esta encuesta, se calcula el total de población (`tot_pob`) por tipo de pobreza multidimensional (`ipm`). Se estima la proporción y el total ajustado para cada categoría de pobreza.

**3. Cálculo de Pobreza:**

Se calculan las tasas de pobreza moderada y extrema utilizando los datos de la encuesta. La pobreza moderada se define como aquellos con ingresos entre los umbrales de pobreza (`li` y `lp`) y con un índice de carencia social mayor que 2. La pobreza extrema se define como aquellos en situación de pobreza multidimensional sin pobreza moderada.

**4. Ajuste y Calibración:**

En cada iteración, se ajusta la muestra post-calibración (`muestra_post`) con las predicciones de los modelos (`pred_segsoc`, `pred_ic_ali_nc`, `pred_ingreso`) y la desviación estándar residual (`desv_estandar_residual`). Se calculan nuevas variables relacionadas con el índice de pobreza multidimensional (IPM) y la pobreza moderada y extrema.

Se crean variables dummy para los municipios y las categorías de IPM y se incorporan al diseño de la encuesta (`diseno_post`). Luego, se realiza la calibración utilizando el método de raking (`calibrate`), ajustando el diseño para que coincida con las estimaciones de la población (`Tx_hat`).

**5. Estimación y Validación:**

Se obtienen las estimaciones calibradas para el IPM y la pobreza a nivel municipal. Estas estimaciones se comparan con los totales calculados previamente para validar la precisión de las predicciones. Se calcula la proporción de cada categoría de IPM y se compara con los totales esperados.

**6. Guardado de Resultados:**

Los resultados de cada iteración se guardan en un archivo `.rds` en una carpeta específica para la entidad y la iteración actual. El archivo incluye:
- `valida`: Resultados de la validación de la pobreza y el IPM.
- `estima_calib`: Estimaciones calibradas para el IPM y la pobreza.

Finalmente, se mide y se imprime el tiempo total que tomó cada iteración. Se libera la memoria utilizada con `gc()` para optimizar el rendimiento durante el proceso iterativo. 

Este procedimiento asegura que las estimaciones sean precisas y que se validen adecuadamente para cada entidad federativa.


``` r
ii_ent <- "02"
iter = 1

for(ii_ent in c("03", "06", "23", "04", "01", "22", "27", "25", "18",
                "05", "17", "28", "10", "26", "09", "32", "08", "19", "29", 
                "24", "11", "31", "13", "16", "12", "14", "15", "07", "21", 
                "30",  "20" ,"02"
)){
  cat("####################################################################\n")
  inicio <- Sys.time()
  print(inicio)

  # Filtrar muestra post
  muestra_post = muestra_ampliada %>% filter(ent == ii_ent)

  # Cálculo de totales por municipio y población
  total_mpio  <- muestra_post %>% group_by(cve_mun) %>% 
    summarise(den_mpio = sum(factor), .groups = "drop") %>% 
    mutate(tot_ent = sum(den_mpio))

  encuesta_sta = encuesta_enigh %>% filter(ent == ii_ent)
  encuesta_sta %<>% na.omit()

  tot_pob <- encuesta_sta %>% 
    group_by(ipm) %>% 
    summarise(num_ent = sum(fep), .groups = "drop") %>% 
    transmute(ipm, prop_ipm = num_ent/sum(num_ent),
              tx_ipm =  sum(total_mpio$den_mpio)*prop_ipm) %>% 
    filter(ipm != "I")

  pobreza <- encuesta_sta %>% summarise(
    pobre_ext = weighted.mean(pobre_extrema, fep),
    pob_mod = weighted.mean(pobre_moderada, fep),
    pobre_moderada = sum(total_mpio$den_mpio)*pob_mod,
    pobre_extrema = sum(total_mpio$den_mpio)*pobre_ext)

  Tx_ipm <- setNames(tot_pob$tx_ipm,paste0("ipm_",tot_pob$ipm))
  Tx_pob <- pobreza[,3:4] %>% as.vector() %>% unlist()

  tx_mun <- setNames(total_mpio$den_mpio,
                     paste0("cve_mun_",total_mpio$cve_mun))

  Tx_hat <- c(Tx_pob, Tx_ipm, tx_mun)

  for(iter in 1:200){
    cat("\n municipio = ", ii_ent,"\n\n")
    cat("\n iteracion = ", iter,"\n\n")

    # Ajuste de la muestra post
    muestra_post %<>% 
      mutate(
        ic_segsoc = rbinom(n = n(), size = 1, prob = pred_segsoc),
        ic_ali_nc = rbinom(n = n(), size = 1, prob = pred_ic_ali_nc),
        ingreso = pred_ingreso + rnorm(n = n(), mean = 0, desv_estandar_residual),
        tol_ic = tol_ic4 + ic_segsoc + ic_ali_nc
      ) %>% mutate(
        ipm   = case_when(
          # Población en situación de pobreza.
          ingreso < lp  &  tol_ic >= 1 ~ "I",
          # Población vulnerable por carencias sociales.
          ingreso >= lp & tol_ic >= 1 ~ "II",
          # Poblacion vulnerable por ingresos.
          ingreso <= lp & tol_ic < 1 ~ "III",
          # Población no pobre multidimensional y no vulnerable.
          ingreso >= lp & tol_ic < 1 ~ "IV"
        ),
        # Población en situación de pobreza moderada.
        pobre_moderada = ifelse(c(ingreso > li & ingreso < lp) &
                                  tol_ic > 2, 1, 0),
        pobre_extrema = ifelse(ipm == "I" & pobre_moderada == 0, 1, 0))  

    Xk <- muestra_post %>% select("ipm", "cve_mun") %>% 
      fastDummies::dummy_columns(select_columns = c("ipm", "cve_mun")) %>% 
      select(names(Tx_hat[-c(1:2)]))

    diseno_post <- bind_cols(muestra_post, Xk) %>% 
      mutate(fep = factor) %>%
      as_survey_design(
        ids = upm,
        weights = fep,
        nest = TRUE
      )

    mod_calib <-  as.formula(paste0("~ -1+",paste0(names(Tx_hat), collapse = " + ")))

    diseno_calib <- calibrate(diseno_post, formula = mod_calib, 
                              population = Tx_hat, calfun = "raking")

    estima_calib_ipm  <- diseno_calib %>% group_by(cve_mun, ipm) %>% 
      summarise(est_ipm = survey_mean(vartype ="var"))

    estima_calib_pob  <- diseno_calib %>% group_by(cve_mun) %>% 
      summarise(est_pob_ext = survey_mean(pobre_extrema , vartype ="var"),
                est_pob_mod = survey_mean(pobre_moderada  , vartype ="var" ))

    estima_calib <- pivot_wider(
      data = estima_calib_ipm,
      id_cols = "cve_mun",
      names_from = "ipm",
      values_from = c("est_ipm", "est_ipm_var"), values_fill = 0
    ) %>% full_join(estima_calib_pob)

    valida_ipm <- estima_calib_ipm %>% inner_join(total_mpio, by = "cve_mun") %>% 
      group_by(ipm) %>% 
      summarise(prop_ampliada = sum(est_ipm*den_mpio)/unique(tot_ent)) %>% 
      full_join(tot_pob, by = "ipm")

    valida_pob <- estima_calib_pob %>% select(cve_mun, est_pob_ext, est_pob_mod) %>% 
      inner_join(total_mpio, by = "cve_mun") %>%
      summarise(pob_ext_ampliada = sum(est_pob_ext * den_mpio) / unique(tot_ent),
                pob_mod_ampliada = sum(est_pob_mod * den_mpio) / unique(tot_ent),
                ipm_I = pob_ext_ampliada + pob_mod_ampliada) %>% 
      bind_cols(pobreza[,1:2])

    saveRDS(list(valida = list(valida_pob = valida_pob, 
                               valida_ipm = valida_ipm),
                 estima_calib = estima_calib),
            file = paste0( "../output/2020/iteraciones/mpio_calib/",
                           ii_ent,"/iter",iter,".rds"))
    gc()
  }
  fin <- Sys.time()
  tiempo_total <- difftime(fin, inicio, units = "mins")
  print(tiempo_total)
  cat("####################################################################\n")
}
```
