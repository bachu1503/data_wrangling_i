Session 5: Tidy
================

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.2 ──
    ## ✔ ggplot2 3.3.6      ✔ purrr   0.3.4 
    ## ✔ tibble  3.1.8      ✔ dplyr   1.0.10
    ## ✔ tidyr   1.2.1      ✔ stringr 1.4.1 
    ## ✔ readr   2.1.2      ✔ forcats 0.5.2 
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

## Data Import: CSVs

``` r
pulse_df <- haven::read_sas('data/public_pulse_data.sas7bdat') %>% 
  janitor::clean_names()
```

## Pivot Longer

With our new understanding of tidy data, we quickly recognize a problem:
the BDI score is spread across four columns, which correspond to four
observation times. We can fix this problem using pivot_longer:

There are also some additional transformations needed to wrap up the
data wrangling process, like changing bl to 00m for consistency across
visits and converting visit to a factor variable. (It’s possible that
you would want visit to be a numeric variable instead, which could be
done with a different call to mutate.) Lastly, it’s nice to organize the
data into a reasonable order.

Altogether, then, the code below will import, tidy, and transform the
PULSE dataset into a usable format:

``` r
pulse_df <- haven::read_sas('data/public_pulse_data.sas7bdat') %>% 
  janitor::clean_names() %>% 
  pivot_longer(
    bdi_score_bl:bdi_score_12m, # columns
    names_to = "visit", # name of new variable
    values_to = "bdi", # name of column where values go
    names_prefix = "bdi_score_" # removes string "bdi_score_" 
  ) %>%
  relocate(visit) %>%
  mutate(
    visit = replace(visit, visit == "bl", "00m"), # replace "bl" with "00m"
    visit = as.factor(visit) # visit as categorical
  ) %>%
  arrange(id, visit)

print(pulse_df, n = 12)
```

    ## # A tibble: 4,348 × 5
    ##    visit    id   age sex     bdi
    ##    <fct> <dbl> <dbl> <chr> <dbl>
    ##  1 00m   10003  48.0 male      7
    ##  2 01m   10003  48.0 male      1
    ##  3 06m   10003  48.0 male      2
    ##  4 12m   10003  48.0 male      0
    ##  5 00m   10015  72.5 male      6
    ##  6 01m   10015  72.5 male     NA
    ##  7 06m   10015  72.5 male     NA
    ##  8 12m   10015  72.5 male     NA
    ##  9 00m   10022  58.5 male     14
    ## 10 01m   10022  58.5 male      3
    ## 11 06m   10022  58.5 male      8
    ## 12 12m   10022  58.5 male     NA
    ## # … with 4,336 more rows

**Learning Assessment**: In the litters data, the variables gd0_weight
and gd18_weight give the weight of the mother mouse on gestational days
0 and 18. Write a data cleaning chain that retains only litter_number
and these columns; produces new variables gd and weight; and makes gd a
numeric variable taking values 0 and 18 (for the last part, you might
want to use recode …). Is this version “tidy”?

``` r
litters <- read_csv("data/FAS_litters.csv") %>%
  janitor::clean_names() %>%
  select(litter_number, ends_with("weight")) %>%
  pivot_longer(
    gd0_weight:gd18_weight,
    names_to = "gd",
    values_to = "weight"
  ) %>%
  mutate(
    gd = recode(gd, "gd0_weight" = 0, "gd18_weight" = 18) # recodes  values
  )
```

    ## Rows: 49 Columns: 8
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (2): Group, Litter Number
    ## dbl (6): GD0 weight, GD18 weight, GD of Birth, Pups born alive, Pups dead @ ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

## Pivot Wider

We’ve been exclusively interested in tidying data, but we’ve admitted
that sometimes untidy is better for human consumption. For that reason
we’re going to take a short digression into untidying your tidy data.

The code below creates a tidy dataset that could result from an
analysis. This is the correct format for additional analysis or
visualization, but doesn’t facilitate quick comparisons for human
readers.

