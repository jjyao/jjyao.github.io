---
layout: post
title: "Interaction between HugeTLBFS and Hugepages"
date: 2019-04-01 19:57:31 -0700
comments: true
categories: 
---

This post shows the interaction between hugetlbfs and huge pages by a example program. All the results are based on linux 3.10.0-514.55.4.el7.x86_64.

<!-- more -->

## Setup
```
sudo mount -t hugetlbfs -o mode=0777,pagesize=2M nodev /mnt/huge
echo '10' | sudo tee /proc/sys/vm/nr_hugepages
```
This creates a hugetlbfs with maximum of 10 huge pages.

## Example Program
``` c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>

#define MB (1024*1024)

void meminfo(char* msg) {
  printf(msg);
  system("cat /proc/meminfo | grep HugePages_");
  printf("\n");
}

int main(int argc, char* argv[]) {
  meminfo("Initial meminfo: \n");

  int fd = open("/mnt/huge/foo.txt", O_RDWR | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
  meminfo("After create and open the file: \n");

  ftruncate(fd, 4 * MB);
  meminfo("After set file size to 4MB: \n");

  void* base = mmap(NULL, 4 * MB, PROT_WRITE | PROT_READ, MAP_SHARED, fd, 0);
  meminfo("After mmap 4MB without MAP_POPULATE: \n");

  munmap(base, 4 * MB);
  meminfo("After munmap: \n");

  base = mmap(NULL, 4 * MB, PROT_WRITE | PROT_READ, MAP_SHARED | MAP_POPULATE, fd, 0);
  meminfo("After mmap 4MB with MAP_POPULATE: \n");

  munmap(base, 4 * MB);
  meminfo("After munmap: \n");

  ftruncate(fd, 8 * MB);
  meminfo("After set file size to 8MB: \n");

  base = mmap(NULL, 8 * MB, PROT_WRITE | PROT_READ, MAP_SHARED | MAP_POPULATE, fd, 0);
  meminfo("After mmap 8MB with MAP_POPULATE: \n");

  munmap(base, 8 * MB);
  meminfo("After munmap: \n");

  close(fd);
  meminfo("After close: \n");
}
```

The output of running this program is:
```
Initial meminfo:
HugePages_Total:      10
HugePages_Free:       10
HugePages_Rsvd:        0
HugePages_Surp:        0

After creat and open the file:
HugePages_Total:      10
HugePages_Free:       10
HugePages_Rsvd:        0
HugePages_Surp:        0

After set file size to 4MB:
HugePages_Total:      10
HugePages_Free:       10
HugePages_Rsvd:        0
HugePages_Surp:        0

After mmap 4MB without MAP_POPULATE:
HugePages_Total:      10
HugePages_Free:       10
HugePages_Rsvd:        2
HugePages_Surp:        0

After munmap:
HugePages_Total:      10
HugePages_Free:       10
HugePages_Rsvd:        2
HugePages_Surp:        0

After mmap 4MB with MAP_POPULATE:
HugePages_Total:      10
HugePages_Free:        8
HugePages_Rsvd:        0
HugePages_Surp:        0

After munmap:
HugePages_Total:      10
HugePages_Free:        8
HugePages_Rsvd:        0
HugePages_Surp:        0

After set file size to 8MB:
HugePages_Total:      10
HugePages_Free:        8
HugePages_Rsvd:        0
HugePages_Surp:        0

After mmap 8MB with MAP_POPULATE:
HugePages_Total:      10
HugePages_Free:        6
HugePages_Rsvd:        0
HugePages_Surp:        0

After munmap:
HugePages_Total:      10
HugePages_Free:        6
HugePages_Rsvd:        0
HugePages_Surp:        0

After close:
HugePages_Total:      10
HugePages_Free:        6
HugePages_Rsvd:        0
HugePages_Surp:        0
```

## Observations
1. Just setting the file size won't allocate/reserve any huge pages. It only affects the logical file size not the physical one.
2. `mmap` only reserves the huge pages.
3. Populating the page table or accessing a page actually allocates the huge pages.
4. `HugePages_Free` includes `HugePages_Rsvd` so the number of free-to-use huge pages is actually `HugePages_Free` minus `HugePages_Rsvd`.

## Reference
1. [Linux下hugetlbpage使用详解](http://www.lenky.info/archives/2012/03/1219)
