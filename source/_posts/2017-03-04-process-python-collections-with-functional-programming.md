---
title: Process Python Collections with Functional Programming
tags:
  - python
  - fp
categories:
  - Programming
date: 2017-03-04 22:32:17
---


I develop Spark applications with Scala, and it has a very powerful [collection system](http://docs.scala-lang.org/overviews/collections/introduction), in which functional programming is certainly a key. Java 8 also introduces Lambda Expression and Stream API. In JavaScript, there is a [Lodash](https://lodash.com/) library that provides powerful tools to process arrays and objects. When my primary work language changes to Python, I am wondering if it's possible to manipulate collections in a FP way, and fortunately Python already provides syntax and tools for functional programming. Though list comprehension is the pythonic way to deal with collections, but the idea and concepts of FP is definitely worth learning.

## Wordcount Example

Let's first write a snippet to count the word occurences from a paragraph, in of course a functional way.

```python
import re
import itertools


content = """
an apple orange the grape
banana an apple melon
an orange banana apple
"""

word_matches = re.finditer(r'\S+', content)
words = map(lambda m: m.group(0), word_matches)
fruits = filter(lambda s: len(s) > 3, words)
grouped_fruits = itertools.groupby(sorted(fruits))
fruit_counts = map(lambda t: (t[0], len(list(t[1]))), grouped_fruits)
print(list(fruit_counts))
```

Run this example and you'll get a list of fruits, along with their counts:

```text
[('apple', 3), ('banana', 2), ('grape', 1), ('melon', 1), ('orange', 2)]
```

This example includes most aspects of processing collections with FP style. For instance, `re.finditer` returns an `iterator` that is lazily evaluated; `map` and `filter` are used to do transformations; `itertools` module provides various functions to cope with iterables; and last but not least, the `lambda` expression, an easy way to define inline anonymous function. All of them will be described in the following sections.

<!-- more -->

## Ingredients of Functional Programming

Python is far from being a functional language, but it provides some basic syntax and tools so that we can choose to write Python in a functional way.

### Function as First-class Citizen

Function is data. It can be assigned to a variable, pass as a parameter to another function, or returned by a function. The later two cases also refers to higher order functions. Python makes it quite easy, you can define and pass around the function:

```python
def add(a, b):
    return a + b
    
add_two = add
print(add_two(1, 2)) # => 3
    
def calculate(a, b, operation):
    return operation(a, b)
        
print(calculate(1, 2, add)) # => 3
```

Or generate a new function from a function:

```python
def add_n(n):
    def add(a):
        return a + n
    return add
                
add_1 = add_n(1)
print(add_1(1)) # => 2
```

To use function in `map`, which applies the function to every element of the iterable:

```python
print(list(map(add_1, [1, 2, 3]))) # => [2, 3, 4]
```

For very short function, we can use lambda expression:

```python
map(lambda a: a + 1, [1, 2, 3])
```

### Being Lazy

Lazy evaluation means postponing the execution until it's necessary. It's a very common optimization strategy in big data transformation, becuase all map-like operations should be chained and assigned to a single task. In Python, there's iterator, an stateful object that remembers the current element during iteration. Let's assume `calc` is a heavy function, and the following two lines differ:

```python
[calc(i) for i in [1, 2, 3]]
map(calc, [1, 2, 3])
```

List comprehension is eager-evaluated, while `map` (from Python 3.x on) returns an iterator. You can use the `next` global function to fetch the next element, or take the first two results using:

```python
from itertools import islice
list(islice(map(calc, [1, 2, 3]), 2))
```

It's worth mentioning that from Python 3.x on a lot of methods returns iterator instead of concrete list, you can refer to [this article](http://shzhangji.com/blog/2017/01/08/python-2-to-3-quick-guide/#Less-Lists-More-Views).

### Purity

A function is pure if its output only depends on its input, and it has no side-effect, i.e. without changing outer/global variable space. Here're some examples of pure/non-pure functions:

```python
def inc(a): # pure
    return a + 1

i = 0
def count(a): # non-pure
    i = len(a)
    
def greet(name): # non-pure, change the console
    print('hi', name)
```

Purity is a good functional style because:

* it makes you re-design the functions so that they become shorter;
* and short functions are easier to test, have less bugs;
* purity also enables parallel execution.

In concurrency programming, sharing state, lock, and context switch are all performance killers. Pure functions ensures codes can be executed in parallel without coordination of states, and can be re-executed multiple times safely.

```python
from concurrent.futures import ThreadPoolExecutor
executor = ThreadPoolExecutor(5)
list(executor.map(add_1, [1, 2, 3]))
```

### Function Composition

There're also topics on combining, currying, partially applying functions, so we can tackle complex problems with small well-defined functions. Python provides `decorator`, `generator` syntax, along with `functools`, `operator` modules for such tasks. These can be found in Python official documentation.

## Chaining Operations

`map`, `filter`, and functions in `itertools` cannot be easily chained. We have to nest the function calls or introduce intermediate variables. Luckily, there's an open-sourced [PyFunctional](https://github.com/EntilZha/PyFunctional) package that can help us transform or aggregate collections in a funcional way quite fluently.

```python
from functional import seq

seq(1, 2, 3, 4)\
    .map(lambda x: x * 2)\
    .filter(lambda x: x > 4)\
    .reduce(lambda x, y: x + y)
# => 14
```

## List Comprehension Or `map`?

List comprehension and generator expression are the pythonic way of processing collections, and the communiy encourages using list comprehension instead of `map`, etc. There's a nice [answer](http://stackoverflow.com/a/6407222/1030720) on StackOverflow that addresses the following principle: use `map` only when you already have a function defined. Otherwise just stick to listcomps for it's more widely accepted. Neverthelss, one should still pay attention to the laziness of various methods.

## Conclusion

Processing collections is only one application of functional programming. This program paradigm can be applied to other phases of designing your systems. Further materials like [SICP](http://deptinfo.unice.fr/~roy/sicp.pdf), [Functional Programming in Scala](https://www.manning.com/books/functional-programming-in-scala) are all very informative. Hope you enjoy.

## References

* [Functional Programming HOWTO](https://docs.python.org/3/howto/functional.html)
* [Functional Programming with Python](http://kachayev.github.io/talks/uapycon2012/)
* [Itertools Recipes](https://docs.python.org/3/library/itertools.html#itertools-recipes)
