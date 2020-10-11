---
layout: post
title: "Lazy Evaluation & Late Binding: Unexpected Behavior of Iterables in Python"
date: 2020-09-22
tags: coding python iterator lazy-evaluation late-binding leetcode heap
---
## Glossary
`iterable` in Python: 
> An object capable of returning its members one at a time.

Anything that can be used in a `for` loop, such as `list`, `tuple`, `str`, `dict`, `set` and iterators. We can convert an iterable to an iterator, which is actually what `for` statement does, by passing it to the built-in function `iter()`. 

`iterator` in Python:
> An object representing a stream of data.

It must have the `__next__()` method (or can be passed to the built-in function `next()`), such as generators, `map` objects.

## The Problem

Recently, I've been practicing on [Leetcode](https://leetcode.com). In problem [373. Find K Pairs with Smallest Sums](https://leetcode.com/problems/find-k-pairs-with-smallest-sums/), I implemented a solution inspired by discussions in the community using `heapq.merge` in Python as follows:
```python
def kSmallestPairs(self, nums1: List[int], nums2: List[int], k: int) -> List[List[int]]:
    import heapq
    # we have m*n pairs, imagine a m*n matrix
    rows = (([u, v] for v in nums2) for u in nums1)
    # since both nums1 and nums2 are ascending
    # each row is already sorted based on the sum of u and v

    # regular slicing cannot be applied to generator since 'generator' object does not have __getitem__
    # we should use itertools.islice
    return list(itertools.islice(heapq.merge(*rows, key=sum), k))

print(kSmallestPairs(nums1=[1, 7, 11], nums2=[2, 4, 6], k=9))
```
The main idea of `heapq.merge` is like a multi-way merge sort on many already-sorted iterables. The difference is that `heapq.merge` uses a heap for ordering. Heap is just a binary tree in Python, and it is implemented with an array. `heapq.merge` initially collects the first element of each input iterable to form a list, then it `heapify` the list, transforming it to a heap. The first element of heap—`h[0]`, is always the smallest in the heap (supposing it is a min-heap). `heap.merge` just `pop` `h[0]` and puts it into the result iterator, and then `push` the following element of `h[0]` from the same input iterable into the heap (more details at [source code](https://github.com/python/cpython/blob/master/Lib/heapq.py#L314)). We can also use heap as a priority queue.  

Back to the previous code, `rows` is a generator of generators. We expect `k` pairs of smallest sums as the output, which should be:
```python
[[1, 2], [1, 4], [1, 6], [7, 2], [7, 4], [7, 6], [11, 2], [11, 4], [11, 6]]
```
However, the output actually is:
```python
[[11, 2], [11, 2], [11, 2], [11, 4], [11, 4], [11, 4], [11, 6], [11, 6], [11, 6]]
```
Is something wrong with the generator? Try following code:
```python
nums1=[1, 7, 11]
nums2=[2, 4, 6]
rows = (([u, v] for v in nums2) for u in nums1)
for row in rows:
    for element in row:
        print(element, end=' ')
```
Surprisingly, the output is what we expect:
```python
[1, 2] [1, 4] [1, 6] [7, 2] [7, 4] [7, 6] [11, 2] [11, 4] [11, 6]
```

## Lazy Evaluation

We know that generators in Python are lazy evaluation. An element in a generator is only evaluated when we iterate to it. Most other objects in Python are eager evaluation.
```python
j = 1
L = [j for _ in range(4)]  # list
G = (j for _ in range(4))  # generator
j = 2
print(L)  # [1, 1, 1, 1]
print(list(G))  # [2, 2, 2, 2]
```
In above snippet, list `L` is evaluated when `j=1`, while generator `G` is not evaluated until we call `list(G)`, at that time `j=2`.

Back to the generator `rows`, if we make it a list of generators instead of a generator of generators:
```python
rows = [([u, v] for v in nums2) for u in nums1]
for row in rows:
    for element in row:
        print(element, end=' ')
```
We have output:
```python
[11, 2] [11, 4] [11, 6] [11, 2] [11, 4] [11, 6] [11, 2] [11, 4] [11, 6] 
```
The output is different from when `rows` is a generator, but it is similar to the output of `heapq.merge`. The only difference is we did not sort the list as in `heapq.merge`.

If we think about the evaluation mechanism, when we create `rows` as a list, since list is eager evaluation, `u` should be 11 after that line is executed. That is why when we iterate those three generators later, the first elements of all pairs are 11. When `rows` is a generator, since generators are lazy evaluation, the value of `u` only changes after we finish iterating each row.

Back to `heapq.merge`, which is actually `heapq.merge(*iterables, key=None, reverse=False)`. The first parameter `iterables` becomes a tuple after it receives input iterables. We can easily check this by following snippet:
```python
rows = (([u, v] for v in nums2) for u in nums1)

def func(*iterables):
    print(type(iterables))  # <class 'tuple'>
    for it in iterables:
        for pair in it:
            print(pair, end=' ')

func(*rows)  # [11, 2] [11, 4] [11, 6] [11, 2] [11, 4] [11, 6] [11, 2] [11, 4] [11, 6]
```
The output is the same as when `rows` is a list even if we pass `rows` as a generator. Because it is converted to a tuple in the function, and tuple is also eager evaluation just like list.

However, even if `heapq.merge` does not convert `rows` to a tuple, we still cannot expect correct output. We mentioned before that `heapq.merge` initially collects the first element of each input iterable, considering following code:
```python
rows = (([u, v] for v in nums2) for u in nums1)
it1 = next(rows)
print(next(it1))  # [1, 2]
it2 = next(rows)
print(next(it1))  # [7, 4], we expect [1, 4] 
print(next(it2))  # [7, 2]
it3 = next(rows)
print(next(it1))  # [11, 6], we expect [1, 6] 
print(next(it2))  # [11, 4], we expect [7, 4]
print(next(it3))  # [11, 2]
```
We find that every time we call `next(rows)`, the value of `u` changes. Thus, even if `heapq.merge` dose not convert `rows` to a tuple, we can only guarantee correct pairs for the first element in each generator. After that, the first value of all subsequent pairs will be the same.

## Solution

The correct way to solve the above problem is to make `rows` a `map` object:
```python
rows = map(lambda u: ([u, v] for v in nums2), nums1)
print(list(heapq.merge(*rows, key=sum)))
# [[1, 2], [1, 4], [1, 6], [7, 2], [7, 4], [7, 6], [11, 2], [11, 4], [11, 6]]
```
This `map` object is also converted to a tuple when passing to `heapq.merge`, however, it gives us correct result. `map` objects are iterators, using lazy evaluation just like generators. The difference here is that this uses a function to create three generators. **That `u` here is a parameter of the lambda function, not a single variable that holds the value of an element in `nums1`, and this function will be called three times with different values passed in.** Therefore, this actually has nothing to do with `map`. It really depends on what `u` is. If we define `rows` as follows, we can also get the correct output:
```python
rows = ((lambda u: ([u, v] for v in nums2))(k) for k in nums1)
print(list(heapq.merge(*rows, key=sum)))
# [[1, 2], [1, 4], [1, 6], [7, 2], [7, 4], [7, 6], [11, 2], [11, 4], [11, 6]]

rows = [(lambda u: ([u, v] for v in nums2))(k) for k in nums1]
print(list(heapq.merge(*rows, key=sum)))
# [[1, 2], [1, 4], [1, 6], [7, 2], [7, 4], [7, 6], [11, 2], [11, 4], [11, 6]]
```
We replaced `u` with `k`, which holds the value of the element currently being iterated in `nums1`, to make the structure more clear. However, this is not necessary since even if we use `u` to iterate `nums1`, in the lambda function, `u` will be overridden by the parameter `u` of the function.

To better understand how this `u` affects the output, we can also use lambda expressions to write the equivalents of previous definitions of `rows` that give us incorrect results:
```python
rows = (([u, v] for v in nums2) for u in nums1)
# is equivalent to
rows = ((lambda: ([u, v] for v in nums2))() for u in nums1)

rows = [([u, v] for v in nums2) for u in nums1]
# is equivalent to
rows = [(lambda: ([u, v] for v in nums2))() for u in nums1]
```

## Late Binding

I also want to mention late binding here because it usually leads to unexpected behavior in Python, though not specifically limited to iterables. According to [Wikipedia](https://en.wikipedia.org/wiki/Late_binding):
> Late binding, dynamic binding, or dynamic linkage is a computer programming mechanism in which the method being called upon an object or the function being called with arguments is looked up by name at runtime.

There is a good [article](https://docs.python-guide.org/writing/gotchas/#late-binding-closures) talking about late binding in Python. We will use some snippets to figure out how late binding works.
```python
funcs = [lambda: u for u in range(5)]
for func in funcs:
    print(func(), end=' ')
# 4 4 4 4 4
```
The output again is a repeat of the last element of `range(5)`. We defined `funcs`, which is a list of 5 functions. Each function does not accept any parameters and returns `u`, which holds the value of the element being iterated in `range(5)`. These functions are not called until we enter the following `for` loop and use `func()`. 

**Late binding means that values of variables used in a function are looked up at the time the function is called.** When we call those five functions in `funcs`, they will all look up the value of `u`, which becomes `4` after we define `funcs`, because `funcs`, which is a list, use eager evaluation. However, if we define `funcs` as generators, the output becomes what we expect:
```python
funcs = (lambda: u for u in range(5))
for func in funcs:
    print(func(), end=' ')
# 0 1 2 3 4 
```
This is obvious since generators use lazy evaluation. The value of `u` changes along the `for` loop.

Another example of late binding without iterables:
```python
i = 1
def func():
    return i
i = 2
print(func()) # 2
```