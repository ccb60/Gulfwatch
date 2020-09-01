Extract Data on PCB Contamination in Mussel Tissue from Casco Bay Sites
From Gulfwatch Data
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
8/20/2020

  - [Overview](#overview)
      - [Data Reorganization in Excel](#data-reorganization-in-excel)
  - [Load Libraries](#load-libraries)
  - [Site Codes](#site-codes)
  - [Load Data](#load-data)
      - [Convert to Long Form](#convert-to-long-form)
      - [Split Columns](#split-columns)
      - [Save Result](#save-result)

<img
  src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
  style="position:absolute;top:10px;right:50px;" />

# Overview

  - PCB files are organized with Parameters (PCBs) in rows, and Sites in
    columns.
  - As for the other Gulfwatch data, files prior to 1998 are organized
    in blocks, not one continuous table.
  - PCBs are identified by congener number, sometimes separated by a
    semicolon when more than one congener is reported on a given line.
    Nomenclature and a crosswalk to other ways of naming compounds are
    provided in the Excel file “PCB\_Nomenclature.xlsx”. As our primary
    focus will be on TOTAL PCBs, nomenclature is not critical.  
  - All sample values are in ng/g dry weight.  
  - Non-detects are abundant, and are shown as “\<{value}”.  
  - Each block of sample data is followed by a related block containing
    data on “Surrogate Recovery” expressed in percent. Recoveries vary
    substantially.  
  - Nomenclature used in the Surrogate Recovery blocks is not consistent
    across years or even with sample data from the same year.  
  - Surrogate recovery data are sometimes encoded with, and sometimes
    without a percent sign (“%”).
  - Although four samples were analyzed from the Royal River in 1993 for
    metals, only three were analyzed for PCBs.
  - A sample from Broad Cove in 1995 appears to have two sets of values
    combined in a a single column, separated by a slash (“/”). We split
    the two values out as replicates. We do not know if these are field
    replicates or laboratory replicates.  
  - A duplicate sample in 1999 was not assigned to a specific sample. By
    context that appears to be a laboratory duplicate (but that is not
    clearly defined). There is no indication of which primary sample it
    duplicates (if any). This makes it impossible to fully assess data
    consistency using this sample, and figure out how to handle it in
    analysis.
  - The PCB data for 2000 included two rows listed as PCB 87. By
    context, the second one Probably should have been PCB 187.
  - Percent lipid data was provided for 1999 data only, and is dropped
    here for consistency with other years.
  - The percent recovery figure for PCB 198 in 1999 is listed as “IN”
    for interference.

## Data Reorganization in Excel

With all those exceptions and considerations, it was simpler to
reorganize these data by hand in Excel, rather than try to write code to
extract data in a readily reproducible way.

Reorganization involved: 1. pooling data from multiple files; 2.
deleting data from sampling locations outside of Casco Bay; 3.
establishing consistent ordering of PCBs 4. adding sample identification
data ( site code, year, and sample type); and 5. Revising the numeric
PCB Codes to generate syntactic names for R. Names start with “PCB\_”
followed by one or two PCB congener numbers, separated by underscores.

We retained both sample analysis data and data on “Surrogate Recovery”
for each sample, but in separate files to simplify analysis.

The reorganized sample data is contained in ‘gulfwatch\_pcb\_Casco.csv’
and the percent recovery data is in
‘gulfwatch\_pcb\_Casco\_pct\_recovery.csv’.

These data require further processing in R, especially to handle
non-detects. We begin that processing here, and continue it in the
accompanying Notebook “Calculate\_Gulfwatch\_PCB\_Totals.Rmd”.

Following CBEP convention, we split data columns containing non-detects
into two data columns, one containing numeric values (either sample
observations or detection limits), and the other containing a logical
flag that indicates which values are non-detects.

# Load Libraries

``` r
library(tidyverse)
```

    ## -- Attaching packages --------------------------------------------------------------------------------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.2     v purrr   0.3.4
    ## v tibble  3.0.3     v dplyr   1.0.0
    ## v tidyr   1.1.0     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.5.0

    ## -- Conflicts ------------------------------------------------------------------------------------------------------------------ tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

# Site Codes

Data in each file indicates locations of samples either with a site code
(most data) or with a site name (most metals data).

``` r
sitecodes <- c('MEPH', 'MEPR', 'MEBC', 'MERY')
sitenames <- c('Portland Harbor', 'Presumpscot River',
               'Broad Cove', 'Royal River')
```

# Load Data

``` r
fn     <- 'gulfwatch_pcb_Casco.csv'
pcb_data <- read_csv(fn)
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_character(),
    ##   Year = col_double(),
    ##   Dups = col_logical()
    ## )

    ## See spec(...) for full column specifications.

## Convert to Long Form

``` r
pcb_data_longer <- pcb_data %>% pivot_longer(cols = PCB_8_5:Total,
                                             names_to = 'PCB',
                                             values_to = 'Original')
```

## Split Columns

Into the numerical value and a ND flag

``` r
pcb_data_working <- pcb_data_longer %>%
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
write_csv(pcb_data_working, 'gulfwatch_pcb_data_long.csv')
```
