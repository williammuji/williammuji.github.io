---
layout: post
title:  "Memory Allocator"
date: 2020-09-017 11:00:00 +0800
categories: jekyll update
---

1. Goals
2. Kernel allocator
3. Malloc allocator
4. TCMalloc allocator


主要讲述kernel、glibc以及tcmalloc中的内存分配器。
Kernel: Buddy/Slab Allocator
Glibc: Malloc Allocator
Google: TCMalloc Allocator


## 1. Goals

**Minimizing Space**
The allocator should not waste space: It should obtain as little memory from the system as possible, and should maintain memory in ways that minimize fragmentation.
**Minimizing Time**
The malloc(), free() and realloc routines should be as fast as possible in the average case.
**Maximizing Tunability**
Optional features and behavior should be controllable by users either statically (via #define and the like) or dynamically (via control commands such as mallopt).
**Maximizing Locality**
Allocating chunks of memory that are typically used together near each other. This helps minimize page and cache misses during program execution.

Doug Lea[A Memory Allocator](http://gee.cs.oswego.edu/dl/html/malloc.html)


## 2. Kernel allocator

Linux下大内存用Buddy allocator，小内存用Slab allocator分配。
每种分配器都会尽量利用locality，减少全局锁争用，所以会有Per-CPU cache。


### 2.1 Buddy algorithm

[Physical Page Allocation](https://www.kernel.org/doc/gorman/html/understand/understand009.html)

**什么用途**
Mangage and allocate physical pages in Linux.

**优点**
Extremely fast in comparison to other allocators.

**原理**
This is an allocation scheme which combines a normal power-of-two allocator with free buffer coalescing and the basic concept behind it is quite simple. 
Memory is broken up into large blocks of pages where each block is a power of two number of pages. 
If a block of the desired size is not available, a large block is broken up in half and the two blocks are buddies to each other. 
One half is used for the allocation and the other is free. 
The blocks are continuously halved as necessary until a block of the desired size is available. 
When a block is later freed, the buddy is examined and the two coalesced if it is free.

![buddy allocator](https://williammuji.github.io/images/buddy-allocator.png)

![buddy allocator split](https://williammuji.github.io/images/buddy-allocator-split.png)

Each zone has a free_area_t struct array called free_area[MAX_ORDER]. It is declared in <linux/mm.h> as follows:
```C
 22 typedef struct free_area_struct {
 23         struct list_head        free_list;	// A linked list of free page blocks
 24         unsigned long           *map;	// A bitmap representing the state of a pair of buddies
 25 } free_area_t; 
```

**Per-CPU Page Lists**
In kernel 2.6, pages are allocated from a struct per_cpu_pageset by buffered_rmqueue(). 
If the low watermark (per_cpu_pageset→low) has not been reached, the pages will be allocated from the pageset with no requirement for a spinlock to be held. 
Once the low watermark is reached, a large number of pages will be allocated in bulk with the interrupt safe spinlock held, added to the per-cpu list and then one returned to the caller.

Higher order allocations, which are relatively rare, still require the interrupt safe spinlock to be held and there will be no delay in the splits or coalescing. 
With 0 order allocations, splits will be delayed until the low watermark is reached in the per-cpu set and coalescing will be delayed until the high watermark is reached.

The implication of this change is straight forward; 
The number of times the spinlock protecting the buddy lists must be acquired is reduced. 

Higher order allocations are relatively rare in Linux so the optimisation is for the common case. 
This change will be noticeable on large number of CPU machines but will make little difference to single CPUs. 

There are a few issues with pagesets but they are not recognised as a serious problem. 
The first issue is that high order allocations may fail if the pagesets hold order-0 pages that would normally be merged into higher order contiguous blocks. 
The second is that an order-0 allocation may fail if memory is low, the current CPU pageset is empty and other CPU's pagesets are full, as no mechanism exists for reclaiming pages from “remote” pagesets. 
The last potential problem is that buddies of newly freed pages could exist in other pagesets leading to possible fragmentation problems.


**Avoiding Fragmentation**

External fragmentation is the inability to service a request because the available memory exists only in small blocks. 
Internal fragmentation is defined as the wasted space where a large block had to be assigned to service a small request.
In Linux, external fragmentation is not a serious problem as large requests for contiguous pages are rare and usually vmalloc() is sufficient to service the request. 
The lists of free blocks ensure that large blocks do not have to be split unnecessarily.

Internal fragmentation is the single most serious failing of the binary buddy system. 
While fragmentation is expected to be in the region of 28%, it has been shown that it can be in the region of 60%, in comparison to just 1% with the first fit allocator.
It has also been shown that using variations of the buddy system will not help the situation significantly.
To address this problem, Linux uses a slab allocator to carve up pages into small blocks of memory for allocation. 
With this combination of allocators, the kernel can ensure that the amount of memory wasted due to internal fragmentation is kept to a minimum.


### 2.2 Slab allocator

**kernel为什么设计slab allocator?**
buddy algorithm适用于大块内存的请求，对于几十或者几百个字节的内存请求，如果用buddy algorithm分配单位为页框，造成内存浪费。
显然如何在一个页框内如何设计数据结构来满足小块内存的分配，一个常规的解决方案，根据几何分布的内存大小(32-131,072字节)建立内存区链表，
虽然保证内碎片小于50%，但是还是会有internal fragmentation，于是引入slab allocator。

**它解决什么问题?**
内核经常请求同一类型的内存区，比如创建进程时，为进程描述符、打开文件对象等固定大小对象分配内存区，进程结束后这些内存区可以被重新利用。进程创建和删除频繁，那么
可以节省反复分配和回收此类内存区的消耗。同时也结减少internal fragmentation。
slab allocator把对象分组放入cache，每个cache是同种对象的一种储备，比如，当一个文件被打开时，存放相应打开文件对象所需的内存区是从一个叫做filp的slab allocator的
cache中得到。
cache被划分为多个slab，每个slab由一个或多个连续的页框组成，这些页框即包含已分配的对象，又包含空闲的对象。

![Cache Slab Object](https://williammuji.github.io/images/cache-slab-object.png)

**如何实现**

[Slab Allocator](https://www.kernel.org/doc/gorman/html/understand/understand011.html)

To speed allocation and freeing of objects and slabs they are arranged into three lists; 
slabs_full, slabs_partial and slabs_free. 
slabs_full has all its objects in use. 
slabs_partial has free objects in it and so is a prime candidate for allocation of objects. 
slabs_free has no allocated objects and so is a prime candidate for slab destruction.

**Per-CPU Object Cache**

One of the tasks the slab allocator is dedicated to is improved hardware cache utilization. 
An aim of high performance computing in general is to use data on the same CPU for as long as possible. 
Linux achieves this by trying to keep objects in the same CPU cache with a Per-CPU object cache, simply called a cpucache for each CPU in the system.

When allocating or freeing objects, they are placed in the cpucache. 
When there are no objects free, a batch of objects is placed into the pool. 
When the pool gets too large, half of them are removed and placed in the global cache. 
This way the hardware cache will be used for as long as possible on the same CPU.

The second major benefit of this method is that spinlocks do not have to be held when accessing the CPU pool as we are guaranteed another CPU won't access the local data. 
This is important because without the caches, the spinlock would have to be acquired for every allocation and free which is unnecessarily expensive.


## 3. Malloc allocator

[Malloc Allocator](https://www.gnu.org/software/libc/manual/html_node/The-GNU-Allocator.html)

The malloc implementation in the GNU C Library is derived from ptmalloc (pthreads malloc), which in turn is derived from dlmalloc (Doug Lea malloc).

As opposed to other versions, the malloc in the GNU C Library does not round up chunk sizes to powers of two, neither for large nor for small sizes. 
Neighboring chunks can be coalesced on a free no matter what their size is. 
This makes the implementation suitable for all kinds of allocation patterns without generally incurring high memory waste through fragmentation. 
The presence of multiple arenas allows multiple threads to allocate memory simultaneously in separate arenas, thus improving performance.


[Malloc Internals](https://sourceware.org/glibc/wiki/MallocInternals)

### 3.1 Structures

**Arena**
A structure that is shared among one or more threads which contains references to one or more heaps, as well as linked lists of chunks within those heaps which are "free". 
Threads assigned to each arena will allocate memory from that arena's free lists.

**Heap**
A contiguous region of memory that is subdivided into chunks to be allocated. Each heap belongs to exactly one arena.

Each arena obtains memory from one or more heaps. The main arena uses the program's initial heap (starting right after .bss et al). 
Additional arenas allocate memory for their heaps via mmap, adding more heaps to their list of heaps as older heaps are used up. 

**Chunk**
A small range of memory that can be allocated (owned by the application), freed (owned by glibc), or combined with adjacent chunks into larger ranges. 
Note that a chunk is a wrapper around the block of memory that is given to the application. Each chunk exists in one heap and belongs to one arena.

**Fast bins**
Small chunks are stored in size-specific bins. 
Chunks added to a fast bin ("fastbin") are not combined with adjacent chunks - the logic is minimal to keep access fast (hence the name). 
Chunks in the fastbins may be moved to other bins as needed. 
Fastbin chunks are stored in a single linked list, since they're all the same size and chunks in the middle of the list need never be accessed.

**Unsorted bins**
When chunks are free'd they're initially stored in a single bin. 
They're sorted later, in malloc, in order to give them one chance to be quickly re-used. 
This also means that the sorting logic only needs to exist at one point - everyone else just puts free'd chunks into this bin, and they'll get sorted later. 
The "unsorted" bin is simply the first of the regular bins.

**Small bins**
The normal bins are divided into "small" bins, where each chunk is the same size, and "large" bins, where chunks are a range of sizes. 
When a chunk is added to these bins, they're first combined with adjacent chunks to "coalesce" them into larger chunks. 
Thus, these chunks are never adjacent to other such chunks (although they may be adjacent to fast or unsorted chunks, and of course in-use chunks). 
Small and large chunks are doubly-linked so that chunks may be removed from the middle (such as when they're combined with newly free'd chunks).

**Large bins**
A chunk is "large" if its bin may contain more than one size. 
For small bins, you can pick the first chunk and just use it. 
For large bins, you have to find the "best" chunk, and possibly split it into two chunks (one the size you need, and one for the remainder).

**Top chunk**
Each arena keeps track of a special "top" chunk, which is typically the biggest available chunk.

**Recently allocated heap**
Each arena keeps track of a chunk refers to the most recently allocated heap.

![Arenas and heaps](https://williammuji.github.io/images/heaps-and-arenas.png)

![Arena and bins](https://williammuji.github.io/images/arena_and_bins.png)

### 3.2 Thread Local Cache(tcache)

Each thread has a thread-local variable that remembers which arena it last used. 
If that arena is in use when a thread needs to use it the thread will block to wait for the arena to become free. 
If the thread has never used an arena before then it may try to reuse an unused one, create a new one, or pick the next one on the global list.

Each thread has a per-thread cache (called the tcache) containing a small collection of chunks which can be accessed without needing to lock an arena. 
These chunks are stored as an array of singly-linked lists, like fastbins, but with links pointing to the payload (user area) not the chunk header. 
Each bin contains one size chunk, so the array is indexed (indirectly) by chunk size. 
Unlike fastbins, the tcache is limited in how many chunks are allowed in each bin (tcache_count). 
If the tcache bin is empty for a given requested size, the next larger sized chunk is not used (could cause internal fragmentation), instead the fallback is to use the normal malloc routines i.e. locking the thread's arena and working from there.

![tcache](https://williammuji.github.io/images/per-thread-cache.png)

### 3.3 Platform-specific Thresholds and Constants

Parameter | 32 bit | i386 | 64 bit
----------|--------|------|--------
MALLOC_ALIGNMENT | 8 | 16 | 16
MIN_CHUNK_SIZE | 16 | 16 | 32
MAX_FAST_SIZE | 80 | 80 | 160
MAX_TCACHE_SIZE | 516 | 1020 | 1032
MIN_LARGE_SIZE | 512 | 1008 | 1024
DEFAULT_MMAP_THRESHOLD | 131,072 | 131,072 | 131,072
DEFAULT_MMAP_THRESHOLD_MAX | 524,288 | 524,288 | 33,554,432
HEAP_MIN_SIZE | 32,768 | 32,768 | 32,768
HEAP_MAX_SIZE | 1,048,576 | 1,048,576 | 67,108,864


### 3.4 Malloc & Free Algotithm

**Malloc**
1. If there is a suitable (exact match only) chunk in the tcache, it is returned to the caller. 
No attempt is made to use an available chunk from a larger-sized bin.
2. If the request is large enough, mmap() is used to request memory directly from the operating system. 
Note that the threshold for mmap'ing is dynamic, unless overridden by M_MMAP_THRESHOLD (see mallopt() documentation), and there may be a limit to how many such mappings there can be at one time.
3. If the appropriate fastbin has a chunk in it, use that. If additional chunks are available, also pre-fill the tcache.
4. If the appropriate smallbin has a chunk in it, use that, possibly pre-filling the tcache here also.
5. If the request is "large", take a moment to take everything in the fastbins and move them to the unsorted bin, coalescing them as you go.
6. Start taking chunks off the unsorted list, and moving them to small/large bins, coalescing as you go (note that this is the only place in the code that puts chunks into the small/large bins). If a chunk of the right size is seen, use that.
7. If the request is "large", search the appropriate large bin, and successively larger bins, until a large-enough chunk is found.
8. If we still have chunks in the fastbins (this may happen for "small" requests), consolidate those and repeat the previous two steps.
9. Split off part of the "top" chunk, possibly enlarging "top" beforehand.

**Free**
1. If there is room in the tcache, store the chunk there and return.
2. If the chunk is small enough, place it in the appropriate fastbin.
3. If the chunk was mmap'd, munmap it.
4. See if this chunk is adjacent to another free chunk and coalesce if it is.
5. Place the chunk in the unsorted list, unless it's now the "top" chunk.
6. If the chunk is large enough, coalesce any fastbins and see if the top chunk is large enough to give some memory back to the system. 
Note that this step might be deferred, for performance reasons, and happen during a malloc or other call.

## 4. TCMalloc allocator

### 4.1 Motivation

TCMalloc is a memory allocator designed as an alternative to the system default allocator that has the following characteristics:
1. Fast, uncontended allocation and deallocation for most objects. 
Objects are cached, depending on mode, either per-thread, or per-logical-CPU. 
Most allocations do not need to take locks, so there is low contention and good scaling for multi-threaded applications.
2. Flexible use of memory, so freed memory can be reused for different object sizes, or returned to the OS.
3. Low per object memory overhead by allocating "pages" of objects of the same size. Leading to space-efficient representation of small objects.
4. Low overhead sampling, enabling detailed insight into applications memory usage.

![TCMalloc internals](https://williammuji.github.io/images/tcmalloc-internals.png)

### 4.2 Front-end (Per-thread/Per-CPU)

The front-end handles a request for memory of a particular size. 
The front-end has a cache of memory that it can use for allocation or to hold free memory. 
This cache is only accessible by a single thread at a time, so it does not require any locks, hence most allocations and deallocations are fast.

The front-end will satisfy any request if it has cached memory of the appropriate size. 
If the cache for that particular size is empty, the front-end will request a batch of memory from the middle-end to refill the cache. 
The middle-end comprises the CentralFreeList and the TransferCache.

If the middle-end is exhausted, or if the requested size is greater than the maximum size that the front-end caches, 
a request will go to the back-end to either satisfy the large allocation, or to refill the caches in the middle-end. 

**Small and Large Object Allocation**

Allocations of “small” objects are mapped onto to one of 60-80 allocatable size-classes.
The size-classes are designed to minimize the amount of memory that is wasted when rounding to the next largest size class. 

When an object of a given size is requested, that request is mapped to a request of a particular class-size, and the returned memory is from that size-class. 
This means that the returned memory is at least as large as the requested size. 
These class-sized allocations are handled by the front-end.

Objects of size greater than the limit defined by kMaxSize are allocated directly from the backend. 
As such they are not cached in either the front or middle ends. Allocation requests for large object sizes are rounded up to the TCMalloc page size.

When an object is deallocated, the compiler will provide the size of the object if it is known at compile time. 
If the size is not known, it will be looked up in the pagemap. 
If the object is small it will be put back into the front-end cache. 
If the object is larger than kMaxSize it is returned directly to the pageheap.

**Per-CPU Mode**

![Per-CPU Mode](https://williammuji.github.io/images/per-cpu-cache-internals.png)

When an object of a particular class-size is requested it is removed from this array, when the object is freed it is added to the array. 
If the array is exhausted the array is refilled using a batch of objects from the middle-end. 
If the array would overflow, a batch of objects are removed from the array and returned to the middle-end.

**Restartable Sequences**

To work correctly, per-CPU mode relies on restartable sequences (man rseq(2)). 
A restartable sequence is just a block of (assembly language) instructions, largely like a typical function. 
A restriction of restartable sequences is that they cannot write partial state to memory, the final instruction must be a single write of the updated state. 

The idea of restartable sequences is that if a thread is removed from a CPU (e.g. context switched) while it is executing a restartable sequence, the sequence will be restarted from the top. 
Hence the sequence will either complete without interruption, or be repeatedly restarted until it completes without interruption. 
This is achieved without using any locking or atomic instructions, thereby avoiding any contention in the sequence itself.

**Per-Thread Mode**

![Per-Thread Mode](https://williammuji.github.io/images/per-thread-structure.png)

In per-thread mode, TCMalloc assigns each thread a thread-local cache. 
Small allocations are satisfied from this thread-local cache. 
Objects are moved between the middle-end into and out of the thread-local cache as needed.

A thread cache contains one singly linked list of free objects per size-class (so if there are N class-sizes, there will be N corresponding linked lists).

On allocation an object is removed from the appropriate size-class of the per-thread caches. 
On deallocation, the object is prepended to the appropriate size-class. 
Underflow and overflow are handled by accessing the middle-end to either fetch more objects, or to return some objects.

### 4.3 Middle-end

The middle-end is responsible for providing memory to the front-end and returning memory to the back-end. 
The middle-end comprises the Transfer cache and the Central free list. 
These caches are each protected by a mutex lock - so there is a serialization cost to accessing them.

**Transfer Cache**

When the front-end requests memory, or returns memory, it will reach out to the transfer cache.
The transfer cache holds an array of pointers to free memory, and it is quick to move objects into this array, or fetch objects from this array on behalf of the front-end.
The transfer cache gets its name from situations where one thread is allocating memory that is deallocated by another thread. 
The transfer cache allows memory to rapidly flow between two different threads.
If the transfer cache is unable to satisfy the memory request, or has insufficient space to hold the returned objects, it will access the central free list.

**Central Free List**

The central free list manages memory in “spans”, a span is a collection of one or more “TCMalloc pages” of memory.
A request for one or more objects is satisfied by the central free list by extracting objects from spans until the request is satisfied. 
If there are insufficient available objects in the spans, more spans are requested from the back-end. 
When objects are returned to the central free list, each object is mapped to the span to which it belongs (using the pagemap and then released into that span). 
If all the objects that reside in a particular span are returned to it, the entire span gets returned to the back-end.

**Pagemap and Spans**

The heap managed by TCMalloc is divided into pages of a compile-time determined size. 
A run of contiguous pages is represented by a Span object. 
A span can be used to manage a large object that has been handed off to the application, or a run of pages that have been split up into a sequence of small objects. 
If the span manages small objects, the size-class of the objects is recorded in the span.

The pagemap is used to look up the span to which an object belongs, or to identify the class-size for a given object.

TCMalloc has a pagemap which maps a virtual address onto the structures that manage the objects in that address range. 
Larger pages mean that the pagemap needs fewer entries and is therefore smaller.

Spans are used in the middle-end to determine where to place returned objects, and in the back-end to manage the handling of page ranges.

![Pagemap](https://williammuji.github.io/images/pagemap.png)

**Storing Small Objects in Spans**

A span contains a pointer to the base of the TCMalloc pages that the span controls. 
For small objects those pages are divided into at most pow(2, 16) objects. 
This value is selected so that within the span we can refer to objects by a two-byte index.

This means that we can use an unrolled linked list to hold the objects. 
For example, if we have eight byte objects we can store the indexes of three ready-to-use objects, and use the forth slot to store the index of the next object in the chain. 
This data structure reduces cache misses over a fully linked list.

When we have no available objects for a class-size we need to fetch a new span from the pageheap and populate it.

像slab。

### 4.4 Back-end

Three jobs:
1. It manages large chunks of unused memory.
2. It is responsible for fetching memory from the OS when there is no suitably sized memory available to fulfill an allocation request.
3. It is responsible for returning unneeded memory back to the OS.

Two backends:
1. The Legacy pageheap which manages memory in TCMalloc page sized chunks.
2. The hugepage aware pageheap which manages memory in chunks of hugepage sizes. 
Managing memory in hugepage chunks enables the allocator to improve application performance by reducing TLB misses.

**Legacy Pageheap**

![Leagacy pageheap](https://williammuji.github.io/images/legacy_pageheap.png)

An allocation for k pages is satisfied by looking in the kth free list. 
If that free list is empty, we look in the next free list, and so forth. 
Eventually, we look in the last free list if necessary. If that fails, we fetch memory from the system mmap.

If an allocation for k pages is satisfied by a run of pages of length > k , the remainder of the run is re-inserted back into the appropriate free list in the pageheap.

When a range of pages are returned to the pageheap, the adjacent pages are checked to determine if they now form a contiguous region, 
if that is the case then the pages are concatenated and placed into the appropriate free list.

像buddy algorithm。


**Hugepage Aware Allocator**

The objective of the hugepage aware allocator is to hold memory in hugepage size chunks. On x86 a hugepage is 2MiB in size. 

Three different caches:
1. The filler cache holds hugepages which have had some memory allocated from them. 
This can be considered to be similar to the legacy pageheap in that it holds linked lists of memory of a particular number of TCMalloc pages. 
Allocation requests for sizes of less than a hugepage in size are (typically) returned from the filler cache. 
If the filler cache does not have sufficient available memory it will request additional hugepages from which to allocate.

2. The region cache which handles allocations of greater than a hugepage. 
This cache allows allocations to straddle multiple hugepages, and packs multiple such allocations into a contiguous region. 
This is particularly useful for allocations that slightly exceed the size of a hugepage (for example, 2.1 MiB).

3. The hugepage cache handles large allocations of at least a hugepage. 
There is overlap in usage with the region cache, but the region cache is only enabled when it is determined (at runtime) that the allocation pattern would benefit from it.
The hugepage cache handles large allocations of at least a hugepage. There is overlap in usage with the region cache, but the region cache is only enabled when it is determined (at runtime) that the allocation pattern would benefit from it.
