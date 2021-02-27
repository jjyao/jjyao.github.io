---
layout: post
title: "Nondeterministic Code"
date: 2021-02-26 16:11:07 -0800
comments: true
categories: 
---

Nondeterministic code is hard to debug since bugs are not consistently reproducible. It's easy to notice that the code is perhaps nondeterministic if multi-threading or random functions are involved. However we can still write nondeterministic single-threaded code without using random functions.
<!-- more -->

## Random Address
[Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) randomly arranges the address space positions of stack and heap of a process, which means the address of a variable changes for each run. This is the source of nondeterminism if we try to iterate through an unordered container of pointers.

``` cpp
#include <iostream>
#include <unordered_set>

int main() {
  std::string a = "aaaaa";
  std::string b = "bb";
  std::string c = "cccccccccccc";
  std::unordered_set<std::string*> set;
  set.emplace(&a);
  set.emplace(&b);
  set.emplace(&c);

  for (std::string* p : set) {
    std::cout << *p << "(" << p << ", " << std::hash<std::string*>{}(p) << ")" << std::endl;
  }

  return 0;
}
```

```
> clang++ demo.cc -o demo -std=c++11
> ./demo
cccccccccccc(0x7ffeeedbf650, 16564608384425460261)
bb(0x7ffeeedbf678, 5964435063914947271)
aaaaa(0x7ffeeedbf690, 696214236533423747)
> ./demo
bb(0x7ffee3cd6678, 2875564916106665130)
cccccccccccc(0x7ffee3cd6650, 14656991250807877473)
aaaaa(0x7ffee3cd6690, 10500981134548248633)
```

As we can see, the address of variable a, b and c keeps changing for each run, which affects their positions in the hash set and, as a result, we will iterate them in different orders.

Lets try to [disable](https://stackoverflow.com/questions/23897963/documented-way-to-disable-aslr-on-os-x) ASLR and verify that the nondeterminism is gone:

```
> clang++ demo.cc -o demo -std=c++11 -Wl,-no_pie
> ./demo
bb(0x7ffeefbff678, 6189850235780456993)
cccccccccccc(0x7ffeefbff650, 14250952053958696967)
aaaaa(0x7ffeefbff690, 4593858558987980877)
> ./demo
bb(0x7ffeefbff678, 6189850235780456993)
cccccccccccc(0x7ffeefbff650, 14250952053958696967)
aaaaa(0x7ffeefbff690, 4593858558987980877)
```

Since the nondeterminism is generally undesired, clang has a [checker](https://clang.llvm.org/docs/analyzer/checkers.html#alpha-nondeterminism-pointeriteration-c) to detect such usages.
