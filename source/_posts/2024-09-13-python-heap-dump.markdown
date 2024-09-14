---
layout: post
title: "Python Heap Dump"
date: 2024-09-13 21:42:32 -0700
comments: true
categories: 
---

This post shows how to heap dump a *running* Python process using [pyrasite](https://pypi.org/project/pyrasite/) and [guppy3](https://pypi.org/project/guppy3/).

<!-- more -->

### Install pyrasite

``` bash
pip install pyrasite
```

`pyrasite` allows you to attach to a running Python process and run arbitrary Python code. It needs ptrace to function properly and the way to enable it varies depending on your OS. For Ubuntu, you can do:

``` bash
sudo sysctl -w kernel.yama.ptrace_scope=0
```

If you use `Conda`, you might need to run `unset LD_LIBRARY_PATH` so that `gdb` can use the system `libstdc++.so` instead of the one installed inside your conda env which can be incompatible.

### Install guppy3

``` bash
pip install guppy3
```

`guppy3` has a subpackage `heapy` that allows you to inspect the heap.

### Dump Heap 

Once everything is installed, we can then use pyrasite to attach to the target running Python process:

``` bash
pyrasite-shell <pid>
```

This attaches to the process and opens a REPL that you can run the heap dump code using `guppy3`:

``` python
from guppy import hpy
h = hpy()
heap = h.heap()

print(heap.all) # print the heap dump
"""
Partition of a set of 222038 objects. Total size = 26540189 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0  67095  30  6591256  25   6591256  25 str
     1  50143  23  3623400  14  10214656  38 tuple
     2  14415   6  2563320  10  12777976  48 types.CodeType
     3  28124  13  2205321   8  14983297  56 bytes
     4   2175   1  2064360   8  17047657  64 type
     5   4816   2  1962952   7  19010609  72 dict (no owner)
     6  13577   6  1846472   7  20857081  79 function
     7   2175   1  1087024   4  21944105  83 dict of type
     8    690   0  1019024   4  22963129  87 dict of module
     9  10472   5   312536   1  23275665  88 int
    10    273   0   244608   1  23520273  89 google._upb._message.MessageMeta
"""

print(heap.all[10].shpaths) # print the shortest path from root to object with index 10
"""
 0: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['ActorTableData']
 1: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['AvailableResources']
 2: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['ErrorTableData']
 3: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['GcsEntry']
 4: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['GcsNodeInfo']
 5: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['JobConfig']
 6: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['JobTableData']
 7: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['PlacementGroupTableData']
 8: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['PubSubMessage']
 9: hp.Root.i0_modules['ray._private.gcs_utils'].__dict__['ResourceDemand']
<... 271 more paths ...>
"""
```

