-   [Introduction](#introduction)
-   [Main features of the package](#main-features-of-the-package)
-   [Installation](#installation)
-   [Functionality](#functionality)
-   [Explanation on how the tool works](#explanation-on-how-the-tool-works)
-   [Application](#application)
-   [Output and tools for measuring information loss](#output-and-tools-for-measuring-information-loss)
-   [Appendix](#appendix)

Introduction
------------

The targeted record swapping method is a pre-tabular Statistical Disclosure Control (SDC) technique. Records holding a high risk of identification are swapped within the level they hold that risk to a different area on the same hierarchy level. Possible swap partners are all records matching a given similarity profile. The goal of this package is to apply record swapping to statistical tables. This algorithm can be applied to micro data sets and is based on C++. For more information on the function `recordSwap()` one can call the corresponding help page through:

``` r
?recordSwap
```

Main features of the package
----------------------------

First an overview of the main features of the `recordSwapping` package before illustrating how to use it on a synthetic data set.

-   The swaprate is fixed and the swapping aims to strike the rate as good as possible.
-   Still, it is possible that the swaprate is overcome or undercut due to rounding mistakes or parameter setting.
-   The record swapping does not change single values, it only swaps records and all values belonging to that record.
-   Identification risk is calculated using counts over the hierarchy levels and the combination of all risk variables.

Installation
------------

The package can be installed with

``` r
library(devtools)
```

``` r
install.packages("recordSwapping")
```

Functionality
-------------

The targeted record swapping can be applied using `recordSwap()`. The function has the following arguments:

-   `data`: micro data set including only integer vectors, the data supplied should ideally be a datatable or dataframe
-   `similar`: column indices of variables in data which should be considered when swapping records. Only records having identical values in the specified variable will be swapped. There is the option to create multiple profiles. In that case the second one will only be checked if the first one does not find any matches (e.g. similar &lt;- c(c(2,3),c(3)) first checks similarity in the second and the third column and if there is no match similarity will be checked for the third column only)
-   `hierarchy`: column indices of variables in data which refer to the (geographic) hierarchy of the micro data set (for instance province &gt; county &gt; district &gt; municipality)
-   `risk_variables`: column indices of variables in data which will be considered for estimating the risk
-   `hid`: column index in data which refers to the household identifier
-   `k_anonymity`: integer defining the threshold of high risk records, a record is considered to be risky if not at least k records have the same value in the columns defined as risk variables
-   `swaprate`: double integer defining the amount of records that should be swapped
-   `seed`: integer defining the seed for the random number generator used for reproducibility

Random test data can be created using `dat_test <- recordSwapping:::create.dat()`.

Explanation on how the tool works
---------------------------------

-   To start with, the micro data is ordered by `hid` and the identification risk for each record gets calculated. So far only counts is used as identification risk.
-   At each hierarchy level a set of values is created for records to be swapped. In all but the lowest hierarchy level this contains of all records which do not fulfill the `k_anonymity` and have not already been swapped. Those records are being swapped with other records, which do not belong to the same area on the same hierarchy level and were not swapped before.
-   Targeted record swapping starts from the highest level down. Swapping is applied at each hierarchy level to all records in the set of values. If `k_anonymity` is set to 0, only the swaprate is considered for swapping. (Careful if the data set is very small, the swaprate is often not stroke.) In general all records not fulfilling `k_anonymity` are swapped, even if the actual swaprate is already overcome. Contrary, additional swaps are made on the lowest hierarchy level if not enough swaps have been made before to reach the swaprate.
-   Swapping refers to the interchange of (geographic) variables defined in `hierarchy`. When a record is swapped all other records belonging to the same `hid` are swapped as well.

Application
-----------

So far the package was tested on randomly generated micro data, containing three or four geographic levels (province, district, county and munincipality) used as hierarchy (`nuts1`to `lau2`). Additionally, the data set contained other soziodemographic information, such as age and sex.

### Loading the package

``` r
library(data.table)
library(recordSwapping)
library(ggplot2)
```

### Creating a random synthetic data set

#### Creating geo data

``` r
N <- 820000   #creating a synthetic population of 820000 people 
              #geo data of length eight and dots separating all hierarchy levels, first create five digit data

set.seed(12)  #generate the same random data everytime, keep it fixed for better comparability
c_geo.m <- c('01.0.51','01.0.53','01.0.54','01.0.55','01.0.56','01.0.57','01.0.58','01.0.59','01.0.60',
             '01.0.61', '01.0.62','02.0.51','02.0.52','02.0.54','02.0.55','02.0.56','02.0.57','02.0.59',
             '02.1.51','02.1.52','02.1.53','02.1.54','02.1.56','02.1.57','02.1.58','02.1.59','02.2.52',
             '02.2.53','02.2.54','02.2.55','02.2.56','02.2.58','02.3.50','02.3.52','02.3.53','02.3.54',
             '02.3.56','02.3.57','02.3.58','02.3.59','02.3.60','02.3.61','03.1.51','03.1.52','03.1.53',
             '03.1.54','03.1.55','03.1.56','03.1.57','03.1.58','03.2.51','03.2.52','03.2.54','03.2.55',
             '03.2.56','03.2.57','03.3.51','03.3.52','03.3.53','03.3.54','03.3.55','03.3.56','03.3.57',
             '03.3.58','03.3.59','03.3.60','03.3.61','03.4.51','03.4.52','03.4.53','03.4.54','03.4.55',
             '03.4.56')

#add level `lau2` consisting of three digit random numbers to the already created geo data to change from `geo.m` to `geo.h`
x1<-matrix(data=NA, nrow= 149, ncol=11, byrow=FALSE, dimnames = NULL)
for (i in 1:11) {
  s<-sample(100:999, 149, replace=FALSE)
  for (j in 1:149){
    x1[j,i]<-paste0(c_geo.m[i],".", as.character(s[j]))
  }
}
#in case of "02" there are no municipalities, therefore lau2 is set to "000" for all data beginning with "02"
x2<-matrix(data=NA, nrow=31 , ncol=1, byrow=FALSE, dimnames = NULL)
for (i in 1:31){
  x2[i,1]<-paste0(c_geo.m[11+i],".","000")
}
x3<-matrix(data=NA, nrow=70 , ncol=30, byrow=FALSE, dimnames = NULL)
for (i in 1:30) {
  s<-sample(100:999, 70, replace=FALSE)
  for (j in 1:70){
    x3[j,i]<-paste0(c_geo.m[43+i],".", as.character(s[j]))
  }
}

x1<-as.vector(x1)
x2<-as.vector(x2)
x3<-as.vector(x3)

#multiply the geo data and attach geo value to each statistical unit
z_geo.h<-c(x1,x2,x3)
p_geo.h <- runif(3770)
p_geo.h <- p_geo.h/sum(p_geo.h)
geo.h <- rep(z_geo.h, round(p_geo.h * N))
geo.h <- sample(geo.h, N, replace=TRUE)
```

#### Creating hierarchy

Create hierarchy levels splitting the original variable `geo.h` into PP-D-CC-MMM (P: province, D: district, C:county, M: munincipality) works with `hier_separate()`.

``` r
library(tidyr)
hier_separate <- function(var, col.names = c("nuts1","nuts2", "nuts3","lau2"), len, dots=TRUE){
  
  if (dots){
    das <- data.table(vname=var, as.data.frame(var) %>% tidyr::separate(var, col.names ))
  } else {
    pat <- paste0("(.{",len,"})", collapse = "")
    das <- data.table(vname=var,as.data.frame(var) %>% extract(var, into=col.names, pat))
  }
  
  names(das)[1] <- deparse(substitute(var))
  return(das)
}
```

If the data includes dots separating the different hierarchy levels it first of all is necessary to eliminate the dots being included in the geo data, using the function `eliminate_dots()`. For more details on the functions see [Appendix](#appendix).

``` r

# if geo data is with dots, so use the following
dt_new <- hier_separate(var=geo.h, col.names = c("nuts1","nuts2", "nuts3","lau2"), dots=TRUE)
dt_new
>               geo.h nuts1 nuts2 nuts3 lau2
>      1: 03.1.54.111    03     1    54  111
>      2: 03.1.53.516    03     1    53  516
>      3: 03.1.52.138    03     1    52  138
>      4: 01.0.58.402    01     0    58  402
>      5: 03.2.52.911    03     2    52  911
>     ---                                   
> 819996: 01.0.60.900    01     0    60  900
> 819997: 03.1.55.317    03     1    55  317
> 819998: 01.0.54.119    01     0    54  119
> 819999: 03.4.53.364    03     4    53  364
> 820000: 01.0.56.941    01     0    56  941



## However, let's assume there is a variable without dots

# First, remove the dots using
eliminate_dots <- function(x) {
  x<-gsub(".", "", x, ignore.case = FALSE, perl = FALSE, fixed = TRUE, useBytes = FALSE)
  return(x)
}
# geo.h.withoutDots <- eliminate_dots(geo.h)

# Then, if geo data does not include dots, use the following
# dt_new <- hier_separate(var=geo.h.withoutDots, dots=FALSE, len=c(2,1,2,3))
# dt_new
```

#### Add remaining variables

``` r
#creating sex variable with two different values
c_sex   <- as.character(c(1,2)) 
p_sex   <- c(0.45,0.55)
sex <- rep(c_sex, round(p_sex * N))
sex <- sample(sex, N)

#creating age variable with two different levels dividing the population into age groups in ranges of fifteen and of five years
set.seed(132) 
c_age.m <-  c('1.1.',  '1.2.',  '1.3.',
              '2.1.',  '2.2.',  '2.3.',
              '3.1.',  '3.2.',  '3.3.',  '3.4.',
              '4.1.',  '4.2.',  '4.3.',
              '5.1.',  '5.2.',  '5.3.',  '5.4.',
              '6.1.',  '6.2.',  '6.3.',  '6.4.')
p_age.m <- c(1,1,2,2,2,2,4,3,4,4,4,5,6,9,6,8,12,6,5,4,1)
p_age.m <- p_age.m/sum(p_age.m)
age.m <- rep(c_age.m, round(p_age.m * N))
age.m <- sample(age.m, N)

#creating year of arrival in country since 1980 until 2021 variable with three levels and ranges of five years on the second level
set.seed(134)
c_yae.h <- c('1.1.1.','1.1.2.',
             '1.2.1.','1.2.2.','1.2.3.','1.2.4.','1.2.5.',
             '1.3.1.','1.3.2.','1.3.3.','1.3.4.','1.3.5.',
             '1.4.1.','1.4.2.','1.4.3.','1.4.4.','1.4.5.',
             '1.5.',
             '1.6.',
             '1.7.',
             '1.8.',
             '1.9.',
             '2.',
             '3.')

p_yae.h <- c(2,2,2,6,8,12,5,3,4,2,1,3,4,6,6,7,2,12,8,9,6,1,1,2)
p_yae.h <- p_yae.h/sum(p_yae.h)
yae.h <- rep(c_yae.h, round(p_yae.h * N))
yae.h <- sample(yae.h, N)

id <- 1:N
```

The test data set contains four geographic hierarchy levels and in total three soziodemographic variables.

``` r
dat_test_orig <- data.table(id=id, 
                            dt_new,
                            age.m=age.m, 
                            yae.h=yae.h, 
                            sex=sex)
```

#### Output

``` r
dat_test_orig
>             id       geo.h nuts1 nuts2 nuts3 lau2 age.m  yae.h sex
>      1:      1 03.1.54.111    03     1    54  111  5.3. 1.2.4.   2
>      2:      2 03.1.53.516    03     1    53  516  6.3. 1.4.4.   1
>      3:      3 03.1.52.138    03     1    52  138  4.1.   1.7.   2
>      4:      4 01.0.58.402    01     0    58  402  5.1. 1.2.4.   1
>      5:      5 03.2.52.911    03     2    52  911  5.1. 1.4.4.   1
>     ---                                                           
> 819996: 819996 01.0.60.900    01     0    60  900  2.3. 1.2.5.   1
> 819997: 819997 03.1.55.317    03     1    55  317  4.2. 1.2.4.   1
> 819998: 819998 01.0.54.119    01     0    54  119  2.1.   1.5.   1
> 819999: 819999 03.4.53.364    03     4    53  364  5.4. 1.4.4.   2
> 820000: 820000 01.0.56.941    01     0    56  941  3.1. 1.2.4.   1
```

#### Change variables with dots

The original data includes `geo.h`, `yae.h` and `age.m` whose different levels are seperated by dots. This data gets falsified completely when record swapping is applied. The `recordSwap()` function can only handle numeric data, therefore all character strings must be compiled to numerics. Eliminating the dots is necessary using `give_position()`. `give_position()` saves each records' position in `c_age.m`, swaps are made with `dat_test` holding these indices as variables and afterwards changes the indices back into the proper age variable. Same problem as dots in `c_age.m` caused were caused for `c_yae.h` therefore using `give_position()` aswell. For more information on `give_position()` see [Appendix](#appendix).

``` r
age_new <- give_position(age.m, c_age.m)
yae_new <- give_position(yae.h, c_yae.h)
```

Use `age_new` and `yae_new` as variables instead of `age.m` and `yae.h` to avoid problems with dots. Now create the test data set including the new values and transferring all values to numerics. Exclude `geo.h` to save some time while swapping, as `geo.h` is only composited of `nuts1` to `lau2`.

``` r
dt_new[, nuts1 := as.numeric(nuts1)]
dt_new[, nuts2 := as.numeric(nuts2)]
dt_new[, nuts3 := as.numeric(nuts3)]
dt_new[, lau2 := as.numeric(lau2)]

dat_test <- data.table(id=id, 
                       dt_new[,-"geo.h"],
                       age_new=age_new, 
                       yae_new=yae_new, 
                       sex=as.numeric(sex))
```

That looks like this:

``` r
dat_test 
>             id nuts1 nuts2 nuts3 lau2 age_new yae_new sex
>      1:      1     3     1    54  111      16       6   2
>      2:      2     3     1    53  516      20      16   1
>      3:      3     3     1    52  138      11      20   2
>      4:      4     1     0    58  402      14       6   1
>      5:      5     3     2    52  911      14      16   1
>     ---                                                  
> 819996: 819996     1     0    60  900       6       7   1
> 819997: 819997     3     1    55  317      12       6   1
> 819998: 819998     1     0    54  119       4      18   1
> 819999: 819999     3     4    53  364      17      16   2
> 820000: 820000     1     0    56  941       7       6   1
```

### Apply targeted record swapping

One possibility to apply record swapping to `dat_test` is:

``` r
risk_variables <- 5
k_anonymity <- 3     
swaprate <- .6    
hid <- 0     
similar <- 6
hierarchy<-1:4
dat_swapped<-recordSwap(dat_test,similar,hierarchy,risk_variables,hid,k_anonymity,swaprate)
```

    >             id nuts1 nuts2 nuts3 lau2 age_new yae_new sex
    >      1:      1     3     1    54  111      16       6   2
    >      2:      2     3     3    57  562      20      16   1
    >      3:      3     3     1    52  138      11      20   2
    >      4:      4     3     3    53  290      14       6   1
    >      5:      5     3     2    52  911      14      16   1
    >     ---                                                  
    > 819996: 819996     3     3    58  317       6       7   1
    > 819997: 819997     3     1    55  317      12       6   1
    > 819998: 819998     1     0    62  483       4      18   1
    > 819999: 819999     3     4    53  364      17      16   2
    > 820000: 820000     3     3    59  500       7       6   1

<a id="here"></a>

Because the dots separating digits of `age.m`and `yae.h` had to be extracted earlier, they will now be added back again, being given the position in `c_age.m` and `c_yae.h`. Also `geo.h` will be included again.

``` r
dat_swapped_orig <- dat_swapped
dat_swapped_orig$age_new <- c_age.m[dat_swapped_orig$age_new]
dat_swapped_orig$yae_new <- c_yae.h[dat_swapped_orig$yae_new]

#adding geo data consisting of all four levels separated by dots
dat_swapped_orig[, nuts1 := ifelse(nchar(nuts1) == 1, paste0("0", nuts1), nuts1)  ]
dat_swapped_orig[, geo.h := paste(nuts1,nuts2,nuts3,lau2, sep=".")]

dat_swapped_orig <- dat_swapped_orig[, .(id, geo.h, nuts1, nuts2, nuts3, lau2, age.m=age_new, yae.h=yae_new, sex)]
```

Output is:

    >             id       geo.h nuts1 nuts2 nuts3 lau2 age.m  yae.h sex
    >      1:      1 03.1.54.111    03     1    54  111  5.3. 1.2.4.   2
    >      2:      2 03.3.57.562    03     3    57  562  6.3. 1.4.4.   1
    >      3:      3 03.1.52.138    03     1    52  138  4.1.   1.7.   2
    >      4:      4 03.3.53.290    03     3    53  290  5.1. 1.2.4.   1
    >      5:      5 03.2.52.911    03     2    52  911  5.1. 1.4.4.   1
    >     ---                                                           
    > 819996: 819996 03.3.58.317    03     3    58  317  2.3. 1.2.5.   1
    > 819997: 819997 03.1.55.317    03     1    55  317  4.2. 1.2.4.   1
    > 819998: 819998 01.0.62.483    01     0    62  483  2.1.   1.5.   1
    > 819999: 819999 03.4.53.364    03     4    53  364  5.4. 1.4.4.   2
    > 820000: 820000 03.3.59.500    03     3    59  500  3.1. 1.2.4.   1

Here the procedure was applied to `dat_test`

-   using hierarchy levels `nuts1` to `lau2`
-   using `age.m` as similarity variable
-   using `yae.h` as risk variable
-   setting the `k_anonymity` rule to `3`
-   setting the `swaprate` to `0.6`

Note: In R indexing usually starts with 1, but because this package was implemented in C++ indexing here begins with 0.

Output and tools for measuring information loss
-----------------------------------------------

The output consists of a data set of the same size including the same variables as the original test data set. The following approach should give information on how high the information loss is. There are a few different options:

### Measure information loss using `ck_cnt_measures()` function

#### Using `makeProblem()` function to create frequency vectors for original and swapped data set

The `makeProblem()` function delivers an opportunity to measure the loss of information by creating frequency tables. The function evaluates all possible cross combinations of the original data set as well as of the swapped data set and gives back the frequency of each combination of values. Afterwards one can apply `ck_cnt_measures()` to get different error metrics.

``` r
library(sdcTable)
library(xfun)
```

``` r
# preparation of the original data which is used for creating hierarchy
dat_test2 <- dat_test_orig[, .(nuts1=as.numeric(nuts1), age_m=age.m, yae_h=yae.h, sex)]

# defining the hierarchy manually (for nuts1 and sex)
dim.nuts1 <- hier_create(root = "Total", nodes = 1:3) 
dim.sex <- hier_create(root = "Total", nodes = 1:2)

# defining the hierarchy using category schemes (for age and yae)
c_age.m <-  c('1.1.',  '1.2.',  '1.3.',
              '2.1.',  '2.2.',  '2.3.',
              '3.1.',  '3.2.',  '3.3.',  '3.4.',
              '4.1.',  '4.2.',  '4.3.',
              '5.1.',  '5.2.',  '5.3.',  '5.4.',
              '6.1.',  '6.2.',  '6.3.',  '6.4.')

c_yae.h <- c('1.1.1.','1.1.2.',
             '1.2.1.','1.2.2.','1.2.3.','1.2.4.','1.2.5.',
             '1.3.1.','1.3.2.','1.3.3.','1.3.4.','1.3.5.',
             '1.4.1.','1.4.2.','1.4.3.','1.4.4.','1.4.5.',
             '1.5.',
             '1.6.',
             '1.7.',
             '1.8.',
             '1.9.',
             '2.',
             '3.')

dim.age <- hier_compute(inp = as.character(c_age.m), dim_spec = c(2,2), root = "Total", method = "len",as = "network")
dim.yae <- hier_compute(inp = as.character(c_yae.h), dim_spec = c(2,2,2), root = "Total", method = "len", as = "network")

# finally define the hierarchy list object
dimList <- list(nuts1=dim.nuts1, sex=dim.sex, age_m=dim.age, yae_h=dim.yae)


## Original Data

# stating the data table for the original data
prob <- makeProblem(data = dat_test2, dimList = dimList) 

# computing the data table for the original data
tab_orig <- sdcProb2df(prob, addDups = TRUE, addNumVars=FALSE, dimCodes="original")


## Swapped Data

# preparation of the swapped data
dat_swapped2 <- dat_swapped_orig[, .(nuts1=as.numeric(nuts1), age_m=age.m, yae_h=yae.h, sex)]

# stating the data table for the swapped data
prob_sw <- makeProblem(data = dat_swapped2, dimList = dimList) 

# computing the data table for the swapped data
tab_sw <- sdcProb2df(prob_sw, addDups = TRUE, addNumVars=FALSE, dimCodes="original")
```

#### Using cellKey method and create frequency vectors for original perturbed and swapped and perturbed data set

The future goal for statistical disclosure is to apply record swapping pre-tabular and afterwards cell key post-tabular. If that is done, one can easily take the `ck_cnt_measures()` function from the cellKey package to calculate the information loss by measuring the relative absolute distance, absolute distances of square roots as well as the cumulated distance and some other values. One option to apply the cell key method using the cellKey package from github is the following.

``` r
library(cellKey)
library(ptable)

#perturbation parameters for count variables
params_cnts <- ck_params_cnts(
  D = 2,
  V = 0.5,
  js = 0,
  pstay = 0.5,
  optim = 1,
  mono = TRUE
)

##Original data

#first apply to the original data and do some modifications
dat <- dat_test_orig[,.(nuts1=factor(as.numeric(nuts1)), age.m=factor(age.m), yae.h=factor(yae.h), sex)]

#create record keys
dat$rkey <- ck_generate_rkeys(dat = dat)

#create dimensions and hierarchy
dim.nuts1  <- hier_compute(inp = levels(unique(dat$nuts1)), dim_spec = c(1,1), root = "Total", method = "len", as = "network")
dim.age <- hier_compute(inp = levels(unique(dat$age.m)), dim_spec = c(2,2), root = "Total", method = "len",as = "network")
dim.yae <- hier_compute(inp = levels(unique(dat$yae.h)), dim_spec = c(2,2,2), root = "Total", method = "len", as = "network")
dim.sex <- hier_create(root = "Total", nodes = 1:2)

#create a named list
dimList <- list(nuts1=dim.nuts1, sex=dim.sex, age.m=dim.age, yae.h=dim.yae)

#define the cell key object
tab <- ck_setup(
  x = dat,
  rkey = "rkey",
  dims = dimList,
  w = NULL,
  countvars = NULL,
  params_cnts = params_cnts
)

tab$perturb(v = c("total"))
tab_pert <- tab$freqtab(v = c("total"))

##Swapped data

#application for the swapped data and do some modifications
dat <- dat_swapped_orig[,.(nuts1=factor(as.numeric(nuts1)), age.m=factor(age.m), yae.h=factor(yae.h), sex)]

#create record keys
dat$rkey <- ck_generate_rkeys(dat = dat)


#define the cell key object
tab <- ck_setup(
  x = dat,
  rkey = "rkey",
  dims = dimList,
  w = NULL,
  countvars = NULL,
  params_cnts = params_cnts
)

tab$perturb(v = c("total"))
tab_sw_pert <- tab$freqtab(v = c("total"))
```

#### Compare original data to swapped and to swapped and perturbed data

``` r
#original and swapped data
ck_cnt_measures(tab_orig$freq, tab_sw$freq)
> $overview
>      noise cnt           pct
>   1: -3396   1 0.00009920635
>   2: -3354   1 0.00009920635
>   3: -2574   1 0.00009920635
>   4: -2535   1 0.00009920635
>   5: -1932   1 0.00009920635
>  ---                        
> 622:  1950   1 0.00009920635
> 623:  2587   1 0.00009920635
> 624:  2632   1 0.00009920635
> 625:  3375   1 0.00009920635
> 626:  3408   1 0.00009920635
> 
> $measures
>       what       d1    d2    d3
>  1:    Min    0.000 0.000 0.000
>  2:    Q10    0.000 0.000 0.000
>  3:    Q20    0.000 0.000 0.000
>  4:    Q30    0.000 0.000 0.000
>  5:    Q40    2.000 0.010 0.136
>  6:   Mean   34.783   Inf 0.462
>  7: Median    4.000 0.022 0.247
>  8:    Q60    8.000 0.039 0.371
>  9:    Q70   15.000 0.077 0.520
> 10:    Q80   28.000 0.148 0.742
> 11:    Q90   60.000 0.290 1.132
> 12:    Q95  127.150 0.500 1.629
> 13:    Q99  554.150 3.000 3.405
> 14:    Max 3408.000   Inf 9.857
> 
> $cumdistr_d1
>       cat  cnt       pct
>   1:    0 3020 0.3032737
>   2:    1 3796 0.3812010
>   3:    2 4319 0.4337216
>   4:    3 4729 0.4748946
>   5:    4 5069 0.5090380
>  ---                    
> 434: 2632 9954 0.9995983
> 435: 3354 9955 0.9996987
> 436: 3375 9956 0.9997992
> 437: 3396 9957 0.9998996
> 438: 3408 9958 1.0000000
> 
> $cumdistr_d2
>            cat  cnt       pct
> 1:    [0,0.02] 4779 0.4799156
> 2: (0.02,0.05] 6355 0.6381804
> 3:  (0.05,0.1] 7385 0.7416148
> 4:   (0.1,0.2] 8456 0.8491665
> 5:   (0.2,0.3] 8991 0.9028921
> 6:   (0.3,0.4] 9274 0.9313115
> 7:   (0.4,0.5] 9500 0.9540068
> 8:   (0.5,Inf] 9958 1.0000000
> 
> $cumdistr_d3
>            cat  cnt       pct
> 1:    [0,0.02] 3079 0.3091986
> 2: (0.02,0.05] 3299 0.3312914
> 3:  (0.05,0.1] 3673 0.3688492
> 4:   (0.1,0.2] 4538 0.4557140
> 5:   (0.2,0.3] 5425 0.5447881
> 6:   (0.3,0.4] 6161 0.6186985
> 7:   (0.4,0.5] 6818 0.6846756
> 8:   (0.5,Inf] 9958 1.0000000
> 
> $false_zero
> [1] 62
> 
> $false_nonzero
> [1] 88
> 
> $exclude_zeros
> [1] TRUE

#original and swapped and perturbed data
ck_cnt_measures(tab_orig$freq, tab_sw_pert$puwc)
> $overview
>      noise cnt           pct
>   1: -3396   1 0.00009920635
>   2: -3353   1 0.00009920635
>   3: -2574   1 0.00009920635
>   4: -2535   1 0.00009920635
>   5: -1931   1 0.00009920635
>  ---                        
> 624:  1950   1 0.00009920635
> 625:  2587   1 0.00009920635
> 626:  2632   1 0.00009920635
> 627:  3375   1 0.00009920635
> 628:  3409   1 0.00009920635
> 
> $measures
>       what       d1    d2    d3
>  1:    Min    0.000 0.000 0.000
>  2:    Q10    0.000 0.000 0.000
>  3:    Q20    1.000 0.000 0.004
>  4:    Q30    1.000 0.003 0.031
>  5:    Q40    2.000 0.011 0.137
>  6:   Mean   35.022   Inf 0.467
>  7: Median    4.000 0.022 0.250
>  8:    Q60    8.000 0.039 0.371
>  9:    Q70   15.000 0.077 0.531
> 10:    Q80   28.000 0.149 0.744
> 11:    Q90   61.000 0.295 1.155
> 12:    Q95  128.600 0.500 1.634
> 13:    Q99  555.880 2.907 3.410
> 14:    Max 3409.000   Inf 9.857
> 
> $cumdistr_d1
>       cat  cnt       pct
>   1:    0 1875 0.1888408
>   2:    1 3660 0.3686172
>   3:    2 4280 0.4310605
>   4:    3 4690 0.4723537
>   5:    4 5011 0.5046833
>  ---                    
> 438: 2632 9925 0.9995971
> 439: 3353 9926 0.9996979
> 440: 3375 9927 0.9997986
> 441: 3396 9928 0.9998993
> 442: 3409 9929 1.0000000
> 
> $cumdistr_d2
>            cat  cnt       pct
> 1:    [0,0.02] 4761 0.4795045
> 2: (0.02,0.05] 6337 0.6382314
> 3:  (0.05,0.1] 7372 0.7424715
> 4:   (0.1,0.2] 8421 0.8481217
> 5:   (0.2,0.3] 8955 0.9019035
> 6:   (0.3,0.4] 9238 0.9304059
> 7:   (0.4,0.5] 9452 0.9519589
> 8:   (0.5,Inf] 9929 1.0000000
> 
> $cumdistr_d3
>            cat  cnt       pct
> 1:    [0,0.02] 2692 0.2711250
> 2: (0.02,0.05] 3237 0.3260147
> 3:  (0.05,0.1] 3642 0.3668043
> 4:   (0.1,0.2] 4540 0.4572464
> 5:   (0.2,0.3] 5381 0.5419478
> 6:   (0.3,0.4] 6128 0.6171820
> 7:   (0.4,0.5] 6759 0.6807332
> 8:   (0.5,Inf] 9929 1.0000000
> 
> $false_zero
> [1] 80
> 
> $false_nonzero
> [1] 77
> 
> $exclude_zeros
> [1] TRUE

# the mean of distances between frequencies of the original and the swapped data set 
mean(tab_orig$freq - tab_sw$freq)
> [1] 0

# the mean of distances between frequencies of the original and the swapped and perturbed data set
mean(tab_orig$freq - tab_sw_pert$puwc)
> [1] -0.00843254
```

### Counting number of changed records from vectors

Another option to measure the amount of changed information is to simply write the data sets as vectors. Having both dataframes as numeric vectors offers the chance to count the number of records where both vectors differ and divide the sum by their length. That works with the function `changed_pct()`.

``` r
changed_pct <- function(v1, v2) {
  stopifnot(all.equal(dim(v1), dim(v2)))
  v1 <- unlist(v1)
  v2 <- unlist(v2)
  sum(v1 != v2) / length(v2)
}
```

``` r
changed_pct(dat_test, dat_swapped)
> [1] 0.2363067
```

In case of the example that delivers that in total 23.6306707 % of all records were changed.

### Count number of records whose relation between `id` and `lau2`

An option for checking on how much data was actually swapped is to simply compare the proportion between `id` and `lau2` from the original data set to the swapped data set. Counting in how many records this proportion changed delivers the number of swaps made. Divided by N this delivers the rate of actual made swaps. This helps to check how close the number of swaps made is to the defined swaprate.

``` r
sum(((dat_test$id)/(dat_test$lau2))!=((dat_swapped$id)/(dat_swapped$lau2)))/N
> [1] 0.5983683
```

So obviously with 59.8368293 % the percentage of swaps made is quite close to the given swaprate of 60 %.

### Summary

The function was applied on randomly generated household data with `820000` households. The parameter `swaprate` was set to `0.6` and `k_anonymity` to `3`. Targeted record swapping was used on four hierarchy levels having a total of 3770 different geographic areas on the lowest level. For swapping the runtime in this case was `6.4653391` min.

Appendix
--------

Function `hier_separate()` splits the data into its hierarchical levels and delivers separate columns as output in a data table.

`hier_separate()`:

``` r
hier_separate <- function(var, col.names = c("nuts1","nuts2", "nuts3","lau2"), len, dots=TRUE){
  
  if (dots){
    das <- data.table(vname=var, as.data.frame(var) %>% tidyr::separate(var, col.names ))
  } else {
    pat <- paste0("(.{",len,"})", collapse = "")
    das <- data.table(vname=var,as.data.frame(var) %>% extract(var, into=col.names, pat))
  }
  
  names(das)[1] <- deparse(substitute(var))
  return(das)
}
```

`give_position()` takes the vector holding data with dots and checks for each row which position the value has in the vector. Then it saves the position instead of the original value before swapping, to avoid problems with data including dots. After swapping one can rechange the value by taking the vector holding the position and read the value it holds for each position in the data set. See [here](#here) for more details.

`give_position()`:

``` r
give_position <- function(variable, codes) {
  out <- unlist(sapply(1:length(variable), 
                       function(x) {
                         query <- codes %in% variable[x]
                         ifelse(all(query), NA, which(query))
                       },
                       simplify = TRUE ))
  
  if (any(is.na(out))) 
    cat(paste0("Input 'codes' is not properly defined. Missing code for: ", variable[is.na(out)], " \n"))
  return(out)
}
```

The function `eliminate_dots()` is a simple tool to delete dots inside variables, like used for `geo.h` above to avoid problems some functions have if the input is not a numeric variable.

`eliminate_dots()`:

``` r
eliminate_dots <- function(x) {
  x<-gsub(".", "", x, ignore.case = FALSE, perl = FALSE, fixed = TRUE, useBytes = FALSE)
  return(x)
}
```