``` r
analysis_result = tibble(
  group = c("treatment", "treatment", "placebo", "placebo"),
  time = c("pre", "post", "pre", "post"),
  mean = c(4, 8, 3.5, 4)
)

analysis_result
```

    ## # A tibble: 4 × 3
    ##   group     time   mean
    ##   <chr>     <chr> <dbl>
    ## 1 treatment pre     4  
    ## 2 treatment post    8  
    ## 3 placebo   pre     3.5
    ## 4 placebo   post    4

An alternative presentation of the same data might have groups in rows,
times in columns, and mean values in table cells. This is decidedly
non-tidy; to get there from here we’ll need to use pivot_wider, which is
the inverse of pivot_longer:

``` r
pivot_wider(
  analysis_result, 
  names_from = "time", 
  values_from = "mean")
```

    ## # A tibble: 2 × 3
    ##   group       pre  post
    ##   <chr>     <dbl> <dbl>
    ## 1 treatment   4       8
    ## 2 placebo     3.5     4

We’re pretty much there now – in some cases you might use select to
reorder columns, and (depending on your goal) use knitr::kable() to
produce a nicer table for reading.

## Binding rows

We’ve looked at single-table non-tidy data, but non-tidiness often stems
from relevant data spread across multiple tables. In the simplest case,
these tables are basically the same and can be stacked to produce a tidy
dataset. That’s the setting in LotR_words.xlsx, where the word counts
for different races and genders in each movie in the trilogy are spread
across distinct data rectangles (these data are based on this example).

To produce the desired tidy dataset, we first need to read each table
and do some cleaning.

``` r
fellowship_ring = 
  readxl::read_excel("./data/LotR_Words.xlsx", range = "B3:D6") %>%
  mutate(movie = "fellowship_ring")

two_towers = 
  readxl::read_excel("./data/LotR_Words.xlsx", range = "F3:H6") %>%
  mutate(movie = "two_towers")

return_king = 
  readxl::read_excel("./data/LotR_Words.xlsx", range = "J3:L6") %>%
  mutate(movie = "return_king")
```

Here it was necessary to add a variable to each dataframe indicating the
movie; that information had stored elsewhere in the original
spreadsheet. As an aside, the three code snippets above are all
basically the same except for the range and the movie name – later we’ll
see a better way to handle cases like this by writing our own functions,
but this works for now.

Once each table is ready to go, we can stack them up using `bind_rows`
and tidy the result:

``` r
lotr_tidy = 
  bind_rows(fellowship_ring, two_towers, return_king) %>%
  janitor::clean_names() %>%
  pivot_longer(
    female:male,
    names_to = "gender", 
    values_to = "words") %>%
  mutate(race = str_to_lower(race)) %>% 
  select(movie, everything()) 

lotr_tidy
```

    ## # A tibble: 18 × 4
    ##    movie           race   gender words
    ##    <chr>           <chr>  <chr>  <dbl>
    ##  1 fellowship_ring elf    female  1229
    ##  2 fellowship_ring elf    male     971
    ##  3 fellowship_ring hobbit female    14
    ##  4 fellowship_ring hobbit male    3644
    ##  5 fellowship_ring man    female     0
    ##  6 fellowship_ring man    male    1995
    ##  7 two_towers      elf    female   331
    ##  8 two_towers      elf    male     513
    ##  9 two_towers      hobbit female     0
    ## 10 two_towers      hobbit male    2463
    ## 11 two_towers      man    female   401
    ## 12 two_towers      man    male    3589
    ## 13 return_king     elf    female   183
    ## 14 return_king     elf    male     510
    ## 15 return_king     hobbit female     2
    ## 16 return_king     hobbit male    2673
    ## 17 return_king     man    female   268
    ## 18 return_king     man    male    2459

Having the data in this form will make it easier to make comparisons
across movies, aggregate within races across the trilogy, and perform
other analyses.

## Joining datasets

