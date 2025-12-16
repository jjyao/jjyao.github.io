---
layout: post
title: "Demystify smaps"
date: 2025-12-14 23:19:03 -0700
comments: true
categories:
---

This post demystifies the Linux `/proc/<pid>/smaps` file and its `Shared_Clean`, `Shared_Dirty`, `Private_Clean` and `Private_Dirty` fields.

<!-- more -->

## What is smaps

The smaps file in Linux provides detailed information for each of the process's VMAs (Virtual Memory Areas), represented by [struct vm_area_struct](https://github.com/torvalds/linux/blob/v5.15/include/linux/mm_types.h#L319). The content of smaps file of a process is constructed by [iterating](https://github.com/torvalds/linux/blob/v5.15/fs/proc/task_mmu.c#L955) through each VMA and calling [show_smap](https://github.com/torvalds/linux/blob/v5.15/fs/proc/task_mmu.c#L811). The `Shared_Clean`, `Shared_Dirty`, `Private_Clean` and `Private_Dirty` fields are printed [here](https://github.com/torvalds/linux/blob/v5.15/fs/proc/task_mmu.c#L790) and calculated [here](https://github.com/torvalds/linux/blob/v5.15/fs/proc/task_mmu.c#L403).

## Shared vs Private

From the perspective of smaps, whether a page is considered shared or private is determined by its [page_mapcount](https://github.com/torvalds/linux/blob/v5.15/fs/proc/task_mmu.c#L467C18-L467C31) which is basically [page->_mapcount](https://github.com/torvalds/linux/blob/v5.15/include/linux/mm.h#L877) NOT by the `MAP_SHARED` or `MAP_PRIVATE` flags used in [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html). [_mapcount](https://github.com/torvalds/linux/blob/v5.15/include/linux/mm_types.h#L201) of a page is a counter tracks how many page table entries (PTEs) point to this physical page across ALL processes. A page is considered shared if there are at least 2 PTEs point to it.

Let's verify this with different cases:

```bash
dd if=/dev/zero of=/tmp/smaps  bs=1M count=1024
```

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char *argv[]) {
  int fd = open("/tmp/smaps", O_RDWR);
  if (fd < 0) {
    perror("Failed to open");
    exit(1);
  }

  const int num_mmaps = atoi(argv[1]);
  const bool map_shared = strcmp(argv[2], "MAP_SHARED") == 0;
  const bool map_populate = strcmp(argv[3], "MAP_POPULATE") == 0;
  printf("Command line args: num_mmaps=%d, map_shared=%d, map_populate=%d\n",
         num_mmaps, map_shared, map_populate);

  for (int i = 0; i < num_mmaps; i++) {
    void *map = mmap(NULL, 512 * 1024 * 1024, PROT_READ | PROT_WRITE,
                     (map_shared ? MAP_SHARED : MAP_PRIVATE) | (map_populate ? MAP_POPULATE : 0),
                     fd, 0);

    if (map == MAP_FAILED) {
      perror("Failed to mmap");
      close(fd);
      exit(1);
    }

    int *data = (int*) map;
    printf("data = %d\n", data[0]);
  }

  while (1) {
    sleep(2000000);
  }
}
```

This is a test program that mmaps 512M out of a 1GB file in various ways.

### Single MAP_SHARED mmap

```bash
./test 1 MAP_SHARED NO_MAP_POPULATE
```

Content of smaps:

```
7f418625d000-7f41a625d000 rw-s 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  64 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:        64 kB
Private_Dirty:         0 kB
...
```

Several things to note here:

- The mmaped pages are considered private even though we use `MAP_SHARED` because each physical page only has 1 PTE points to it.
- mmap is lazy and doesn't populate the page table unless MAP_POPULATE is specified and that's why we didn't see 524288 KB in Private_Clean.
- We also didn't see 4 KB in Private_Clean even though we just read one integer from the file and that's due to [fault-around](https://github.com/torvalds/linux/blob/v5.15/mm/memory.c#L4107).

### Multiple MAP_SHARED mmaps in single process

```bash
./test 2 MAP_SHARED NO_MAP_POPULATE
```

Content of smaps:

```
7f764be9a000-7f766be9a000 rw-s 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  32 kB
Shared_Clean:         64 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
...
7f766be9a000-7f768be9a000 rw-s 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  32 kB
Shared_Clean:         64 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
...
```

In this case, we mmaps the same file twice in the same process so each page now has 2 PTEs point to it and that's why they are considered shared.

### Multiple MAP_SHARED mmaps in multiple processes

```bash
./test 1 MAP_SHARED NO_MAP_POPULATE
```

```bash
./test 1 MAP_SHARED NO_MAP_POPULATE
```

Content of smaps:

```
7fb76d49e000-7fb78d49e000 rw-s 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  32 kB
Shared_Clean:         64 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
```

```
7f4e1eb61000-7f4e3eb61000 rw-s 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  32 kB
Shared_Clean:         64 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
```

In this case, we mmaps the same file once in two processes so each page also has 2 PTEs point to it and that's why they are considered shared.

### Multiple MAP_PRIVATE mmaps in single process

```bash
./test 2 MAP_PRIVATE NO_MAP_POPULATE
```

Content of smaps:

```
7f726c909000-7f728c909000 rw-p 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  32 kB
Shared_Clean:         64 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
...
7f728c909000-7f72ac909000 rw-p 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  64 kB
Pss:                  32 kB
Shared_Clean:         64 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
...
```

Since MAP_PRIVATE is copy-on-write (COW) and we only read from the file so the two VMAs are backed by the same physical pages and hence those pages are considered shared.

### Multiple MAP_PRIVATE mmaps with MAP_POPULATE in single process

```bash
./test 2 MAP_PRIVATE MAP_POPULATE
```

Content of smaps:

```
7fafbaabe000-7fafdaabe000 rw-p 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:              524288 kB
Pss:              524288 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:    524288 kB
...
7fafdaabe000-7faffaabe000 rw-p 00000000 103:01 132                       /tmp/smaps
Size:             524288 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:              524288 kB
Pss:              524288 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:    524288 kB
...
```

If the mmap is PROT_WRITE, MAP_PRIVATE and MAP_POPULATE, then kernel will not only populate the page tables but also [eagerly triggers COW](https://github.com/torvalds/linux/blob/v5.15/mm/gup.c#L1477) so each VMA is backed by their own private physcial pages.

## Clean vs Dirty

From the perspective of smaps, whether a page is considered clean or dirty is determined by [this](https://github.com/torvalds/linux/blob/v5.15/fs/proc/task_mmu.c#L419). Conceptually a page is dirty if it cannot just be discarded without data loss, which means it has dirty writes that haven't been flushed to the underlying file or swap space yet.

