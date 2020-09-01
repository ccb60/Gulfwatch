Extract Data on Metal CoOntamination in Mussel Tissue from Casco Bay
Sites From Gulfwatch Data
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
8/20/2020

  - [Overview](#overview)
      - [Strategy](#strategy)
  - [Load Libraries](#load-libraries)
  - [Site Codes](#site-codes)
  - [Load Data](#load-data)
      - [Establish Folder Reference](#establish-folder-reference)
      - [Filenames](#filenames)
      - [Other Files](#other-files)
      - [Combine Files](#combine-files)
  - [Save Results](#save-results)

<img
  src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
  style="position:absolute;top:10px;right:50px;" />

# Overview

1.  Metals data is divided into separate files by year.
2.  Files are organized with samples in rows, and parameters in columns.
3.  Data is often organized into multiple blocks within each file,
    rather than in one large table.
4.  Headers appear consistent between blocks within years, but not
    between years. Most data columns appear in all years, but not always
    in the same order. One extra data column (“Days Deployed”) was
    included in 1992, but not in subsequent years.
5.  Column names have inconsistent capitalization, and even some
    extraneous spaces.
6.  Some years contain additional data that LACKS headers, generally
    containing supplementary information like sampling dates or sample
    ID numbers that are not provided for most samples and years, and
    therefore can not be analyzed.
7.  The data appears to use different conventions for reporting
    non-detects from year to year. In most cases, non-detects are
    represented by “ND”, followed by a value. It is not clear whether
    that value represents a detection limit or not. NDs at Casco Bay
    sites were only seen for silver (Ag).
8.  In 2000, the data switches from using Site Names to using Site Codes
    to specify where samples were collected.
9.  The Site Name “Broad Cove” can refer either to a site in Casco Bay
    or a Site in Nova Scotia.

The 1992 data includes the following annotations, which appear to apply
to all years data:  
\> ND = Non Detect  
\> K = less than  
\> Sample: the letter indicates the type of sample; c=caged, P=preset;
and N=indigenous.  
\> Sample: the \# refers to the number of composites (20
mussels/composite) per station.

## Strategy

The best strategy is to read in data for each Year, and join them
together with “bind\_rows”, which respects column names.

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

## Filenames

``` r
metalfns <- list.files(niece, pattern = '^metal')
(metalfns <- metalfns[c(2:7,1)])
```

    ## [1] "metal92.txt" "metal93.txt" "metal94.txt" "metal95.txt" "metal98.txt"
    ## [6] "metal99.txt" "metal00.txt"

We could read all files in using lapply, or something similar, but the
file specific formatting makes that more difficult. A Fore loop makes
the logic easier to follow. Even so, the complex file format means we
have lots of parsing errors. luckily, they don’t affect the data we
want. \#\# First data file Because 1992 data has a different number of
data columns, we read it in first, then read the others.

``` r
afn <- 'metal92.txt'
apath <- file.path(niece, afn)
start <- read_tsv(apath, skip=4,
                  col_types = cols(col_integer(), col_character(),
                                col_character(), col_integer(),
                                col_character(), col_double(),
                                col_double(), col_double(),
                                col_double(), col_double(),
                                col_double(), col_double(),
                                col_double(), col_double(),
                                col_double()  )) %>%
  filter(Station %in% sitenames | Station %in% sitecodes ) %>%
  select (-Days) %>%
  rename(solid = `% Solids`) %>%
  rename_all(tolower) %>%
  filter(row_number(station)<4)    # Eliminate Nova Scotia Broad Cove locations
```

    ## Warning: 66 parsing failures.
    ## row  col   expected   actual                                                                                       file
    ##   1 Days an integer Deployed 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal92.txt'
    ##  16 Year an integer Year     'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal92.txt'
    ##  16 Days an integer Days     'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal92.txt'
    ##  16 Cd   a double   Cd       'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal92.txt'
    ##  16 Cr   a double   Cr       'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal92.txt'
    ## ... .... .......... ........ ..........................................................................................
    ## See problems(...) for more details.

## Other Files

It turns out, we either have Maine or Nova Scotia “Broad Cove” sites,
but never both in subsequent years, so we either save Broad Cove, or
not. We don’t want Broad Cove in 1993 or 1999.

``` r
out <- list(Yr92=start)
remainingmetalfns <- metalfns[2:7]

skipno <- c(2,4,4,2,2,2)     # Number of blank lines a top of files, in order
BCflag <- c(FALSE, TRUE, TRUE, TRUE, FALSE, TRUE)  # Do we want Broad Cove?

for (index in seq_along(remainingmetalfns)) {
  fnm <- remainingmetalfns[index]
  nm <- paste0('Yr', substr(fnm,6,7))
  reduced_sitenames <- sitenames[c(1,2,4)]

  apath <- file.path(niece, fnm)
  
  dat <- read_tsv(apath, skip=skipno[index],
                     col_types = cols(col_integer(), col_character(),
                                   col_character(), col_character(),
                                   col_double(), col_double(),
                                   col_double(), col_double(),
                                   col_double(), col_double(),
                                   col_double(), col_double(),
                                   col_double(), col_double())) %>%

    rename_all(tolower) %>%
    rename_all(~sub('% ', '', .)) %>%
    rename_all(trimws) %>%
    rename_all(~sub('solids', 'solid', .))


  if(BCflag[index] ) {
    dat <- dat %>%
    filter(station %in% sitenames | station %in% sitecodes)
    cat('\n', nm, '\n')
  }
  else {
    dat <- dat %>%
    filter(station %in% reduced_sitenames | station %in% sitecodes )
    cat('\n', '     ', nm , '\n')
  }

  
  out[[nm]] <- dat
}
```

    ## Warning: Missing column names filled in: 'X15' [15]

    ## Warning: Unnamed `col_types` should have the same length as `col_names`. Using
    ## smaller of the two.

    ## Warning: 125 parsing failures.
    ## row col   expected     actual                                                                                       file
    ##   1  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal93.txt'
    ##   2  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal93.txt'
    ##   3  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal93.txt'
    ##   4  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal93.txt'
    ##   5  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal93.txt'
    ## ... ... .......... .......... ..........................................................................................
    ## See problems(...) for more details.

    ## 
    ##        Yr93

    ## Warning: Missing column names filled in: 'X15' [15]

    ## Warning: Unnamed `col_types` should have the same length as `col_names`. Using
    ## smaller of the two.

    ## Warning: 185 parsing failures.
    ## row col   expected     actual                                                                                       file
    ##   1  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal94.txt'
    ##   2  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal94.txt'
    ##   3  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal94.txt'
    ##   4  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal94.txt'
    ##   5  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal94.txt'
    ## ... ... .......... .......... ..........................................................................................
    ## See problems(...) for more details.

    ## 
    ##  Yr94

    ## Warning: 52 parsing failures.
    ## row  col   expected        actual                                                                                       file
    ##  22 CD   a double   New Hampshire 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal95.txt'
    ##  23 Year an integer Year          'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal95.txt'
    ##  23 CD   a double   CD            'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal95.txt'
    ##  23 CR   a double   CR            'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal95.txt'
    ##  23 CU   a double   CU            'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal95.txt'
    ## ... .... .......... ............. ..........................................................................................
    ## See problems(...) for more details.

    ## 
    ##  Yr95

    ## Warning: 52 parsing failures.
    ## row  col   expected actual                                                                                       file
    ##   3 NI   a double   ND 0.8 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal98.txt'
    ##   4 NI   a double   ND 0.8 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal98.txt'
    ##   5 NI   a double   ND 0.8 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal98.txt'
    ##  16 Year an integer Year   'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal98.txt'
    ##  16 AL   a double   AL     'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal98.txt'
    ## ... .... .......... ...... ..........................................................................................
    ## See problems(...) for more details.

    ## 
    ##  Yr98

    ## Warning: 45 parsing failures.
    ## row  col   expected actual                                                                                       file
    ##  26 Year an integer   Year 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal99.txt'
    ##  26 AL   a double     AL   'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal99.txt'
    ##  26 CD   a double     CD   'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal99.txt'
    ##  26 CR   a double     CR   'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal99.txt'
    ##  26 CU   a double     CU   'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal99.txt'
    ## ... .... .......... ...... ..........................................................................................
    ## See problems(...) for more details.

    ## 
    ##        Yr99

    ## Warning: Missing column names filled in: 'X15' [15]

    ## Warning: Unnamed `col_types` should have the same length as `col_names`. Using
    ## smaller of the two.

    ## Warning: 189 parsing failures.
    ## row col   expected     actual                                                                                       file
    ##   1  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal00.txt'
    ##   2  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal00.txt'
    ##   3  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal00.txt'
    ##   4  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal00.txt'
    ##   5  -- 14 columns 15 columns 'C:/Users/curtis.bohlen/Dropbox/Work/Shellfish_Toxics/Original_Data/Gulfwatch/metal00.txt'
    ## ... ... .......... .......... ..........................................................................................
    ## See problems(...) for more details.

    ## 
    ##  Yr00

## Combine Files

``` r
metals_data <- bind_rows(out) %>%
  mutate(agflag = grepl('ND', ag)) %>%
  mutate(ag_rev = if_else(agflag, as.numeric(substr(ag, 3,nchar(ag))), as.numeric(ag))) 
```

    ## Warning in replace_with(out, !condition, false, fmt_args(~false), glue("length
    ## of {fmt_args(~condition)}")): NAs introduced by coercion

# Save Results

``` r
write_csv(metals_data, 'gulfwatch_metals.csv')
```
