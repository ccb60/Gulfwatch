Recalculate PAH Totals From Gulfwatch, Using different ND Conventions
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
8/20/2020

  - [Introduction](#introduction)
      - [Tasks](#tasks)
          - [Benzo(b+k)fluoranthene](#benzobkfluoranthene)
          - [Totals](#totals)
  - [Load Data](#load-data)
  - [Addressing Toxicity in Edible
    Tissues](#addressing-toxicity-in-edible-tissues)

<img
  src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
  style="position:absolute;top:10px;right:50px;" />

``` r
library(tidyverse)
```

    ## -- Attaching packages ---------------------------------------------------------------------------------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.2     v purrr   0.3.4
    ## v tibble  3.0.3     v dplyr   1.0.0
    ## v tidyr   1.1.0     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.5.0

    ## -- Conflicts ------------------------------------------------------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(CBEPgraphics)
load_cbep_fonts()
theme_set
```

    ## function (new) 
    ## {
    ##     old <- ggplot_global$theme_current
    ##     ggplot_global$theme_current <- new
    ##     invisible(old)
    ## }
    ## <bytecode: 0x00000000190c7630>
    ## <environment: namespace:ggplot2>

``` r
library(LCensMeans)
```

# Introduction

The Gulfwatch PAH data includes many non-detects. It is important to
analyze these non-detects in a consistent manner. There are three common
(non-statistical) strategies for addressing non-detects, and one more
statistically-based approach. Those four approaches are: 1. Replace all
non-detects with the detection limit 2. Replace all non-detects with
HALF the detection limit 3. Replace all non-detects with ZERO 4. Replace
all non-detects with Maximum Likelihood estimates of conditional means.
5. Kaplan-Meier (Non-parametric) method 6. Regression on Order
Statistics

Approach \# 4 through \# 6 are statistically grounded. Of those, \#4 is
conceptually the most straight forward, but it requires assumptions
about the underlying probability distribution of unobserved data.

All the statistical methods are probably inappropriate when there are
few observed values (fewer than perhaps 20% of observations).

We calculate estimated conditional means using \# 4, via our LCensMean
package. We make the assumption that data for many contaminants are
distributed approximately lognormal.

The “Totals” Reported in the raw Gulfwatch data implicitly use strategy
3. Non-detects are not included in those totals at all.

## Tasks

We need to calculate totals by site, and also address inconsistencies in
reporting Benzo(b)fluoranthene and Benzo(k)fluoranthene.

### Benzo(b+k)fluoranthene

Benzo(b)fluoranthene and Benzo(k)fluoranthene are sometimes (1999 and
2000) reported separately and sometimes (1993, 1994, 1995) reported as a
sum. We only get detection limits implicitly, for the non-detect
samples, and non-detects did not happen in all years. But other
detection limits are consistent from 1993-1995, and separately for 1999
and 2000. If that is again the case, detection limits were as follows:

Chemical Constituent | NDs Were Reported | Detection Limit (ng/g)
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_|\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_|\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
Benzo(b+k)fluoranthene | 1993 and 1995 | \< 10 Benzo(b) fluoranthene |
1999 | \< 8 Benzo(k) fluoranthene | 1999 | \< 2

For consistency, we want to estimate the sum of Benzo(b+k) fluoranthene
for the latter two years. That is easy to do when both compounds were
detected, as for all observations in 2000, but it is far from obvious
how to do that correctly when one or both compounds were not detected.

It turns out that the detection limit in 1999 and 2000 for Benzo(b)
fluoranthene was relatively high, at 8 ng/l, and the compound was never
detected, while Benzo(k) fluoranthene had a lower DL, at 2 ng/l, and it
was detected several times.

Best use is probably to formally use ML methods, but impact is going to
be small.

The different methods here can only matter if we look at individual
PAHs, which we do not now plan to do. Also, impact is likely to be
small.

### Totals

We calculate values using methods 1 through 4 to see how they differ.

We prefer to go with method 4 where the data is sufficient for that to
make sense. The reason is that this approach is based on solid
statistical logic, and makes use not only of the data on observed
values, but also data on the proportion of all observations that lie
below the detection limits. It is conceptually simpler than the other
statistic-based methods.

# Load Data

``` r
fn       <- 'gulfwatch_pah_data_long.csv'
pah_data <- read_csv(fn)
```

    ## Parsed with column specification:
    ## cols(
    ##   Original_Code = col_character(),
    ##   Site = col_character(),
    ##   Year = col_double(),
    ##   Sample_Type = col_character(),
    ##   PAH = col_character(),
    ##   Flag = col_logical(),
    ##   Concentration = col_double()
    ## )

``` r
pah_data_working <- pah_data %>%
  mutate(nd_to_zero = if_else(Flag, 0, Concentration),
         nd_to_half = if_else(Flag, Concentration/2, Concentration),
         nd_to_nd   = Concentration) %>%
  group_by(PAH) %>%
  mutate(n_samps  = sum(! is.na(Concentration)),
         n_nds    = sum(Flag, na.rm = TRUE),
         drop     = n_nds/n_samps > 0.80) %>%
  mutate(nd_to_ml = sub_cmeans(Concentration, Flag)) %>%
  mutate(nd_to_ml = if_else(drop, NA_real_, nd_to_ml)) %>%
  ungroup() %>%
  select(-n_samps, -n_nds, -drop, -Concentration)
```

``` r
pah_data_working_2 <- pah_data_working %>% 
  pivot_wider(id_cols = Original_Code:Sample_Type, names_from = PAH,
              values_from  = c(nd_to_zero, nd_to_half, nd_to_nd, nd_to_ml, Flag))%>%
  rename_all(~if_else(grepl('nd_to_zero', .), paste0(substr(.,12,nchar(.)),'_ZE'), .)) %>%
  rename_all(~if_else(grepl('nd_to_half', .), paste0(substr(.,12,nchar(.)),'_HF'), .)) %>%
  rename_all(~if_else(grepl('nd_to_nd',   .), paste0(substr(.,10,nchar(.)),'_ND'), .)) %>%
  rename_all(~if_else(grepl('nd_to_ml', .),  paste0(substr(.,10,nchar(.)),'_ML'), .)) %>%
  rename_all(~if_else(grepl('Flag', .), paste0(substr(.,6,nchar(.)),'_Flag'), .))
```

``` r
pah_data_working_3 <- pah_data_working_2 %>%
rowwise() %>%
  mutate(Total_ZE = sum(c_across(Naphthalene_ZE:`Benzo(g,h,i)Perylene_ZE`), na.rm=TRUE)) %>%
  mutate(Total_HF = sum(c_across(Naphthalene_HF:`Benzo(g,h,i)Perylene_HF`), na.rm=TRUE)) %>%
  mutate(Total_ND = sum(c_across(Naphthalene_ND:`Benzo(g,h,i)Perylene_ND`), na.rm=TRUE)) %>%
  mutate(Total_ML = sum(c_across(Naphthalene_ML:`Benzo(g,h,i)Perylene_ML`), na.rm=TRUE)) %>%
  select(Original_Code:Sample_Type, contains('Total'))
```

``` r
ggplot(pah_data_working_3, aes(x=Total_ZE))  +
  geom_line(aes(y=Total_HF), color = 'orange') +
  geom_line(aes(y=Total_ML), color = 'yellow') +
  geom_line(aes(y=Total_ND), color = 'red') +
  geom_abline(slope = 1, intercept = 0) +
  geom_abline(slope = 1, intercept = 100, lty = 2) +
  geom_abline(slope = 1, intercept = 200, lty = 3)
```

![](Calculate_Gulfwatch_PAH_Totals_files/figure-gfm/exploratory_graphic-1.png)<!-- -->
So, in general, selection of the type of total used makes a difference
of as much as several hundred ng/l, principally because adding the
detection limit or half the detection limit puts a quantitative floor
under possible values for any parameter for which there are missing
values. THe ML method uses al lavailalbe data to estimate likely
values,a nd generally does NOT put such a floor. it’s also worth
poiunitng out that bias is highest for compounds that have many
non-detects.

# Addressing Toxicity in Edible Tissues

> EPA. 2000. "Guidance for Assessing Chemical Contaminant Data for Use
> in Fish Advisories. Volume 2 Risk Assessment and Fish Consumption
> Limits. Third Edition.

Includes limited data on toxicity of individual PAHs, including the
following:

**5.6.11 Summary of EPA Health Benchmarks**  
\* *Chronic Toxicity*

|              |                  |
| ------------ | ---------------- |
| anthracene   | 3 x 10-1 mg/kg-d |
| fluoranthene | 4 x 10-2 mg/kg-d |
| fluorene     | 4 x 10-2 mg/kg-d |
| pyrene       | 3 x 10-2 mg/kg-d |

  - *Carcinogenicity*

<table style="width:49%;">
<colgroup>
<col style="width: 22%" />
<col style="width: 26%" />
</colgroup>
<tbody>
<tr class="odd">
<td>benzo[a]pyrene</td>
<td><div class="line-block">7.3 per mg/kg-d</div></td>
</tr>
</tbody>
</table>

In general, risk levels are expressed in terms of daily doses in mg/kg-d
Several individual PAHs have established reference doses (RfDs), but
confidence in those values i sgenerally low.

Values for individual PAHs tend to hover around 10^-2 t0 10^-1 mg/kg or
slightly higher

But I can’t go from those to acceptable levels in shellfish without
estimating size of shellfish meats, size of human, etc.

-----

We can guestimate, based on information found from unverifiable sources
on-line: One pound of mussels (let’s call it 500gms) is \~ 20 to 25
animals, and a fairly typical serving. It might have on the order of 20%
meat, so total consumed meat in a serving might be on the order of 100g.

Maine: An Encyclopedia"
(<https://maineanencyclopedia.com/blue-mussel-harvests/)suggests> meat
averages about 17% of weight of whole mussels in Maine.

so, if a 60 kg ( \~ 130 lb) human consumed a regular serving

dose == conc in ng/g / 1000mg/ng \* 100 g / (60kg \*1000g/kg)

we’re seeing total PAH concentrations in the most contaminated samples
from Portland Harbor on theorder of 11500 ng.g, so

Table 5-2 gives “Toxicity Equivalency Factors” for PAHs, apparently
referenced to Benzo\[a\]pyrene, for which we have a benchmark dose ,
above.

There apears to be an error listing the TEF for Benz\[a\]anthracene as
0, where byt context, and knowledge of its significant toxic effects,
0.1 appears more likely.
