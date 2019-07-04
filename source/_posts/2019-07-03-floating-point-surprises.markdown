---
layout: post
title: "Floating-Point Surprises"
date: 2019-07-03 15:57:02 -0700
comments: true
categories: 
---

The single-precision floating-point or double-precision floating-point has finite precision so [loss of significance](https://en.wikipedia.org/wiki/Loss_of_significance) can happen and cause surprises.

<!-- more -->

Let's take float, which has 23 bits mantissa, as an example. `198705381` as an integer has the binary representation `00001011 11011000 00000000 11100101`. `198705381.0f` as a float has the binary representation `0 10011010 01111011000000000001110`. Here a round-down happens due to the default IEEE 754 rounding mode `round to the nearest, ties to even` and `198705381.0f` is rounded down to `198705376.0f`.

``` cpp
std::cout << std::setprecision(16) << 198705381.0f << "\n"; // output is 198705376
```

`199698905` as an integer has the binary representation `00001011 11100111 00101001 11011001`. `199698905.0f` as a float has the binary representation `0 10011010 01111100111001010011110`. Here a round-up happens and `199698905.0f` is rounded up to `199698912.0f`.

``` cpp
std::cout << std::setprecision(16) << 199698905.0f << "\n"; // output is 199698912
```

Due to the loss of significance, many surprises can happen:

```
std::cout << 198705381.0f - 198705380.0f << "\n";  // output is 0 since they are both rounded down to the same value

std::cout << 198705381.0f * 1.005f - 199698905.0f << "\n"; // output is negative (should be positive if we use double instead of float) since one is rounded down and the other is rounded up
```

### References
[1] https://benjaminjurke.com/content/articles/2015/loss-of-significance-in-floating-point-computations <br/>
[2] https://www.h-schmidt.net/FloatConverter/IEEE754.html
