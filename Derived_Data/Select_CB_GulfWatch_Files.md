Identify Files Containing Data on Casco Bay Sites From Gulfwatch Data
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
8/20/2020

  - [Introduction](#introduction)
  - [Site Codes](#site-codes)
  - [Load Data](#load-data)
      - [Establish Folder Reference](#establish-folder-reference)
  - [Search Files to Find Casco Bay
    Data](#search-files-to-find-casco-bay-data)
      - [Assemble REGEX Patterns](#assemble-regex-patterns)
      - [Identify Files we Need](#identify-files-we-need)
      - [Move Files We Don’t Need](#move-files-we-dont-need)

\<img
src=“<https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg>”

# Introduction

The Gulfwatch data was downloaded in annual files, but data from Casc
oBay was not collected in every year. This preliminary script identifies
which files contain Casco Bay related data and moves all files that do
not contain Casco Bay data into a sub-folder. This simplified access to
files in the data access scripts.

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

# Site Codes

Data in each file indicates locations of samples either with a site code
(most data) or with a site name (most metals data).

``` r
sitecodes <- c('MEPH', 'MEPR', 'MEBC', 'MERY')
sitenames <- c('Portland Harbor', 'Presumpscot River',
               'Broad Cove', 'Royal River')
```

# Load Data

## Establish Folder Reference

``` r
sibfldnm <- 'Original_Data'
parent   <- dirname(getwd())
sibling  <- file.path(parent,sibfldnm)
niecefn = 'Gulfwatch'
niece = file.path(sibling, niecefn)
```

``` r
(fnlist <- list.files(niece, pattern = '^(metal|pah|pcb|pest)'))
```

    ##  [1] "metal00.txt" "metal91.txt" "metal92.txt" "metal93.txt" "metal94.txt"
    ##  [6] "metal95.txt" "metal96.txt" "metal97.txt" "metal98.txt" "metal99.txt"
    ## [11] "pah00.txt"   "pah92.txt"   "pah93.txt"   "pah94.txt"   "pah95.txt"  
    ## [16] "pah96.txt"   "pah97.txt"   "pah98.txt"   "pah99.txt"   "pcb00.txt"  
    ## [21] "pcb92.txt"   "pcb93.txt"   "pcb94.txt"   "pcb95.txt"   "pcb96.txt"  
    ## [26] "pcb97.txt"   "pcb98.txt"   "pcb99.txt"   "pest00.txt"  "pest92.txt" 
    ## [31] "pest93.txt"  "pest94.txt"  "pest95.txt"  "pest96.txt"  "pest97.txt" 
    ## [36] "pest98.txt"  "pest99.txt"

# Search Files to Find Casco Bay Data

## Assemble REGEX Patterns

``` r
sitecodepattern = ''
sitecodepattern <- paste0(sitecodepattern, paste(sitecodes, collapse = '|' ))
sitecodepattern
```

    ## [1] "MEPH|MEPR|MEBC|MERY"

``` r
sitenamepattern = ''
sitenamepattern <- paste0(sitenamepattern, paste(sitenames, collapse = '|' ))
sitenamepattern
```

    ## [1] "Portland Harbor|Presumpscot River|Broad Cove|Royal River"

## Identify Files we Need

The following could probably be managed with functional programming /
vectorized code instead of for loops. It makes little difference here,
since the time consuming steps are opening files and searching.

Also, note that the list of files that do NOT contain Casco Bay data is
empty after this notebook is run for the first time.

``` r
res <- logical(length(fnlist))
for (index in seq_along(fnlist)) {
    lns <- read_lines(file.path(niece,fnlist[index]))
    res[index] <- (any(grepl(sitecodepattern, lns)) | any(grepl(sitenamepattern, lns)))
}
(files_with_CB_data <- fnlist[res])
```

    ##  [1] "metal00.txt" "metal92.txt" "metal93.txt" "metal94.txt" "metal95.txt"
    ##  [6] "metal98.txt" "metal99.txt" "pah00.txt"   "pah93.txt"   "pah94.txt"  
    ## [11] "pah95.txt"   "pah99.txt"   "pcb00.txt"   "pcb93.txt"   "pcb94.txt"  
    ## [16] "pcb95.txt"   "pcb99.txt"   "pest00.txt"  "pest93.txt"  "pest94.txt" 
    ## [21] "pest95.txt"  "pest99.txt"

``` r
cat('\n\n')
```

``` r
(files_wo_CB_data <- fnlist[! res])
```

    ##  [1] "metal91.txt" "metal96.txt" "metal97.txt" "pah92.txt"   "pah96.txt"  
    ##  [6] "pah97.txt"   "pah98.txt"   "pcb92.txt"   "pcb96.txt"   "pcb97.txt"  
    ## [11] "pcb98.txt"   "pest92.txt"  "pest96.txt"  "pest97.txt"  "pest98.txt"

## Move Files We Don’t Need

We place the data files that do not contain Casco Bay Data into the
“Unused” folder.

``` r
from <- file.path(niece, files_wo_CB_data)
to   <- file.path(niece, 'Unused', files_wo_CB_data)
fs::file_move(from, to)
```
