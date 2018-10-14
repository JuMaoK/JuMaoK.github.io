---
layout: post
title: "Egg Dropping problem"
description: "假设n个理想的蛋，在一座塔的第k层扔下去会碎掉，n层以下怎么扔都能保持完好状态。怎样用有限的蛋，尽可能快地找出楼层k."
tags: [Algorithms]
---

* 目录
{:toc .toc}
---

### reference
> [Egg Dropping](https://brilliant.org/wiki/egg-dropping/#see-also)

# 2 Eggs, 100 Floors
二分法在这里不可行。

最优是从14层开始丢。

| 1st egg | if the first egg breaks --> 2nd egg | drops   |
| ------- | ----------------------------------- | ------- |
| 14      | 1-2-3-4-5-6-7-8-9-10-11-12-13       | 1+13=14 |
| 27      | 15-16-17-18-19-20-21-22-23-24-25-26 | 2+12=14 |
| ...     |                                     |         |
| 95      | 91-92-93-94                         | 10+4=14 |
| 99      | 100                                 | 11+1=12 |

这种办法，大部分情况都只需最多14次就能找到n。

# 2 Eggs, k Floors

> A good way to start this problem is to ask "Are we able to cover all the floors with  x drops?"The crucial point here is understanding that we are not trying to find the minimum number of drops knowing the best strategy; actually, we are trying to find the best strategy supposing that the minimum number of drops is , and we have to determine if covering all the floors using at most  attempts is possible or not.
> 如引述，解决这个问题的思路从最小次数变为覆盖所有楼层（k层）的最小次数。

假设从x层开始，覆盖x次，总共可以覆盖：

$$ x + (x-1)  + (x-2) + (x-3) + ... + 2 + 1 (+ 0) = \frac {x(x+1)} {2} $$

层楼。

为使覆盖楼层大于等于k层，即：

$$ \frac {x(x+1)} {2} \geq k $$

解方程，得：

$$ x = \frac {-1 + \sqrt{1+8k}} {2} $$

注意：x必须为整数

如2蛋100层的情况，x = 13.65 = 14

# N Eggs, k Floors

问题要求我们用最少的鸡蛋，覆盖所有k楼层（在最坏的情况下）。

要比较有效率地解决这个问题，需要先知道以下的数学例子：

**组合 (Combinations)**

$$ C\binom{n}{k} = C(n, k) = \binom{n}{k} = \frac{n!}{k!(n-k)!} $$

**杨辉三角 (Pascal triangle)**

![](https://ds055uzetaobb.cloudfront.net/image_optimizer/78406ed1c4b37b62d760853d723b91a300f2ce62.png)

**并且**

![](https://ds055uzetaobb.cloudfront.net/image_optimizer/a837ebdd3d308279fbe02ff8b99355b7692a387e.png)

可找到规律：

$$ C(n, k) = C(n-1, k) + C(n-1, k-1) $$

$$ C(n, 0) = \frac{n!}{0!(n-0)!} = 1$  and  $C(n, n) = \frac{n!}{n!(n-n)!} = 1 $$

## 思路

设方程$f(d, n)$，代表用被覆盖的楼层数。

其中，n代表剩余的鸡蛋，d代表在最坏情况下，剩下所需投下的次数。

*（可以用2蛋100层的例子简化思路来理解）*

1. 蛋碎了：剩下$f(d-1, n-1)$层楼需要被覆盖；
2. 蛋不碎：剩下$f(d-1, n)$层楼需要被覆盖

所以：

$$ f(d, n) = 1 + f(d-1, n-1) + f(d-1, n) $$

为找到这个方程，我们首先引入一个辅助方程$$ g(d, n) $$:

$$ g(d, n) = f(d, n+1) - f(d, n) $$

变形可得：

$$
\begin{align}g(d, n) & = f(d, n+1)-f(d,n)\\&= f(d-1,n+1)+f(d-1,n)+1-f(d-1,n)-f(d-1,n-1)-1\\&=[f(d-1, n+1)-f(d-1,n)] + [f(d-1,n)-f(d-1,n-1)]\\&=g(d-1,n) + g(d-1,n-1)\end{align}
$$

可发现这个公式与上面combinas的公式一模一样，不妨把它写作

$$ g(d,n)=\binom{d}{n} $$

但还有个问题，就是$f(0,n)$与$g(0,n)$都等于0，所以需要重设公式：

$$ g(d,n)=\binom{d}{n+1} $$

具体验证就不做了。。。

回到$f(d,n)$，求和：

$$
\begin{align}f(d,n)&=[f(d,n)-f(d,n-1)]\\&+[f(d,n-1)-f(d,n-2)]\\&+\cdots\\&+[f(d,1)-f(d,0)]\\&+f(d,0)\end{align}
$$

因为$$ f(d,0)=0 $$

$$ f(d,n) =g(d,n-1)+g(d,n-2) + \cdots + g(d,0) $$

并且

$$ g(d,n)=\binom{d}{n+1} $$

所以：

$$ g(d,n-1)+g(d,n-2) + \cdots + g(d,0) = \binom{d}{n}+\binom{d}{n-1}+\cdots+\binom{d}{1} $$

最后：

$$ f(d,n) = \sum_{i=0}^{N}{\binom{d}{i}} $$

**求解**

我们现已至$f(d,n)$的公式，接下来要用它找到最小的扔下次数d。

因为$f(d,n)$代表在最坏情况下，用n只蛋在不超过d次内可以覆盖的楼层数。

对于k层楼而言，只需要：

$$ f(d,n)\geq k $$，即

$$ \sum_{i=0}^{N}{\binom{d}{i}} \geq k $$

即告完成！

## 代码(有bug)

```python
from math import factorial


def comb(n, k):
    return factorial(n) / (factorial(k) * factorial(n-k))


def cover_floor(drops, n_eggs):
    return sum([comb(drops, N) for N in range(n_eggs+1)])


def answer(n_eggs, k_floors):
    hi = k_floors
    lo = 0
    while lo <= hi:
        mid = (lo + hi) // 2
        if cover_floor(mid, n_eggs) > k_floors:
            hi = mid - 1
        elif cover_floor(mid, n_eggs) < k_floors:
            lo = mid + 1
    return lo
```

