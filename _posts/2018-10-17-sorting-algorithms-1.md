---
layout: post
title: "Sorting algorithms - 1"
tags: [Algorithms]
---

quick sort & merge sort

* 目录
{:toc .toc}
------

### Reference

> [quicksort](https://rosettacode.org/wiki/Sorting_algorithms/Quicksort)
>
> [mergesort](https://rosettacode.org/wiki/Sorting_algorithms/Merge_sort)



# Quick sort

### Background

Quick sort采取divide-and-conquer策略对序列进行排序。

#### performence

quick sort是以速度闻名的排序算法，但并不稳定，主要取决于pivot的选择。理想情况是pivot将序列分成接近等长的两段子序列，这时候时间复杂度为`O(n log n)`；但如果其中一个子序列是空序列，时间复杂度将为`O(n^2)`。

#### Divide

1. 从序列中选一个元素做pivot（轴点）；
2. 通过与pivot比较大小，将序列分成两个子序列，其中一个所有元素都比pivot小，另一个反之。

#### Conquer

对分裂出的子序列递归地排序，最后将所有子序列重组。

### Python code

```python
def quicksort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[0]
    less = [i for i in arr[1:] if i < pivot]
    more = [i for i in arr[1:] if i > pivot]
    return quicksort(less) + [pivot] * arr.count(pivot) + quicksort(more)
```



# Merge sort

### background

Merge sort也是采取divide-and-conquer策略的排序算法。

#### Performance

$$ O(n log(n))$$in average and worst-case

$$ O(n)$$in best case 

在worst case下，merge sort比在average case下的quick sort 少约39%的比较运算。[^1]

#### Divide

不断从中间将序列切开，分成等长的两个序列组。

#### Conquer

从头到尾对比两段序列的每个元素。

#### Merge

经对比后，元素重新组合。

### Python code

也可以直接用python的heapq模块，里面就有merge[^2]

```python
# from heapq import merge
def merge_sort(arr):
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)


def merge(left, right):
    result = []
    l_index, r_index = 0, 0
    while l_index < len(left) and r_index < len(right):
        if left[l_index] <= right[r_index]:
            result.append(left[l_index])
            l_index += 1
        else:
            result.append(right[r_index])
            r_index += 1
    if l_index < len(left):
        result.extend(left[l_index:])
    if r_index < len(right):
        result.extend(right[r_index:])
    return result

```





[^1]: [wiki](https://en.wikipedia.org/wiki/Merge_sort)
[^2]: [python doc](https://docs.python.org/3.7/library/heapq.html#heapq.merge)