---
layout: post
title: "HugeTLBFS Read Bug"
date: 2017-11-27 22:29:46 -0800
comments: true
categories: 
---

Hit a Linux kernel bug that `read()` returns wrong data if it crosses a hugepage boundary.

<!-- more -->

## Scenario
When I read a file in the `hugetlbfs` using `std::ifstream`, I fail to get the exact data of the file.

``` cpp
std::string file = "/mnt/huge/foo";
std::ifstream fin;
fin.open(file, std::ifstream::binary);
char c;
while (fin.get(c)) {
  std::cout << c;
}
fin.close()
```

However, if I use `fread()` I can get the correct data:

``` cpp
std::string file = "/mnt/huge/foo";
FILE* fin = fopen(file.c_str(), "rb");
char c;
while (fread(&c, 1, 1, fin) == 1) {
  std::cout << c;
}
fclose(fin);
```

To figure out the reason, I `strace` these two programs:

```
open("/mnt/huge/foo", O_RDONLY)         = 3
read(3, "\220N\210\344\36\227\276\303\305\301\334\346\246\245\3717tmg/\25\235C\365k\7\273T2\266\220\327"..., 8191) = 8191
read(3, "%\361!\253lek&\30\306\370\333f\304\357L6@z\224W<ef\335\206\225\246\342!\327\6"..., 8191) = 8191
read(3, "B\222\327-\17`'\250E[]\327mi\37\3308u\250\231F\200\250\35-\v\276\245>H\321R"..., 8191) = 8191
read(3, "u\311w\336\10h\374\f\214\301\376-\0258'\263;Iu1\273\267\345\313\246\22O\320\335\254'\7"..., 8191) = 8191
read(3, "\342\265\263\314\222\265rr\265*A\27\34<\342\344F\244|\371\f\231\345\331\343=\321SZx\273\240"..., 8191) = 8191
read(3, "=?\241\337\20\235\367\233\10\234;^\234\337\274\322\237\242\346\32\32\233gb\231\236DZ\336t\364]"..., 8191) = 8191
read(3, "1\233\21z\345\355?\243\342\361e\335\334\246\363\316A\267\361Nv\304\250\225\240Q\267\31\r\265\314'"..., 8191) = 8191
read(3, "$\24\277\\\213\320jGj\nb4\317\370p\216>5V\331\1\2561\275\24\233\326d+\1UM"..., 8191) = 8191
read(3, "\262\355\327!\2h\303\332\373\16\257\3\32y!O\303]5\331\256?Q\277t\27\262\223\316\357j("..., 8191) = 8191
read(3, "pd\2043\261\350C\313\356\200\366}\17\25\335\240?\357\225Fs\226qKW\241r\227b\2424\347"..., 8191) = 8191
...
```

```
open("/mnt/huge/foo", O_RDONLY)         = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=5242880, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f9328b03000
read(3, "\220N\210\344\36\227\276\303\305\301\334\346\246\245\3717tmg/\25\235C\365k\7\273T2\266\220\327"..., 4096) = 4096
read(3, "\v\327\265M\276\253\357\325m\244\253\351\237\350\273\21E<\326\356\03003\210\5\277\210h\200V5\376"..., 4096) = 4096
read(3, "\361!\253lek&\30\306\370\333f\304\357L6@z\224W<ef\335\206\225\246\342!\327\6\t"..., 4096) = 4096
read(3, "\343\300\202/\343\t\300\340\332\215\2148\226\342\251\377\fq_\21n\370\212\273tn\305\210#\320@`"..., 4096) = 4096
read(3, "\327-\17`'\250E[]\327mi\37\3308u\250\231F\200\250\35-\v\276\245>H\321R\277\36"..., 4096) = 4096
read(3, "t\346p0\204OeD\211\256\233g\242\3513\3X\367\0323\332\235\330\215\375\261G\234\217\17\34\375"..., 4096) = 4096
read(3, "\336\10h\374\f\214\301\376-\0258'\263;Iu1\273\267\345\313\246\22O\320\335\254'\7\205\r\325"..., 4096) = 4096
read(3, "xK\373\233\300\n\354\350>s\243\270\365D\276\263\226/\276\27S\225\"yL\4V\352\272\26b\261"..., 4096) = 4096
read(3, "\222\265rr\265*A\27\34<\342\344F\244|\371\f\231\345\331\343=\321SZx\273\240)\245h\224"..., 4096) = 4096
read(3, "P\332o>+\355\17\372\251\275n\266 \n\310aB\210\235\30u{\365\34\255\367\36\375\365\v\27\331"..., 4096) = 4096
read(3, "\235\367\233\10\234;^\234\337\274\322\237\242\346\32\32\233gb\231\236DZ\336t\364]\225\216.=C"..., 4096) = 4096
read(3, "p$\350r\31\215\"\225\331&\354\200\361\344\333L\201\37e\r\"\353\255\244\250?\253O\252A3\371"..., 4096) = 4096
...
```

Based on `strace`, `std::ifstream` reads 8191 bytes at a time and `fread()` reads 4096 bytes at a time. To check if the read size matters, I change the `std::ifstream` program so that it also reads 4096 bytes:

``` cpp
std::string file = "/mnt/huge/foo";
std::ifstream fin;
// with a user-provided buffer, libstdc++ reads n-1 bytes at a time
char buf[4096 + 1];
fin.rdbuf()->pubsetbuf(buf, sizeof(buf));
fin.open(file, std::ifstream::binary);
char c;
while (fin.get(c)) {
  std::cout << c;
}
fin.close()
```

After I change `std::ifstream` to read 4096 bytes at a time, I'm able to read the correct data so the read size matters. `read()` is a system call and it should handle all kinds of read size so the experiment indicates that there might be a bug somewhere in the kernel. After looking at the kernel commit log, something interesting shows up:

```
Author: Al Viro <viro@zeniv.linux.org.uk>
Date:   Fri Apr 3 11:31:35 2015 -0400

    switch hugetlbfs to ->read_iter()

    ... and fix the case when the area we are asked to read crosses
    a hugepage boundary

    Signed-off-by: Al Viro <viro@zeniv.linux.org.uk>
```

So I actually hit a kernel bug.

## Solution
Change the `std::ifstream` read size by providing a user-provided buffer so that read won't cross the hugepage boundary or upgrade the Linux kernel version to include the fix.
