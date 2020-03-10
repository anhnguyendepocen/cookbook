# Snippets

Random useful snippets that do not fit anywhere else. 

## Exporting tables that work in both PDF and Word

The `kable()` function from `library(knitr)` formats tables for all 3 output formats, but it is a bit limited.
`library(kableExtra)` is great for table customisations that work in PDF and HTML.
`kableExtra` functions do not work for Word output, but `library(flextable)` does work for Word (but not for PDF).

We can define our own table printing function that uses kableExtra or flextable based on output type:


```r
# This makes table resize or continue over multiple pages in all output types
# PDF powered by kableExtra, Word by flextable
mytable = function(x, caption = "", longtable = FALSE, ...){
  
  # if not latex or html then else is Word
  if (knitr::is_latex_output() | knitr::is_html_output()){
    knitr::kable(x, row.names = FALSE, align = c("l", "l", "r", "r", "r", "r", "r", "r", "r"), 
          booktabs = TRUE, caption = caption, longtable = longtable,
          linesep = "", ...) %>%
    kableExtra::kable_styling(latex_options = c("scale_down", "hold_position"))
  }else{
    flextable::flextable(x) %>% 
      flextable::autofit() %>% 
      flextable::width(j = 1, width = 1.5) %>% 
      flextable::height(i = 1, height = 0.5, part = "header")
  }
  
}
library(dplyr)
cars %>% 
  head() %>% 
  mytable()
```

\begin{table}[!h]

