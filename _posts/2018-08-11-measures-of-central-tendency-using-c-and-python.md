---
layout: post
title: Measures of central tendency using C and Python
date: 2018-08-11
tags: [math, statistics, programming, python, c]
---

Calculating measures of central tendency in Python with Numpy and Scipy and in C with GNU Scientific Library.


## Arithmetic mean

The arithmetic mean is the sum of all numbers in a dataset divided by the length of the dataset.

![](https://wikimedia.org/api/rest_v1/media/math/render/svg/90330653b40adf032ea8e144f84d7eec1a88054d)


*For our example dataset we expect a result of 2.0.*

### Verbose

```python
a = [1.0, 1.4, 5.6, 1.1, 0.9]
r = sum(a)/len(a)
print r
```

### Python code

```python
import numpy as np

a = np.array([1.0, 1.4, 5.6, 1.1, 0.9])
r = np.mean(a)
print r
```

### C code

```c
/*
 * gsl_mean.c
 * gcc -L/usr/local/lib -lgsl -lgslcblas -lm gsl_mean.c -o gsl_mean
 */
 
#include <stdio.h>
#include <gsl/gsl_statistics_double.h>

int main(void)
{
  double const a[] = {1, 1.4, 5.6, 1.1, 0.9};

  /* double gsl_stats_mean(double const data[], size_t stride, size_t n) */
  double r = gsl_stats_mean(a, 1, 5);
  printf("%f\n", r);
}
```

## Median
The median is the value that separates the higher half from the lower half of a dataset.
If the length of the dataset is odd, the middle number is returned, if it's even the average of the two middle numbers is returned.

*For our example dataset we expect a result of 3.5.*

### Verbose

```python
a = [2.0, 1.0, 3.0, 4.0, 5.0, 6.0]
a.sort()
l = len(a)
if l % 2 == 0:
    r = sum(a[(l/2)-1:(l/2)+1]) / 2.0
else:
    r = a[(l-1)/2]
print r
```

### Python code

```python
import numpy as np

a = np.array([2.0, 1.0, 3.0, 4.0, 5.0, 6.0])
np.median(a)
```

### C code

```c
/* 
 * gsl_median.c
 * gcc -L/usr/local/lib -lgsl -lgslcblas -lm gsl_median.c -o gsl_median 
 */

#include <stdio.h>
#include <gsl/gsl_sort.h>
#include <gsl/gsl_statistics_double.h>

int main(void)
{
  double a[] = {2, 1, 3, 4, 5, 6};

  /* void gsl_sort(double * data, const size_t stride, size_t n) */
  gsl_sort(a, 1, 6);
  /* double gsl_stats_median_from_sorted_data(double const sorted_data[], size_t stride, size_t n) */
  double r = gsl_stats_median_from_sorted_data(a, 1, 6);
  printf("%f\n", r);
}
```

## Mode

The mode is the value in a dataset that appears most often.

*For our example dataset we expect a result of 5.*

### Verbose

See the C code.

### Python code

```python
import numpy as np
from scipy import stats

a = np.array([1, 2, 3, 4, 4, 4, 5, 5, 5, 5, 6])
stats.mode(a) # returns mode array and count array
```

### C code

```c
/*
 * mode.c
 * gcc mode.c -o mode
 */

#include <stdio.h>

/* return the smallest mode from a sorted dataset */
double mode_from_sorted_data(double const data[], size_t n)
{
    double lead = 0, curr = data[0];
    int lc = 0, cc = 1; // lead count, current count

    for (int i = 1; i < n; i++) {
        if (data[i] == curr) {
            cc++;
        } else {
            if (cc > lc) {
                lc = cc;
                lead = curr;
            }
            curr = data[i];
            cc = 1;
        }
    }
    return (cc > lc) ? curr : lead;
}

int main(int argc, char **argv)
{
    double const a[] = {1, 2, 3, 4, 4, 4, 5, 5, 5, 5, 6};
    double r = mode_from_sorted_data(a, 11);
    printf("%f\n", r);
}
```

## Geometric mean

A Pythagorean mean along with arithmetic mean and harmonic mean. The geometric mean is always the middle of the three.

The geometric mean is the *n*th root of the product of the *n* members of a dataset.

![](https://wikimedia.org/api/rest_v1/media/math/render/svg/026cae6801f672b9858d55935ec7397183dc3a36)

Be aware that overflow can happen if we calculate the product of many and/or large numbers!

*For our example dataset we expect a result of ~4.3.*

### Verbose

```python
a = [1.25, 2, 3, 4, 5, 6, 7, 8, 10]
r = 1
for n in a:
    r = r * n
r = r**(1.0/len(a))
print r
```


### Python code

The `stats.gmean` function reduces chances of overflow by mapping numbers to a log domain.

```python
import numpy as np
from scipy import stats

a = np.array([1.25, 2, 3, 4, 5, 6, 7, 8, 10])
r = stats.gmean(a)
print r
```

### C code

To understand the code below you need to be aware that

![](https://wikimedia.org/api/rest_v1/media/math/render/svg/209a47a2b2696e9b5bada5d7c4f4340cc905afec)

By using logarithmic identities to transform the formula we reduce the chances of large and/or many numbers causing an overflow.

```c
/*
 * geomean.c
 * gcc -L/usr/local/lib -lgsl -lgslcblas -lm geomean.c -o geomean
 */

#include <stdio.h>
#include <gsl/gsl_sf_log.h>
#include <gsl/gsl_sf_exp.h>

double geometric_mean(double const data[], size_t n)
{
    double tmp = 0.0;

    for (int i = 0; i < n; i++)
        tmp += gsl_sf_log(data[i]);
    return gsl_sf_exp(tmp/n);
}

int main(void)
{
  double const a[] = {1.25, 2, 3, 4, 5, 6, 7, 8, 10};
  double r = geometric_mean(a, 9);
  printf("%f\n", r);
}
```

## Harmonic mean

A Pythagorean mean along with arithmetic mean and geometric mean. The harmonic mean is always the smallest of the three.

The harmonic mean is the reciprocal of the arithmetic mean of the reciprocals of the values in a dataset.

![](https://wikimedia.org/api/rest_v1/media/math/render/svg/753130a05a1fab890e5785924b5bdbb5f97c8b6a)

*For our example dataset we expect a result of ~6.35.*

### Verbose

```python
a = [1000, 12, 21, 99, 1.55]
tmp = 0
for n in a:
    tmp += n**-1
r = (tmp/len(a))**-1
print r
```

### Python code
```python
import numpy as np
from scipy import stats

a = np.array([1000, 12, 21, 99, 1.55])
r = stats.hmean(a)
print r
```

### C code

```c
/*
 * harmean.c
 * gcc -L/usr/local/lib -lgsl -lgslcblas -lm harmean.c -o harmean
 */

#include <stdio.h>
#include <gsl/gsl_statistics_double.h>
#include <gsl/gsl_sf_pow_int.h>

double harmonic_mean(double const data[], size_t n)
{
    double tmp = 0.0;

    for (int i = 0; i < n; i++)
        tmp += gsl_sf_pow_int(data[i], -1);
    return gsl_sf_pow_int(tmp/n, -1);
}


int main(void)
{
  double const a[] = {1000, 12, 21, 99, 1.55};

  double r = harmonic_mean(a, 5);
  printf("%f\n", r);
}
```

## Weighted mean

An arithmetic mean where the data points contribute to the final average according to a given weight instead of contributing equally.

*For our example dataset we expect a result of 3.76.*

### Verbose

```python
a = [1, 2, 3, 4, 5, 6, 7, 8, 9]
w = [0.1, 0.2, 0.4, 0.01, 0.09, 0.05, 0.03, 0.02, 0.1]

tmp = 0
for n in zip(a,w):
    tmp += n[0] * n[1]
r = tmp/sum(w)
print r
```

### Python code

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5, 6, 7, 8, 9])
w = np.array([0.1, 0.2, 0.4, 0.01, 0.09, 0.05, 0.03, 0.02, 0.1])

r = np.average(a, weights=w)
print r
```

### C code

```c
/*
 * gsl_wmean.c 
 * gcc -L/usr/local/lib -lgsl -lgslcblas -lm gsl_wmean.c -o gsl_wmean
 */

#include <stdio.h>
#include <gsl/gsl_statistics_double.h>

int main(void)
{
    double const a[] = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    double const w[] = {0.1, 0.2, 0.4, 0.01, 0.09, 0.05, 0.03, 0.02, 0.1};

    /* double gsl_stats_wmean(const double w[], size_t wstride, const double data[], size_t stride, size_t n) */
    double r = gsl_stats_wmean(w, 1, a, 1, 9);
    printf("%f\n", r);
}
```
