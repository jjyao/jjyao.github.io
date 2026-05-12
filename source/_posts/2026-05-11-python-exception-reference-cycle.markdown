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

A Python exception has reference cycle when it has a reference chain back to itself (i.e. "exc -> ... -> exc"). The following code shows an example of the cycle:

```python
def inner():
    exc = ValueError("")
    raise exc

try:
  inner()
except Exception as exc:
  assert exc.__traceback__.tb_next.f_locals['exc'] is exc
```

## Why are Python exception reference cycles bad


## How to avoid Python exception reference cycles
