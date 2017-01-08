---
title: Python 2 to 3 Quick Guide
categories: ['Programming']
tags: ['python', 'english']
---

is python3 ready?

1. `print` Becomes a Function
2. 

<!-- more -->

## `print` Becomes a Function

In Python 2, `print` is a statement, and now it's used as a function.

```python
print   # Old: print a new line
print() # New

print 'hello', 'world',          # Old: trailing comma suppresses new line
print('hello', 'world', end=' ') # New: end defaults to '\n'

print >>sys.stderr, 'error'     # Old: write to stderr
print('error', file=sys.stderr) # New
```

`print` function also provides `sep` and `flush` parameters:

```python
print('hello', 'world', sep=',', flush=True)

# Instead of:
print ','.join(('hello', 'world'))
sys.stdout.flush()
```

## All Strings Are Unicode

When dealing with non-ASCII encodings in Python 2, there're `str`, `unicode`, `u'...'`, `s.encode()`, etc. In Python 3, there're only **text** and **binary data**. The former is `str`, strings always represented in Unicode; the later is `bytes`, which is just a sequence of byte numbers.

* Conversion between `str` and `bytes`:

```python
# str to bytes
'abc'.encode('UTF-8')
bytes('abc', encoding='UTF-8')

# bytes to str
b'bytes'.decode('UTF-8')
str(b'bytes', encoding='UTF-8')
```

* `basestring` is removed, use `str` as type: `isinstance(s, str)`
* `bytes` is immutable, the corresponding mutable version is `bytearray`
* The default source encoding is UTF-8 now.

## Less Lists More Views

A lot of well-known methods now return iterators, or 'views',  instead of eager-evaluated lists. 

* Dictionary's `keys`, `items`, and `values` methods, while removing `iterkeys`, `iteritems`, and `itervalues`. When you need a sorted key list:

```python
k = d.keys(); k.sort() # Old
k = sorted(d.keys())   # New
```

* `map`, `filter`, and `zip`, while removing `imap` methods in `itertools` module. To get a concrete list, use list comprehension or the `list` global function:

```python
[x * 2 for x in [1, 2, 3]]
list(map(lambda x: x * 2, [1, 2, 3]))
```

* `range` is now equivalent to `xrange` in Python 2, the later is removed.
* For iterators, the `next` method is renamed to `__next__`, and there's a global `next` function, which accepts an iterator and calls its `__next__` method.

```python
iter([1]).next()     # Old
iter([1]).__next__() # New
next(iter([1]))      # New
```

## Integer Division Returns Float

```
print 1 / 2   # Old: prints 0
print 1 / 2.0 # Old: prints 0.5
print(1 / 2)  # New: prints 0.5
print(1 // 2) # New: prints 0
```

long and int
octal

## Comparison Operators Raises `TypeError`

`<`, `<=`, `>=`, `>`

`==`, `!=`

`cmp` is removed

## Set Literal Support

```python
s = set([1, 2, 3]) # Old, also valid in Python 3
s = {1, 2, 3}      # New
s = set()          # Empty set
d = {}             # Empty dict
```



## Modules

urllib
other renames

## Relative Import

## Global Function Changes

reduce
intern
apply
exec
execfile
repr (back ticks)
raw_input
callable

## Exception Handling

try except as
raise
throw


## Major Third-party Packages

## References

* [What’s New In Python 3.0](https://docs.python.org/3/whatsnew/3.0.html)
* [Porting Code to Python 3 with 2to3](http://www.diveintopython3.net/porting-code-to-python-3-with-2to3.html)
* [The key differences between Python 2.7.x and Python 3.x with examples](http://sebastianraschka.com/Articles/2014_python_2_3_key_diff.html)
