
<!-- README.md is generated from README.Rmd. Please edit that file -->

# dtplyr

<!-- badges: start -->

[![CRAN
status](https://www.r-pkg.org/badges/version/dtplyr)](https://cran.r-project.org/package=dtplyr)
[![Travis build
status](https://travis-ci.org/tidyverse/dtplyr.svg?branch=master)](https://travis-ci.org/tidyverse/dtplyr)
[![Codecov test
coverage](https://codecov.io/gh/tidyverse/dtplyr/branch/master/graph/badge.svg)](https://codecov.io/gh/tidyverse/dtplyr?branch=master)
[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
<!-- badges: end -->

## Overview

dtplyr provides a dplyr backend for
[data.table](http://r-datatable.com/). The goal of dtplyr is to allow
you to write dplyr code that is automatically translated to the
equivalent, but usually much faster, data.table code.

Compared to the previous release, this version of dtplyr is a complete
rewrite that focusses only on lazy evaluation triggered by use of
`lazy_dt()`. This means that no computation is performed until you
explicitly request it with `as.data.table()`, `as.data.frame()` or
`as_tibble()`. This has a considerable advantage over the previous
version (which eagerly evaluated each step) because it allows dtplyr to
generate significantly more performant translations. This is a large
change that breaks all existing uses of dtplyr. But frankly, dtplyr was
pretty useless before because it did such a bad job of generating
data.table code. Fortunately few people used it, so a major overhaul was
possible.

See `vignette("translation")` for details of the current translations,
and [table.express](https://github.com/asardaes/table.express) and
[rqdatatable](https://github.com/WinVector/rqdatatable/) for related
work.

## Development status

dtplyr is currently marked as experimental because it has had little
usage in the wild. I expect that it will rapidly move towards maturing
after a few weeks of feedback from users.

## Installation

You can install from CRAN with:

``` r
install.packages("dtplyr")
```

Or try the development version from GitHub with:

``` r
# install.packages("devtools")
devtools::install_github("tidyverse/dtplyr")
```

## Usage

To use dtplyr, you must at least load dtplyr and dplyr. You may also
want to load [data.table](http://r-datatable.com/) so you can access the
other goodies that it provides:

``` r
library(data.table)
library(dtplyr)
library(dplyr, warn.conflicts = FALSE)
```

Then use `lazy_dt()` to create a “lazy” data table that tracks the
operations performed on it.

``` r
mtcars2 <- lazy_dt(mtcars)
```

You can preview the transformation (including the generated data.table
code) by printing the result:

``` r
mtcars2 %>% 
  filter(wt < 5) %>% 
  mutate(l100k = 235.21 / mpg) %>% # liters / 100 km
  group_by(cyl) %>% 
  summarise(l100k = mean(l100k))
#> Source: local data table [?? x 2]
#> Call:   `_DT1`[wt < 5][, `:=`(l100k = 235.21/mpg)][, .(l100k = mean(l100k)), 
#>     keyby = .(cyl)]
#> 
#>     cyl l100k
#>   <dbl> <dbl>
#> 1     4  9.05
#> 2     6 12.0 
#> 3     8 14.9 
#> 
#> # Use as.data.table()/as.data.frame()/as_tibble() to access results
```

But generally you should reserve this only for debugging, and use
`as.data.table()`, `as.data.frame()`, or `as_tibble()` to indicate that
you’re done with the transformation and want to access the results:

``` r
mtcars2 %>% 
  filter(wt < 5) %>% 
  mutate(l100k = 235.21 / mpg) %>% # liters / 100 km
  group_by(cyl) %>% 
  summarise(l100k = mean(l100k)) %>% 
  as_tibble()
#> # A tibble: 3 x 2
#>     cyl l100k
#>   <dbl> <dbl>
#> 1     4  9.05
#> 2     6 12.0 
#> 3     8 14.9
```

## Why is dtplyr slower than data.table?

There are three primary reasons that dtplyr will always be somewhat
slower than data.table:

  - Each dplyr verb must do some work to convert dplyr syntax to
    data.table syntax. This takes time proportional to the complexity of
    the input code, not the input *data*, so should be a negligible
    overhead for large datasets. [Initial
    benchmarks](https://dtplyr.tidyverse.org/articles/translation.html#performance)
    suggest that the overhead should be under 1ms per dplyr call.

  - Some data.table expressions have no direct dplyr equivalent. For
    example, there’s no way to express cross- or rolling-joins with
    dplyr.

  - To match dplyr semantics, `mutate()` does not modify in place by
    default. This means that most expressions involving `mutate()` must
    make a copy that would not be necessary if you were using data.table
    directly. (You can opt out of this behaviour in `lazy_dt()` with
    `immutable = FALSE`).
