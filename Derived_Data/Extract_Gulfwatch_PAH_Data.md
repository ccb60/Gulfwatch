Extract Data on PAH Contamination in Mussel Tissue from Casco Bay Sites
From Gulfwatch Data
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
8/20/2020

  - [Overview](#overview)
      - [Data Reorganization in Excel](#data-reorganization-in-excel)
  - [Load Libraries](#load-libraries)
  - [Site Codes](#site-codes)
      - [REGEX patterns](#regex-patterns)
  - [Load Data](#load-data)
      - [Convert to Long Form](#convert-to-long-form)
      - [Split Columns](#split-columns)
      - [Save Result](#save-result)

<img
  src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
  style="position:absolute;top:10px;right:50px;" />

# Overview

  - PAH files are organized with Parameters (PAHs) in rows, and Sites in
    columns.

  - As for the Metals, data prior to 1998 are organized in blocks, not
    one continuous table.  

  - Several compounds are written in slightly different ways in
    different years. THis mostly represents differences in
    capitalization and presence or absence of spacing or punctuation.

  - All sample values are in ng/g dry weight.

  - Non-detects are abundant, and are shown as “\<{value}”.

  - Each block of sample data is followed by a related block containing
    data on “Surrogate Recovery” expressed in percent. Recoveries vary
    substantially.

  - Nomenclature used in the Surrogate Recovery blocks is not consistent
    across years or even with sample data from the same year.

  - Surrogate recovery data are sometimes encoded with, and sometimes
    without a percent sign (“%”).

  - File ‘pah93.txt’ contains internal labels that suggest the file
    contains 1994 data, but the data is not the same as in ‘pah94.txt’.
    Data is shown for the Royal River, which was sampled in 1993, but
    not in 1994, so we believe the internal label is incorrect, and the
    table contains data from 1993.

  - Although four samples were analyzed from the Royal River in 1993 for
    metals, only three were analyzed for PAHs.

  - A sample from Broad Cove in 1995 appears to have two sets of values
    combined in a a single column, separated by a slash (“/”). We split
    the two values out as replicates. We do not know if these are field
    replicates or laboratory replicates.

## Data Reorganization in Excel

With all those exceptions and considerations, it was simpler to
reorganize these data by hand in Excel, rather than try to write code to
extract data in a readily reproducible way.

Reorganization involved: 1. pooling data from multiple files; 2.
deleting data from sampling locations outside of Casco Bay; 3.
establishing consistent PAH nomenclature (resolving spelling and
capitalization differences); 4. adding sample identification data ( site
code, year, and sample type); and 5. transposing the data, so samples
are on rows and parameters are on columns.

We retained both sample analysis data and data on “Surrogate Recovery”
for each sample, but in separate files to simplify analysis.

The reorganized sample data is contained in ‘gulfwatch\_pah\_Casco.csv’
and the percent recovery data is in
‘gulfwatch\_pah\_Casco\_pct\_recovery.csv’.

These data require further processing in R, especially to handle
non-detects. We begin that processing here, and continue it in the
accompanying Notebook “Calculate\_Gulfwatch\_PAH\_Totals.Rmd”.

Following CBEP convention, we split data columns containing non-detects
into two data columns, one containing numeric values (either sample
observations or detection limits), and the other containing a logical
flag that indicates which values are non-detects.

# Load Libraries

``` r
library(tidyverse)
```

    ## -- Attaching packages --------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.2     v purrr   0.3.4
    ## v tibble  3.0.3     v dplyr   1.0.0
    ## v tidyr   1.1.0     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.5.0

    ## -- Conflicts ------------------------------------------ tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())

library(LCensMeans)
```

# Site Codes

Data in each file indicates locations of samples either with a site code
(most data) or with a site name (most metals data).

``` r
sitecodes <- c('MEPH', 'MEPR', 'MEBC', 'MERY')
sitenames <- c('Portland Harbor', 'Presumpscot River',
               'Broad Cove', 'Royal River')
```

## REGEX patterns

``` r
sitecodepattern = ''
sitecodepattern <- paste0(sitecodepattern, paste(sitecodes, collapse = '|' ))
sitecodepattern
```

    ## [1] "MEPH|MEPR|MEBC|MERY"

# Load Data

``` r
fn     <- 'gulfwatch_pah_Casco.csv'
pah_data <- read_csv(fn)
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_character(),
    ##   Year = col_double()
    ## )

    ## See spec(...) for full column specifications.

## Convert to Long Form

``` r
pah_data_longer <- pah_data %>% pivot_longer(cols = Naphthalene:Total,
                                             names_to = 'PAH',
                                             values_to = 'Original')
```

## Split Columns

Into the numerical value and a ND flag

``` r
pah_data_working <- pah_data_longer %>%
  mutate(Flag = Original == 'ND' | grepl('<', Original),
         Concentration = if_else(Flag,
                                 as.numeric(substr(Original, 2,nchar(Original))),
                                 as.numeric(Original))) %>%
  select (-Original)
```

    ## Warning in if_else(Flag, as.numeric(substr(Original, 2, nchar(Original))), : NAs
    ## introduced by coercion

    ## Warning in replace_with(out, !condition, false, fmt_args(~false), glue("length
    ## of {fmt_args(~condition)}")): NAs introduced by coercion

## Save Result

``` r
write_csv(pah_data_working, 'gulfwatch_pah_data_long.csv')
```
