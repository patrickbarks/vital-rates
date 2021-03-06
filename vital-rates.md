Deriving vital rates for moving window analysis
================
Patrick Barks
2019-01-30

#### Preliminaries

``` r
# devtools::install_github("jonesor/Rcompadre")
# devtools::install_github("patrickbarks/Rage", ref = "devel")

library(tidyverse)
library(Rcompadre)
library(Rage)
library(popbio)
```

#### Prepare data from COMPADRE

``` r
# load compadre db
comp <- cdb_fetch("~/COMPADRE_v.X.X.X.RData")

# species of interest
spp <- c("Sphaeralcea_coccinea", "Psoralea_tenuiflora", "Paronychia_jamesii",
         "Echinacea_angustifolia", "Solidago_mollis", "Cirsium_undulatum",
         "Thelesperma_megapotamicum", "Ratibida_columnifera",
         "Hedyotis_nigricans", "Lesquerella_ovalifolia", "Cirsium_pitcheri_8",
         "Dicerandra_frutescens", "Silene_spaldingii", "Cirsium_pitcheri_4",
         "Eryngium_alpinum", "Haplopappus_radiatus", "Astragalus_tyghensis",
         "Brassica_insularis", "Helianthemum_juliae", "Kosteletzkya_pentacarpos")

# filter to individual annual mpms from species of interest
comp_spp <- comp %>% 
  filter(MatrixComposite == "Individual",
         MatrixTreatment == "Unmanipulated",
         SpeciesAuthor %in% spp) %>% 
  cdb_flag(c("check_ergodic", "check_zero_U")) %>%
  filter(check_zero_U == FALSE) %>% # one matrix all 0s
  cdb_unnest() %>% 
  # identify propagule/dormant stage classes
  mutate(prop = map(MatrixClassOrganized, ~ .x == "prop"),
         dorm = map(MatrixClassOrganized, ~ .x == "dorm"),
         prop_or_dorm = map(MatrixClassOrganized, ~ .x %in% c("prop", "dorm")))
```

#### Collape to mean MPM by species

``` r
# collapse to mean matrix by species to calculate stable distribution and set of
#  possible transitions
comp_mean <- comp_spp %>% 
  cdb_collapse("SpeciesAuthor") %>% 
  cdb_unnest() %>% 
  mutate(stable_stage = map(matA, popbio::stable.stage),
         posU = map(matU, ~ .x > 0),
         posF = map(matF, ~ .x > 0)) %>%
  CompadreData() %>% 
  select(SpeciesAuthor, stable_stage, posU, posF)
```

#### Derive lambda, survival, and growth conditional on survival

``` r
# calculate lambda, mean survival, and mean growth conditional on survival
comp_vitals <- comp_spp %>% 
  left_join(comp_mean, by = "SpeciesAuthor") %>% 
  mutate(lambda = map_dbl(matA, popbio::lambda)) %>% 
  mutate(survival = mapply(vr_scalar_survival,
                           matU = matU,
                           posU = posU,
                           # weights_col = stable_stage, # weight by stable stage
                           exclude_col = prop_or_dorm)) %>% 
  mutate(growth = mapply(vr_scalar_growth,
                         matU = matU,
                         posU = posU,
                         exclude_row = dorm,
                         exclude_col = prop_or_dorm)) %>% 
  mutate(SpeciesAuthor = reorder(SpeciesAuthor,
                                 MatrixStartYear,
                                 function(x) length(unique(x)))) %>% 
  mutate(year = as.Date(paste0(MatrixStartYear, "-01-01")))
```

#### Check plots

``` r
# plot survival over time, by species
ggplot(comp_vitals, aes(year, survival)) +
  geom_point() +
  scale_x_date(date_labels = "%y") +
  facet_wrap(~ SpeciesAuthor, scales = "free_x")
```

![](vital-rates_files/figure-markdown_github/unnamed-chunk-5-1.png)

``` r
# plot growth over time, by species
ggplot(filter(comp_vitals, !is.na(growth)), aes(year, growth)) +
  geom_point() +
  scale_x_date(date_labels = "%y") +
  facet_wrap(~ SpeciesAuthor, scales = "free_x")
```

![](vital-rates_files/figure-markdown_github/unnamed-chunk-5-2.png)

``` r
# plot lambda over time, by species
ggplot(comp_vitals, aes(year, lambda)) +
  geom_point() +
  scale_x_date(date_labels = "%y") +
  scale_y_log10() +
  coord_cartesian(ylim = c(0.1, 8)) +
  facet_wrap(~ SpeciesAuthor, scales = "free_x")
```

![](vital-rates_files/figure-markdown_github/unnamed-chunk-5-3.png)
