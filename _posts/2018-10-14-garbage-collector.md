---
layout: post
title: "Python的垃圾回收机制"
tags: [Python, Concept]
---
Python的垃圾回收机制

* 目录
{:toc .toc}
---
### Reference
> [Visualizing Garbage Collection by Pat Shuaghnessy](http://patshaughnessy.net/2013/10/24/visualizing-garbage-collection-in-ruby-and-python)
>
> [Generational GC in Python and Ruby by Pat Shuaghnessy](http://patshaughnessy.net/2013/10/30/generational-gc-in-python-and-ruby)
>
> [quora](https://www.quora.com/How-does-garbage-collection-in-Python-work-What-are-the-pros-and-cons)



# Garbage Collector的作用

1. 为新对象分配内存（allocate memory for new objects）
2. 辨别垃圾对象（identify garbage objects, and）
3. 回收内存（reclaim memory from garbage objects.）



# Reference Counting

Python会在创建新对象时实时向系统申请内存。

新的对象创建后，对象内部用*reference count*记录被引用数，每新增一个引用reference count增加1，反之减少1。当减少到0时，内存被回收。

> Just as before, Python sets the reference count in JKL to be 1. However, also notice since we changed n1 to point to JKL, it no longer references ABC, and that Python decremented its reference count down to 0.
>
> At this point, the Python garbage collector immediately jumps into action! Whenever an object’s reference count reaches zero, Python immediately frees it, returning it’s memory to the operating system:

![img](http://patshaughnessy.net/assets/2013/10/24/python6.png)

**reference counting的缺点**

1. 需要为每个对象额外增加空间以支持reference count（minor space penalty）。
2. 速度更慢。
3. 有时候会失效：含有循环结构的数据结构（cyclic data structure）的被引用数永不为0，如下代码所示

```python
class Node:
    def __init__(self, val):
        self.val = val

n1 = Node('ABC')  # refcnt = 1
n2 = Node('DEF')  # refcnt = 1
n1.next = n2      # refcnt = 2
n2.prev = n1      # refcnt = 2
n1 = None         # refcnt = 1
n2 = None         # refcnt = 1
```

# generational garbage collection

由于Reference counting无法解决循环引用的问题，Python引入了第二个算法：*generational garbage collection*（约在2.1版本里开始引入）。

Python用一个linked list来追踪被激活的对象，称为*Generation Zero*。python会逐个对对象检测是否含有相互引用并降低其refcnt。

### Generation Zero，One，Two

随程序运行，被占用的资源达到某个阀值时，python的collector被激活，对Generation Zero进行清洗。但并不是全部清除，那些将会被继续使用的对象会被迁移到Generation One。当被占用资源继续上升，到达某个阀值时，类似的过程会发生在Generation One上，“幸存”的对象继续被迁移到Generation Two。

因此，0代的对象会被更频繁地进行清理，2代频率最低。这意味着，约“老”的对象倾向于存活的更久。这种处理办法的根据是Weak Generational Hypothesis。





