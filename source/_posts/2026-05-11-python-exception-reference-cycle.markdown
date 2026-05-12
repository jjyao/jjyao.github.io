---
layout: post
title: "Python Exception Reference Cycle"
date: 2026-05-11 20:59:07 -0700
comments: true
categories:
---

This post talks about Python exception reference cycles: what they are, why they can be problematic and how to avoid them.

<!-- more -->

## What are Python exception reference cycles

A Python exception has a reference cycle when it has a reference chain back to itself (i.e. "exc -> ... -> exc"). The following code shows an example of the cycle:

```python
def inner():
    exc = ValueError("")
    raise exc

try:
  inner()
except Exception as exc:
  assert exc.__traceback__.tb_next.tb_frame.f_locals['exc'] is exc
  print("Found Python exception reference cycle!")
```

## Why are Python exception reference cycles bad

To understand why Python exception reference cycles are bad, we need to understand some behaviors of Python GC and Python exceptions.

### Python GC

Python has two ways to find objects to deallocate: reference counting finds objects whose ref counts are zero and the garbage collector finds objects with circular references. For exceptions with reference cycles, only the garbage collector can deallocate them.

Unless manually triggered by `gc.collect()`, Python GC runs every [X (allocations - deallocations)](https://docs.python.org/3/library/gc.html#gc.set_threshold). Unlike other languages, it's not triggered under memory pressure so it's purely count based. The implication is that a program can hold lots of memory in a few large objects with reference cycles and doesn't trigger GC because the count threshold isn't reached. In the worst case, the program can OOM.

### Python exceptions

Unlike many other languages, a raised Python exception has references to all frames of the call stack from main all the way to where the exception is raised, and each frame has references to all local variables. As a result, a Python exception can keep lots of objects alive (i.e. prevent ref counts from going down to zero).

```python
def inner():
  big = [1] * 1024 * 1024
  exc = ValueError("")
  raise exc

def outer():
  large = [2] * 1024 * 1024
  try:
    inner()
  except Exception as exc:
    assert exc.__traceback__.tb_next.tb_frame.f_locals['big'] == [1] * 1024 * 1024
    assert exc.__traceback__.tb_frame.f_locals['large'] == [2] * 1024 * 1024
    assert exc.__traceback__.tb_frame.f_back.f_locals['huge'] == [3] * 1024 * 1024
    print("Exception has reference to big, large and huge objects!")

huge = [3] * 1024 * 1024
outer()
```

The net effect is that an exception with a reference cycle can hold lots of objects alive until the next GC happens which can be greatly delayed and the program may just OOM or run out of resources (e.g. open files) before that.

```python
import gc

class Giant:
  def __init__(self):
    self.data = [4] * 1024 * 1024

  def __del__(self):
    print("Giant __del__ is called")

def inner():
  exc = ValueError("")
  raise exc

def outer():
  giant = Giant()
  try:
    inner()
  except Exception:
    pass

outer()
print("Calling GC")
gc.collect()
```

If you run the above program, you will get

```
Calling GC
Giant __del__ is called
```

and this shows that we now rely on GC to deallocate the Giant object which is undesirable.

## How to avoid Python exception reference cycles

It turns out the issue we described above is well known to the Python developers and there is a [PEP](https://peps.python.org/pep-3110/) that's implemented to help us break the exception reference cycle. Basically we should avoid having local variables point to the exception. In our example, we can do the following to break the reference cycle:

```python
def inner(): # BAD, has cycle
  exc = ValueError("")
  raise exc

def inner(): # GOOD, no cycle
  raise ValueError("")

def inner(): # GOOD, no cycle
  exc = ValueError("")
  try:
    raise exc
  finally:
    del exc
```