\caption{(\#tab:unnamed-chunk-1)}
\centering
\resizebox{\linewidth}{!}{
\begin{tabular}[t]{ll}
\toprule
speed & dist\\
\midrule
4 & 2\\
4 & 10\\
7 & 4\\
7 & 22\\
8 & 16\\
9 & 10\\
\bottomrule
\end{tabular}}
\end{table}



## Creating Reproducible R Examples to Share in the Group (binder, holepunch and docker)

When asking for help with R code having a reproducible example is crucial (some mock data that others can use along with your code to reproduce your error). Often this can be done easily with creation of a small tibble and posting of the code on slack but sometimes it requires more complex data or the error is due to something in the Linux system in which RStudio server is hosted. For example if the `cairo` package for Linux isn't installed then plots don't work. The `holepunch` package helps to reproduce examples like these (not suitable for projects with confidential data).

### Create Basic Reproducible Examples

The three main parts of the reproducible example (reprex) in Surgical Informatics are 1. packages, 2. small dataset and 3. code. Other things like R version and Linux version can be assumed as we all use one of only a few servers.

If you have a small (and confidential) set of data in a tibble or data frame called `my_data` and want it to be easily copied run: `dput(droplevels(my_data))`. This will print out code in the console that can be copy-pasted to reproduce the data frame. Alternatively use the `tibble` or `tribble` functions to create it from scratch (this is preferable for simple datasets). Then copy in the packages and finally the code (ideally the least amount possible to generate the error) and share with the group e.g.:


```r
library(tidyverse)

# Output generated from dput(droplevels(my_data))
data = structure(list(a = c(1, 2, 3), b = c("a", "b", "c"), c = 10:12), .Names = c("a", 
"b", "c"), row.names = c(NA, -3L), class = c("tbl_df", "tbl", 
"data.frame"))

data %>% 
  mutate(newvar = a /b)
```

```
## Error in a/b: non-numeric argument to binary operator
```


### `holepunch` - Complex Reproducible Examples

From your project with data you are happy to make public make sure you are backed up to `git` and `GitHub`. See the relevant chapter on how to do this. Then run the following:


```r
# Holepunch testing

remotes::install_github("karthik/holepunch")

library(holepunch)
write_compendium_description(package = "Title of my project", 
                             description = "Rough description of project or issue")

write_dockerfile(maintainer = "SurgicalInformatics")

generate_badge()

build_binder()
```

The file will generate some text to copy into the top of a `README.md` file. It will look like:

```r
<!-- badges: start -->
[![Launch Rstudio Binder](http://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/SurgicalInformatics/<project>/master?urlpath=rstudio)
<!-- badges: end -->
```

Now, whenever somebody clicks on the badge in the `README` on `GitHub` they will be taken to an RStudio server instance with all your files (excluding files listed in `.gitignore`), all the current versions of your package, all the current Linux packages and the current R version. They can then test your code in an near-identical environment to help identify the source of the error, their session will time out after 10 minutes of inactivity or 12 hours since starting and will not save anything so should only be used for bug-testing or quick examples.

As this is a free version of RStudio server there is a limit to what is supported and it shouldn't be used for computationally-intensive processes. 

And, as mentioned: **No confidential data.**

## Working with CHIs

Here are 4 functions for CHIs that could even be put in a small package. 
The Community Health Index (CHI) is a population register, which is used in Scotland for health care purposes. 
The CHI number uniquely identifies a person on the index.

### `chi_dob()` - Extract date of birth from CHI

Note `cutoff_2000`. 
As CHI has only a two digit year, need to decide whether year is 1900s or 2000s. 
I don't think there is a formal way of determining this. 


```r
library(dplyr)
chi = c("1009701234", "1811431232", "1304496368")
# These CHIs are not real. 
# The first is invalid, two and three are valid. 

# Cut-off any thing before that number is considered 2000s
# i.e. at cutoff_2000 = 20, "18" is considered 2018, rather than 1918. 
chi_dob = function(.data, cutoff_2000 = 20){
  .data %>% 
    stringr::str_extract(".{6}") %>% 
    lubridate::parse_date_time2("dmy", cutoff_2000 = cutoff_2000) %>% 
    lubridate::as_date() # Make Date object, rather than POSIXct
}

chi_dob(chi)
```

```
## [1] "1970-09-10" "1943-11-18" "1949-04-13"
```

```r
# From tibble
tibble(chi = chi) %>% 
  mutate(
    dob = chi_dob(chi)
  )
```

```
## # A tibble: 3 x 2
##   chi        dob       
##   <chr>      <date>    
## 1 1009701234 1970-09-10
## 2 1811431232 1943-11-18
## 3 1304496368 1949-04-13
```

### `chi_gender()` - Extract gender from CHI

Ninth digit is odd for men and even for women. 
A test for even is `x modulus 2 == 0`.


```r
chi_gender = function(.data){
  .data %>% 
    stringr::str_sub(9, 9) %>% 
    as.numeric() %>% 
    {ifelse(. %% 2 == 0, "Female", "Male")}
}

chi_gender(chi)
```

```
## [1] "Male"   "Male"   "Female"
```

```r
# From tibble
tibble(chi = chi) %>% 
  mutate(
    dob = chi_dob(chi),
    gender = chi_gender(chi)
  )
```

```
## # A tibble: 3 x 3
##   chi        dob        gender
##   <chr>      <date>     <chr> 
## 1 1009701234 1970-09-10 Male  
## 2 1811431232 1943-11-18 Male  
## 3 1304496368 1949-04-13 Female
```

### `chi_age()` - Extract age from CHI

Works for a single date or a vector of dates.


```r
chi_age = function(.data, ref_date, cutoff_2000 = 20){
  dob = chi_dob(.data, cutoff_2000 = cutoff_2000)
  lubridate::interval(dob, ref_date) %>% 
    as.numeric("years") %>% 
    floor()
}

# Today
chi_age(chi, Sys.time())
```

```
## [1] 49 76 70
```

```r
# Single date
library(lubridate)
chi_age(chi, dmy("11/09/2018"))
```

```
## [1] 48 74 69
```

```r
# Vector
dates = dmy("11/09/2018",
            "09/05/2015",
            "10/03/2014")
chi_age(chi, dates)
```

```
## [1] 48 71 64
```

```r
# From tibble
tibble(chi = chi) %>% 
  mutate(
    dob = chi_dob(chi),
    gender = chi_gender(chi),
    age = chi_age(chi, Sys.time())
  )
```

```
## # A tibble: 3 x 4
##   chi        dob        gender   age
##   <chr>      <date>     <chr>  <dbl>
## 1 1009701234 1970-09-10 Male      49
## 2 1811431232 1943-11-18 Male      76
## 3 1304496368 1949-04-13 Female    70
```

### `chi_valid()` - Logical test for valid CHI

The final digit of the CHI can be used to test that the number is correct via the modulus 11 algorithm. 


```r
chi_valid = function(.data){
  .data %>% 
    stringr::str_split("", simplify = TRUE) %>% 
    .[, -10] %>%              # Working with matrices hence brackets
    apply(1, as.numeric) %>%  # Convert from string
    {seq(10, 2) %*% .} %>%    # Multiply and sum step
    {. %% 11} %>%             # Modulus 11
    {11 - .} %>%              # Substract from 11
    dplyr::near(              # Compare result with 10th digit. 
      {stringr::str_sub(chi, 10) %>% as.numeric()}
    ) %>% 
    as.vector()
}

chi_valid(chi)
```

```
## [1] FALSE  TRUE  TRUE
```

```r
# From tibble
tibble(chi = chi) %>% 
  mutate(
    dob = chi_dob(chi),
    gender = chi_gender(chi),
    age = chi_age(chi, Sys.time()),
    chi_valid = chi_valid(chi)
  )
```

```
## # A tibble: 3 x 5
##   chi        dob        gender   age chi_valid
##   <chr>      <date>     <chr>  <dbl> <lgl>    
## 1 1009701234 1970-09-10 Male      49 FALSE    
## 2 1811431232 1943-11-18 Male      76 TRUE     
## 3 1304496368 1949-04-13 Female    70 TRUE
```

## Working with dates

### Difference between two dates

I always forget how to do this neatly. 
I often want days as a numeric, not a lubridate type object. 


```r
library(lubridate)
date1 = dmy("12/03/2018", "14/05/2017")
date2 = dmy("11/09/2019", "11/04/2019")

interval(date1, date2) %>% 
  as.numeric("days")
```

```
## [1] 548 697
```

### Lags

This is useful for calculating, for instance, the period off medications. Lags are much better than long to wide solutions for this. 


```r
library(tidyverse)
library(lubridate)
id = c(2, 2, 2, 2, 3, 5) 
medication = c("aspirin", "aspirin", "aspirin", "tylenol", "lipitor", "advil") 
start.date = c("05/01/2017", "05/30/2017", "07/15/2017", "05/01/2017", "05/06/2017", "05/28/2017")
stop.date = c("05/04/2017", "06/10/2017", "07/27/2017", "05/15/2017", "05/12/2017", "06/13/2017")
df = tibble(id, medication, start.date, stop.date)
df
```

```
## # A tibble: 6 x 4
##      id medication start.date stop.date 
##   <dbl> <chr>      <chr>      <chr>     
## 1     2 aspirin    05/01/2017 05/04/2017
## 2     2 aspirin    05/30/2017 06/10/2017
## 3     2 aspirin    07/15/2017 07/27/2017
## 4     2 tylenol    05/01/2017 05/15/2017
## 5     3 lipitor    05/06/2017 05/12/2017
## 6     5 advil      05/28/2017 06/13/2017
```

```r
df %>%
  mutate_at(c("start.date", "stop.date"), lubridate::mdy) %>% # make a date
  arrange(id, medication, start.date) %>% 
  group_by(id, medication) %>% 
  mutate(
    start_date_diff = start.date - lag(start.date),
    medication_period = stop.date-start.date
  )
```

```
## # A tibble: 6 x 6
## # Groups:   id, medication [4]
##      id medication start.date stop.date  start_date_diff medication_period
##   <dbl> <chr>      <date>     <date>     <drtn>          <drtn>           
## 1     2 aspirin    2017-05-01 2017-05-04 NA days          3 days          
## 2     2 aspirin    2017-05-30 2017-06-10 29 days         11 days          
## 3     2 aspirin    2017-07-15 2017-07-27 46 days         12 days          
## 4     2 tylenol    2017-05-01 2017-05-15 NA days         14 days          
## 5     3 lipitor    2017-05-06 2017-05-12 NA days          6 days          
## 6     5 advil      2017-05-28 2017-06-13 NA days         16 days
```

### Pulling out "change in status" data

If you have a number of episodes per patient, each with a status and a time, then you need to do this as a starting point for CPH analysis. 

#### Example data


```r
library(dplyr)
library(lubridate)
library(finalfit)
mydata = tibble(
  id = c(1,1,1,1,2,2,2,2,3,3,3,3,4,4,4,4,5,5,5,5),
  status = c(0,0,0,1,0,0,1,1,0,0,0,0,0,1,1,1,0,0,1,1),
  group = c(rep(0, 8), rep(1, 12)) %>% factor(),
  opdate = rep("2010/01/01", 20) %>% ymd(),
  status_date = c(
    "2010/02/01", "2010/03/01", "2010/04/01", "2010/05/01",
    "2010/02/02", "2010/03/02", "2010/04/02", "2010/05/02",
    "2010/02/03", "2010/03/03", "2010/04/03", "2010/05/03",
    "2010/02/04", "2010/03/04", "2010/04/04", "2010/05/04",
    "2010/02/05", "2010/03/05", "2010/04/05", "2010/05/05"
  ) %>% ymd()
)
mydata
```

```
## # A tibble: 20 x 5
##       id status group opdate     status_date
##    <dbl>  <dbl> <fct> <date>     <date>     
##  1     1      0 0     2010-01-01 2010-02-01 
##  2     1      0 0     2010-01-01 2010-03-01 
##  3     1      0 0     2010-01-01 2010-04-01 
##  4     1      1 0     2010-01-01 2010-05-01 
##  5     2      0 0     2010-01-01 2010-02-02 
##  6     2      0 0     2010-01-01 2010-03-02 
##  7     2      1 0     2010-01-01 2010-04-02 
##  8     2      1 0     2010-01-01 2010-05-02 
##  9     3      0 1     2010-01-01 2010-02-03 
## 10     3      0 1     2010-01-01 2010-03-03 
## 11     3      0 1     2010-01-01 2010-04-03 
## 12     3      0 1     2010-01-01 2010-05-03 
## 13     4      0 1     2010-01-01 2010-02-04 
## 14     4      1 1     2010-01-01 2010-03-04 
## 15     4      1 1     2010-01-01 2010-04-04 
## 16     4      1 1     2010-01-01 2010-05-04 
## 17     5      0 1     2010-01-01 2010-02-05 
## 18     5      0 1     2010-01-01 2010-03-05 
## 19     5      1 1     2010-01-01 2010-04-05 
## 20     5      1 1     2010-01-01 2010-05-05
```

#### Compute time from op date to current review
... if necessary


```r
mydata = mydata %>% 
  arrange(id, status_date) %>% 
  mutate(
    time = interval(opdate, status_date) %>% as.numeric("days")
  )
mydata
```

```
## # A tibble: 20 x 6
##       id status group opdate     status_date  time
##    <dbl>  <dbl> <fct> <date>     <date>      <dbl>
##  1     1      0 0     2010-01-01 2010-02-01     31
##  2     1      0 0     2010-01-01 2010-03-01     59
##  3     1      0 0     2010-01-01 2010-04-01     90
##  4     1      1 0     2010-01-01 2010-05-01    120
##  5     2      0 0     2010-01-01 2010-02-02     32
##  6     2      0 0     2010-01-01 2010-03-02     60
##  7     2      1 0     2010-01-01 2010-04-02     91
##  8     2      1 0     2010-01-01 2010-05-02    121
##  9     3      0 1     2010-01-01 2010-02-03     33
## 10     3      0 1     2010-01-01 2010-03-03     61
## 11     3      0 1     2010-01-01 2010-04-03     92
## 12     3      0 1     2010-01-01 2010-05-03    122
## 13     4      0 1     2010-01-01 2010-02-04     34
## 14     4      1 1     2010-01-01 2010-03-04     62
## 15     4      1 1     2010-01-01 2010-04-04     93
## 16     4      1 1     2010-01-01 2010-05-04    123
## 17     5      0 1     2010-01-01 2010-02-05     35
## 18     5      0 1     2010-01-01 2010-03-05     63
## 19     5      1 1     2010-01-01 2010-04-05     94
## 20     5      1 1     2010-01-01 2010-05-05    124
```

#### Pull out "change of status"

```r
mydata = mydata %>% 
  group_by(id) %>% 
  mutate(
    status_change = status - lag(status) == 1,                          # Mark TRUE if goes from 0 to 1
    status_nochange = sum(status) == 0,                                 # Mark if no change from 0
    status_nochange_keep = !duplicated(status_nochange, fromLast= TRUE) # Mark most recent "no change" episode
  ) %>% 
  filter(status_change | (status_nochange & status_nochange_keep)) %>%  # Filter out other episodes
  select(-c(status_change, status_nochange, status_nochange_keep))      # Remove columns not needed
mydata
```

```
## # A tibble: 5 x 6
## # Groups:   id [5]
##      id status group opdate     status_date  time
##   <dbl>  <dbl> <fct> <date>     <date>      <dbl>
## 1     1      1 0     2010-01-01 2010-05-01    120
## 2     2      1 0     2010-01-01 2010-04-02     91
## 3     3      0 1     2010-01-01 2010-05-03    122
## 4     4      1 1     2010-01-01 2010-03-04     62
## 5     5      1 1     2010-01-01 2010-04-05     94
```

#### Run CPH

```r
mydata %>% 
  finalfit("Surv(time, status)", "group")
```

```
##  Dependent: Surv(time, status)         all          HR (univariable)
##                          group 0 2 (100.0)                         -
##                                1 3 (100.0) 0.76 (0.10-5.51, p=0.786)
##         HR (multivariable)
##                          -
##  0.76 (0.10-5.51, p=0.786)
```