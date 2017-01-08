---
title: Python 2 to 3 Quick Guide
tags:
  - python
  - english
categories:
  - Programming
date: 2017-01-08 12:26:54
---


Few years ago I was programming Python 2.7, when 3.x was still not an option, because of its backward-incompatibiliy and lack of popular third-party libraries support. But now it's safe to say Python 3 is [totally ready](http://py3readiness.org/), and here's a list of references for those (including me) who are adopting Python 3 with a 2.x background.

1. All Strings Are Unicode
2. `print` Becomes a Function
3. Less Lists More Views
4. Integer Division Returns Float
5. Comparison Operators Raises `TypeError`
6. Set Literal Support
7. New String Formatting
8. Exception Handling
9. Global Function Changes
10. Renaming Modules and Relative Import

## All Strings Are Unicode

When dealing with non-ASCII encodings in Python 2, there're `str`, `unicode`, `u'...'`, `s.encode()`, etc. In Python 3, there're only **text** and **binary data**. The former is `str`, strings that are always represented in Unicode; the later is `bytes`, which is just a sequence of byte numbers.

* Conversion between `str` and `bytes`:

```python
# str to bytes
'str'.encode('UTF-8')
bytes('str', encoding='UTF-8')

# bytes to str
b'bytes'.decode('UTF-8')
str(b'bytes', encoding='UTF-8')
```

* `basestring` is removed, use `str` as type: `isinstance(s, str)`
* `bytes` is immutable, the corresponding mutable version is `bytearray`.
* The default source file encoding is UTF-8 now.

<!-- more -->

## `print` Becomes a Function

In Python 2, `print` is a statement, and now it's used as a function:

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

## Less Lists More Views

A lot of well-known methods now return iterators, or 'views',  instead of eager-evaluated lists. 

* Dictionary's `keys`, `items`, and `values` methods, while removing `iterkeys`, `iteritems`, and `itervalues`. For example, when you need a sorted key list:

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

* There's no difference between `long` and `int` now, use `int` only.
* Octal literals are represented as `0o755`, instead of `0755`.

## Comparison Operators Raises `TypeError`

* `<`, `<=`, `>=`, `>` can no longer be used between different types.
* `==` and `!=` remains the same.
* `cmp` parameter in `sort` is removed. Use `key` to extract a comparison key from each element.

## Set Literal Support

```python
s = set([1, 2, 3]) # Old, also valid in Python 3
s = {1, 2, 3}      # New
s = set()          # Empty set
d = {}             # Empty dict
```

## New String Formatting

Python 3 introduces a new form of string formatting, and it's also back-ported to Python 2.x. The old `%s` formatting is still available in 3.x, but the [new format](https://docs.python.org/3/library/string.html#format-string-syntax) seems more expressive and powerful.

```python
# by position
'{}, {}, {}'.format('a', 'b', 'c')    # a, b, c
'{2}, {1}, {0}'.format('a', 'b', 'c') # c, b, a

# by name
'Hello, {name}'.format(name='Jerry') # Hello, Jerry

# by attribute
c = 1 - 2j
'real: {0.real}'.format(c) # real: 1.0

# by index
'X: {0[0]}, Y: {0[1]}'.format((1, 2)) # X: 1, Y: 2

# format number
'{:.2f}'.format(1.2)   # 1.20
'{:.2%}'.format(0.012) # 1.20%
'{:,}'.format(1234567) # 1,234,567

# padding
'{:>05}'.format(1) # 00001
```

Furthermore, Python 3.6 introduces literal string interpolation ([PEP 498](https://www.python.org/dev/peps/pep-0498/)).

```python
name = 'Jerry'
print(f'Hello, {name}')
```

## Exception Handling

Raise and catch exceptions in a more standard way:

```python
# Old
try:
  raise Exception, 'message'
except Exception, e:
  tb = sys.exc_info()[2]
  
# New
try:
  raise Exception('message')
except Exception as e:
  tb = e.__traceback__
```

## Global Function Changes

Some global functions are (re)moved to reduce duplication and language cruft.

* `reduce` is removed, use `functools.reduce`, or explict `for` loop instead.
* `apply` is removed, use `f(*args)` instead of `apply(f, args)`.
* `execfile` is removed, use `exec(open(fn).read())`
* Removed backticks, use `repr` instread.
* `raw_input` is renamed to `input`, and the old `input` behaviour can be achieved by `eval(input())`

## Renaming Modules and Relative Import

* Different URL modules are unified into `urllib` module, e.g.

```python
from urllib.request import urlopen, Request
from urllib.parse import urlencode
req = Request('http://shzhangji.com?' + urlencode({'t': 1})
with urlopen(req) as f:
  print(f.read())
```

* Some modules are renamed according to [PEP 8](https://www.python.org/dev/peps/pep-0008), such as:
  * ConfigParser -> configparser
  * copy_reg -> copyreg
  * test.test_support -> test.support
* Some modules have both pure Python implementation along with an accelerated version, like StringIO and cStringIO. In Python 3, user should always import the standard module, and fallback would happen automatically.
  * StringIO + cStringIO -> io
  * pickle + cPickle -> pickle
* All `import` forms are interpreted as absolute imports, unless started with `.`:

```python
from . import somemod
from .somemod import moremod
```

## References

* [What’s New In Python 3.0](https://docs.python.org/3/whatsnew/3.0.html)
* [Porting Code to Python 3 with 2to3](http://www.diveintopython3.net/porting-code-to-python-3-with-2to3.html)
* [The key differences between Python 2.7.x and Python 3.x with examples](http://sebastianraschka.com/Articles/2014_python_2_3_key_diff.html)
