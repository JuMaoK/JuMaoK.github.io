---
layout: post
title: "Binary Search"
description: "Binary Search using bisect modual"
tags: [Algorithms]
---

> A binary search divides a range of values into halves, and continues to narrow down the field of search until the unknown value is found. It is the classic example of a "divide and conquer" algorithm.
>
> As an analogy, consider the children's game "[guess a number](https://rosettacode.org/wiki/Guess_the_number/With_feedback)." The scorer has a secret number, and will only tell the player if their guessed number is higher than, lower than, or equal to the secret number. The player then uses this information to guess a new number.
>
> As the player, an optimal strategy for the general case is to start by choosing the range's midpoint as the guess, and then asking whether the guess was higher, lower, or equal to the secret number. If the guess was too high, one would select the point exactly between the range midpoint and the beginning of the range. If the original guess was too low, one would ask about the point exactly between the range midpoint and the end of the range. This process repeats until one has reached the secret number.

```python
from bisect import bisect_left 

def binary_search(lst, x): 
​    i = bisect_left(lst, x) 
​    if i != len(lst) and lst[i] == x:   # ？ 
​        return i 
​    raise ValueError 
```