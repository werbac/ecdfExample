ecdfExample
===========

An example evaluating an empirical cdf in R, R with Rcpp and in Julia

## Background

A Ph.D student working with Sunduz Keles was preparing an `R` package for
analysis of the results of a high throughput biological assay.  The
final step - creating p-values for the observed response from a
reference sample - was taking an inordinate amount of time in their `R`
code. 

The samples were large: about 6,000,000 numeric values in the
reference distribution sample and about 2,000,000 in the observed
sample.  The reference distribution was not a Gaussian but we will
generate samples from this distribution for illustration.

```r
set.seed(1234321)  # for reproducibility of results
ref <- rnorm(6000000L)
samp <- rnorm(2000000L)
```


The important step is determining what portion of the reference sample
is less than each element of the observed sample.  For a single
element in the observed sample a vectorized comparison would be

```r
(numlt1 <- sum(ref < samp[1]))
```

```
## [1] 4174960
```

```r
(prop1 <- numlt1/length(ref))
```

```
## [1] 0.6958
```

 
An individual evaluation like this is fast but 2,000,000 such evaluations took a long time.
For illustration we consider a subsample of size 100.

```r
numltr <- function(obs, ref) {
    nobs <- length(obs)
    ans <- integer(nobs)
    for (i in seq_len(nobs)) ans[i] <- sum(ref < obs[i])
    ans
}
system.time(numlt1 <- numltr(samp[1:100], ref))
```

```
##    user  system elapsed 
##   7.988   0.732   8.778
```


On this computer a single evaluation takes about 0.1 seconds.  Obviously 2,000,000 evaluations will be slow.

## Using `std::lower_bound` in `C++`

Those familiar with algorithms like binary search will realize that by sorting the reference sample and using a binary search the number of comparisons can be reduced enormously.  In C++ the `std::lower_bound` algorithm from the STL (Standard Template Library) returns the index of the last element in the sorted reference sample that is less than a given value.

The file `ecdf.cpp` is a C++ source file containing
```c++
#include <Rcpp.h>
using namespace Rcpp;
//[[Rcpp::export]]
IntegerVector cpplb(NumericVector samp, NumericVector sref) {
  int nobs = samp.size();
  IntegerVector ans(nobs);
  for (int i = 0; i < samp.size(); ++i)
    ans[i] = std::lower_bound(sref.begin(), sref.end(), samp[i]) - sref.begin();
  return ans;
}
```