Data can be spread across multiple related tables, in which case it is
necessary to combine or join them prior to analysis. We’ll focus on the
problem of combining two tables only; combining three or more is done
step-by-step using the same ideas.

There are four major ways join dataframes x and y:

-   Inner: keeps data that appear in both x and y
-   Left: keeps data that appear in x
-   Right: keeps data that appear in y
-   Full: keeps data that appear in either x or y

Left joins are the most common, because they add data from a smaller
table y into a larger table x without removing anything from x.

As an example, consider the data tables in `FAS_pups.csv` and
`FAS_litters.csv`, which are related through the Litter Number variable.
The former contains data unique to each pup, and the latter contains
data unique to each litter. We can combine these using a left join of
litter data into pup data; doing so retains data on each pup and adds
data in new columns.

(While revisiting this example, take a look at the `group` variable in
the litters dataset: this encodes both dose and day of treatment! We’ll
fix that bit of untidiness as part of the processing pipeline. I’m also
going to address a pet peeve of mine, which is coding sex as an
ambiguous numeric variable.)

``` r
pup_data = 
  read_csv("./data/FAS_pups.csv") %>%
  janitor::clean_names() %>%
  mutate(
    sex = recode(sex, `1` = "male", `2` = "female"),
    sex = factor(sex)) 
## Rows: 313 Columns: 6
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (1): Litter Number
## dbl (5): Sex, PD ears, PD eyes, PD pivot, PD walk
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

litter_data = 
  read_csv("./data/FAS_litters.csv") %>%
  janitor::clean_names() %>%
  separate(group, into = c("dose", "day_of_tx"), sep = 3) %>%
  relocate(litter_number) %>%
  mutate(
    wt_gain = gd18_weight - gd0_weight,
    dose = str_to_lower(dose))
## Rows: 49 Columns: 8
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (2): Group, Litter Number
## dbl (6): GD0 weight, GD18 weight, GD of Birth, Pups born alive, Pups dead @ ...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

fas_data = 
  left_join(pup_data, litter_data, by = "litter_number")

fas_data
## # A tibble: 313 × 15
##   litter_n…¹ sex   pd_ears pd_eyes pd_pi…² pd_walk dose  day_o…³ gd0_w…⁴ gd18_…⁵
##   <chr>      <fct>   <dbl>   <dbl>   <dbl>   <dbl> <chr> <chr>     <dbl>   <dbl>
## 1 #85        male        4      13       7      11 con   7          19.7    34.7
## 2 #85        male        4      13       7      12 con   7          19.7    34.7
## 3 #1/2/95/2  male        5      13       7       9 con   7          27      42  
## 4 #1/2/95/2  male        5      13       8      10 con   7          27      42  
## 5 #5/5/3/83… male        5      13       8      10 con   7          26      41.4
## # … with 308 more rows, 5 more variables: gd_of_birth <dbl>,
## #   pups_born_alive <dbl>, pups_dead_birth <dbl>, pups_survive <dbl>,
## #   wt_gain <dbl>, and abbreviated variable names ¹​litter_number, ²​pd_pivot,
## #   ³​day_of_tx, ⁴​gd0_weight, ⁵​gd18_weight
```

We made the key explicit in the join. By default, the `*_join` functions
in `dplyr` will try to determine the key(s) based on variable names in
the datasets you want to join. This is often but not always sufficient,
and an extra step to make the key clear will help you and others reading
your code.

Note that joining is not particularly amenable to the `%>%` operator
because it is fundamentally non-linear: two separate datasets are coming
together, rather than a single dataset being processed in a step-by-step
fashion.

As a final point, the `*_join` functions are very much related to SQL
syntax, but emphasize operations common to data analysis.

**Learning Assessment**: The datasets in this zip file contain
de-identified responses to surveys included in past years of this
course. Both contain a unique student identifier; the first has
responses to a question about operating systems, and the second has
responses to questions about degree program and git experience. Write a
code chunk that imports and cleans both datasets, and then joins them.