The `sourceCpp` function in the `Rcpp` package is used compile this C++ function and make it visible as an `R` function.   In [Rstudio](http://www.rstudio.com) this operation is even easier because the user can open the `.cpp` file and click the "source on save" button so that every time the file is saved it is loaded with `sourceCpp` into the current `R` sesssion.

```r
library(Rcpp)
sourceCpp("ecdf.cpp")
system.time(sref <- sort(ref))
```

```
##    user  system elapsed 
##   1.448   0.008   1.458
```

```r
system.time(numlb <- cpplb(samp[1:100], sref))
```

```
##    user  system elapsed 
##       0       0       0
```

```r
str(numlt1)
```

```
##  int [1:100] 4174960 3589165 560670 86823 640624 4314046 5096987 2202714 5477351 3914446 ...
```

```r
str(numlb)
```

```
##  int [1:100] 4174960 3589165 560670 86823 640624 4314046 5096987 2202714 5477351 3914446 ...
```

```r
all.equal(numlt1, numlb)
```

```
## [1] TRUE
```

The `cpplb` function is too fast to time on only 100 evaluations.  We can actually run the entire 
sample in a few seconds with this function.

```r
system.time(numlt2 <- cpplb(samp, sref))
```

```
##    user  system elapsed 
##   1.472   0.000   1.497
```

## Using a sorted sample in `C++`

Even with a binary search we are still performing over 20 comparisons for each element of `samp`.
If the elements of `samp` are examined in increasing order, however, we can start from the last
lower bound and get the next lower bound after about 3 comparisons, on average.  The C++ code is
a bit more complicated and it took me several tries to get it right,
```c++
//[[Rcpp::export]]
IntegerVector cppcp(NumericVector samp, NumericVector ref, IntegerVector ord) {
  int nobs = samp.size();
  IntegerVector ans(nobs);
  for (int i = 0, j = 0; i < nobs; ++i) {
    int ind(ord[i] - 1); // C++ uses 0-based indices
    double ssampi(samp[ind]);
    while (ref[j] < ssampi && j < ref.size()) ++j;
    ans[ind] = j;     // j is the 1-based index of the lower bound
  }
  return ans;
}
```
but it executes very quickly

```r
system.time(ord <- order(samp))
```

```
##    user  system elapsed 
##   2.416   0.004   2.453
```

```r
system.time(numlt3 <- cppcp(samp, sref, ord))
```

```
##    user  system elapsed 
##   0.240   0.000   0.239
```

```r
all.equal(numlt3, numlt2)
```

```
## [1] TRUE
```


This process is reasonably convenient, especially with the support available in [Rstudio](http://rstudio.org) but it does involve two languages and interface code.

## Using `searchsortedlast` in `Julia`
[Julia](http://julialang.org) provides speed comparable to compiled `C` or `C++` code within a dynamic, interactive language.  The `searchsortedlast` function in `Julia` provides the results of `std::lower_bound`.  After transferring the data from `R` to `Julia` and sorting the reference sample we can create the result using a "comprehension"
```julia
julia> using DataFrames

julia> dat = read_rda("/home/bates/Rproj/ecdfExample/data.rda");
Written by version 2.15.2
Minimal R version: 2.3.0

julia> sref = dat["ref"].data;

julia> @elapsed sref = sort!(sref)  # an in-place sort as original is no longer needed
0.3402528762817383

julia> samp = dat["samp"].data;

julia> numlt2 = dat["numlt2"].data;

julia> dump(sref)
Array(Float64,(6000000,)) [-4.92784, -4.86842, -4.85339, -4.83884  …  4.8352, 4.87281, 4.94616, 4.95661, 4.97419]

julia> dump(samp)
Array(Float64,(2000000,)) [0.512579, 0.248878, -1.31952, -2.18544, -1.24358  …  0.716465, 0.28251, 0.0592095, 0.18775]

julia> dump(numlt2)
Array(Int32,(2000000,)) [4174960, 3589165, 560670, 86823, 640624  …  4837702, 4578656, 3667005, 3140388, 3445791]

julia> numlt4 = [searchsortedlast(sref, s) for s in samp];

julia> @elapsed [searchsortedlast(sref, s) for s in samp]
2.5520858764648438

julia> @assert (all(numlt4 .== numlt2))

```
The speed is comparable to the speed of the method using `Rcpp` and `std::lower_bound`.

As is common in `Julia` the `searchsortedlast` function is written in `Julia`.
```julia
# index of the last value of vector v that is less than or equal to x;
# returns 0 if x is less than all values of v.
function searchsortedlast(o::Ordering, v::AbstractVector, x, lo::Int, hi::Int)
    lo = lo-1
    hi = hi+1
    while lo < hi-1
        m = (lo+hi)>>>1
        if lt(o, x, v[m])
            hi = m
        else
            lo = m
        end
    end
    return lo
end

for s in {:searchsortedfirst, :searchsortedlast}
    @eval begin
        $s(o::Ordering, v::AbstractVector, x) = $s(o, v, x, 1, length(v))
        $s{O<:Ordering}(::Type{O}, v::AbstractVector, x) = $s(O(), v, x)
        $s(v::AbstractVector, x) = $s(Forward(), v, x)
    end
end
```
(The operator `>>>` is an arithmetic right shift so `(lo+hi)>>>1` is Geek for the integer division of `(lo+hi)` by 2.)

This definition covers a wide range of vector types and comparison operators succinctly.

## Using a sorted sample in `Julia`
A similar definition and timing of a function like `cppcp` is
```julia
julia> ord = sortperm(samp);

julia> @elapsed sortperm(samp)
0.9143240451812744

julia> dump(ord)
Array(Int64,(2000000,)) [31333, 440792, 788079, 1496204, 449572, 297156  …  11333, 963459, 1204917, 883672, 1331988]

julia> function julcp(samp::Vector{Float64}, sref::Vector{Float64}, ord::Vector{Int})
           j = 1
           ans = similar(ord)
           for i in 1:length(samp)
               while (sref[j] < samp[ord[i]] && j <= length(sref)) j += 1 end
               ans[ord[i]] = j - 1
           end
           ans
       end

julia> numlt5 = julcp(samp, sref, ord);

julia> @elapsed julcp(samp, sref, ord)
0.24966001510620117

julia> @assert all(numlt5 .== numlt2)
```

# Summary

When I first was asked speeding up this calculation, I didn't think of it as evaluating an "ecdf".  Had I done so, I would have realized that there is an `ecdf` function in the `stats` package and I could have saved myself some trouble.  It takes a bit of reading to decide that the `ecdf` function applied to the reference sample returns a function that is applied to the observed sample to get the quantiles.

```r
system.time(ed <- ecdf(ref))
```

```
##    user  system elapsed 
##   5.513   0.284   5.818
```

```r
system.time(quant <- ed(samp))
```

```
##    user  system elapsed 
##   1.856   0.032   1.895
```

The function itself is hidden by a class

```r
ed
```

```
## Empirical CDF 
## Call: ecdf(ref)
##  x[1:6000000] = -4.9, -4.9, -4.9,  ...,   5,   5
```

```r
unclass(ed)
```

```
## function (v) 
## .C(C_R_approxfun, as.double(x), as.double(y), n, xout = as.double(v), 
##     as.integer(length(v)), as.integer(method), as.double(yleft), 
##     as.double(yright), as.double(f), NAOK = TRUE)$xout
## <bytecode: 0x9bd4ff0>
## <environment: 0x9bd3d10>
## attr(,"call")
## ecdf(ref)
```

This performance would undoubtably been acceptable but I didn't think to look for the function.  Also, if you want to understand how it is evaluating the quantiles you need to wade your way through the `C` functions in the file `./src/src/library/stats/src/approx.c` in the `R` source tree.  This is not impossibly difficult but neither is it easy.

Later ChenLiang Xu reminded me of the `findInterval` function in `R` which, as the name implies, finds the index of the interval in a sorted reference sample for each element of an observed sample.  It looks like

```r
system.time(numlt4 <- findInterval(samp, sref))
```

```
##    user  system elapsed 
##   5.148   0.156   5.324
```

```r
str(numlt4)
```

```
##  int [1:2000000] 4174960 3589165 560670 86823 640624 4314046 5096987 2202714 5477351 3914446 ...
```

```r
all.equal(numlt4, numlt3)
```

```
## [1] TRUE
```

A comparison of the Rcpp and Julia execution times on each stage is:

operation|R/Rcpp | Julia
-----------|-------------|---------
sort reference sample|1.456|0.340
quantiles by binary search|1.452|2.552
permutation to order the observed sample|2.475|0.914
sequential search on ordered sample|0.244|0.249

I should note that there is a `sort` templated function in Standard Template Library (STL) for `C++` which makes it easy to write a function `cppsort` for `R`
```c++
//[[Rcpp::export]]
NumericVector cppsort(NumericVector v) {
    NumericVector sv(clone(v));
    std::sort(sv.begin(), sv.end());
    return sv;
}
```
I wasn't able to find a native `sort` function in `Rcpp` so this function is specific to numeric vectors.  It should be possible to write a generic sort for vector objects in `R` (numeric, integer and perhaps character vectors) but that is beyond my skill in C++ template metaprogramming.

This function is faster than the `sort` function in `R` but still not as fast as the Julia `sort` function.

```r
all.equal(sref, cppsort(ref))
```

```
## [1] TRUE
```

```r
system.time(cppsort(ref))
```

```
##    user  system elapsed 
##   0.652   0.012   0.667
```

