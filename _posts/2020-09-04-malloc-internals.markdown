---
layout: post
title:  "Malloc internals"
date: 2020-09-04 16:00:00 +0800
categories: jekyll update
---

1. Overview of Malloc
2. What is a Chunk
3. Arenas and Heaps
4. Thread Local Cache(tcache)
5. Malloc Algorithm
6. Free Algorithm
7. Realloc Algorithm
8. Switching arenas
9. Platform-specific Thresholds and Constants
10. Defect


## 1. Overview of Malloc

glibc malloc库提供了应用程序地址空间内的内存分配函数。它继承自ptmalloc(pthreads malloc)，ptmalloc又继承自dlmalloc(Doug Lea malloc)。这种
类型的malloc分配一般在一大段heap中分配不同大小的chunks。在早期，一般一个应用程序一个heap，而glibc malloc库可支持多个heaps，每个heap可以
在其地址空间上动态扩展。有以下几个概念：

**Arena**

多线程可以共用Arena，每个Arena包含一个或多个heaps的refrences，还包括这些heaps内空闲的chunks的链表。多线程通过Arena内空闲链表来分配内存。

**Heap**

一块连续的内存区，它切分了很多chunk用作内存分配，每个heap属于一个arena。

**Chunk**

一小块内存区，可以用于应用程序分配(allocated)，glibc来回收(freed)，或者相邻的chunks可以联合变成更大的chunk。chunk对于应用程序就是一块
内存的封装。每个chunk属于一个heap，且属于一个Arena。

**Memory**

一块应用程序地址空间，一般由RAM或者swap页框对应。

## 2. What is a Chunk?

Glibc's malloc是面向chunk设计。它切分了一大块内存(heap)为多个不同大小的chunks。每块chunk的header包含了chunk size、相邻的chunk信息等。如果应用
程序使用了一个chunk，那么会记录该chunk的size。如果该chunk被应用程序free，chunk所对应的内存就会被重新登记一些额外的有关arena相关信息，比如arena
里链表所保存的指针，这样如果有需要，大小合适的chunks就会迅速被重新分配。被freed的chunk最后一个字包含了该chunk size。

在malloc库中，chunk指针或者mchunkptr并不是指向chunk的开始段，而是指向了上一个chunk的最后一个word，如果上一个chunk没有free，mchunkptr的第一个字段
是无效的。

每个chunk大小是8个字节的倍数，3LSBs大小的chunk size用作标志位。

**A(0x04)**

标识已分配的Arena，一个应用程序对应一个main arena，其他arenas一般使用mmap堆分配内存。如果该bit为0，那么这个chunk是从main arena的heap分配，如果该
bit为1，那么这个chunk来自于mmap映射的arena内存，且所在的heap可以由chunk地址计算获取。

**M(0x02)**

标识mmap分配的chunk，该chunk来自于一个单独的mmap调用分配的内存，它不来自任何一个heap。

**P(0x01)**

标识前一个chunk正在使用中，如果应用程序还使用前一个chunk，那么prev_size是无效的，fastbins一般会把该标志位设上，而不管应用程序是否释放该chunk。这个
标志意味着前一个chunk不应该被合并或者被用作重新分配内存。因为它代表这应用程序使用中，或者是需要在malloc代码层面上的优化。

chunk一般最小是4*sizoef(void*)大小，如果平台ABI要求，chunk最小大小可能因为额外的对齐要求会变得更大。

![In-use Free Chunk](https://williammuji.github.io/images/used-free-chunk.png)

注意到，chunks在内存中相邻，所以如果知道堆中第一块chunk地址，那么可以通过chunk字段的size信息遍历heap中的所有chunks，但这也仅在递增的地址条件下有效，
如果先是遍历到heap中的最后一个chunk，那么遍历会比较困难。

已allocated的heap一般是2的倍数对齐，如果heap中一块chunk被allocated，那么heap_info的地址就可以通过chunk的地址计算处。


## 3. Arenas and Heaps

为了支持多线程应用程序，glibc库的malloc允许多块内存区处于active状态，因而，不同线程可以同时访问不同区域的内存，且不会干扰彼此。这些内存区就称为
arena，每个应用程序有一个主arena，它跟应用程序初始化的heap相关，malloc代码有一个静态变量指向这个主arena，每个arena有一个next指针链接到其他的arenas。

为了应付多线程访问collision，一般通过mmap来创建额外的arenas来减轻压力，一般设计arena的容量为cpu核数的8倍，这意味着重度繁忙多线程应用程序依然有争用
情况，但是好处多个arena可以减少碎片(fragmentation)产生。

每个arena结构包含一个mutex，用于控制对arena的访问，对于fastbins的访问则不需要mutex，因为它的原子化操作。其他的操作需要该线程对arena加锁，正是因为多线程
对锁的争用，所以设计了多个arenas，这样访问不同arena的线程就不会耗在等待锁的释放上，线程会自动切换到未用（未加锁）的arena上。

每个arena从一个或多个heap的内存获取内存，主arena使用程序初始化的heap（.bss段结束开始的那部分heap），其他的arena则通过mmap来分配内存，如果该块内存消耗完，就再
追加分配内存并添加到已有的heap链表中，每个arena保存了一个特别的"top" chunk，它一般是最大的可用的chunk，arena同时也保存最近已分配的heap。

对于已分配的arena，内存可以方便的从初始化的heap获取。

![Heaps and Arenas](https://williammuji.github.io/images/heaps-and-arenas.png)

在每个arena中，chunks要么被应用程序占用，要么被释放，对于使用中的chunks，arena一般不追踪，空闲的chunks一般根据大小或者分配历史存储在不同的链表中，这样库
就可以快速的找到适合的chunk来满足对应的分配需求，这些列表，被称为"bins":

**Fast**

小块的chunks被存储在特定大小的bins中，添加到fastbin中的chunks不允许被合并，因为它为了保证快速的访问内存，fastbins中的chunk有可能被移动到其他bins中，fastbin
中的chunks被存储到一个链表中，该链表中的每个chunk大小都相同，且不会访问到链表中间部位。

一般有10种大小的fast bins。每个fast bin都是一个单链表，添加和删除都发生在链表的头部（后进先出）。每个bin中的chunk size都一样，10种bin 大小分别为16,24,32,40,48,
56,64,72,80,88。这些大小包含了元数据，保存chunk至少需要4个字节（如果平台支持指针大小为4个字节），chunk中的prev_size和size包含了已分配chunk的元数据，下一块chunk
的prev_size字段持有user data。fast bin中的连续free chunk不会被合并。

```cpp
typedef struct malloc_chunk *mfastbinptr;

mfastbinptr fastbinsY[]; // Array of pointers to chunks
```

**Unsorted**

当chunks被释放后它们被储存到一个单独的bin中，它们随后会根据大小来排序，以便更快的被重新使用，这也意味着所有人只需要把已释放的chunk丢到该bin中，bin会在一个合适
的时间对这些chunks进行排序，Unsorted bin是常规bins中的第一个。

unsorted bin只有一个，small 或者 large bins，如果被释放，就会放入unsorted bin中，这个bin的主要目的是缓存，用来加速allocation和deallocation请求。

```cpp
typedef struct malloc_chunk* mchunkptr;

mchunkptr bins[]; // Array of pointers to chunks
```

**Small**

常规的bins被分为"small" bins和"large" bins，small bins中每个chunk是同样大小，large bins中每个chunk大小不等。当一个chunk要添加进large bins中，它会检查相邻的chunk
是否可以合并，如果可以就把合并后的chunk添加进large bins中，对于small bins中的chunk则永远不会合并，不管它们原先处于fast或者是unsorted chunk，或者是正在被占用中的
chunk，small 和 large bins都是使用一个双向链表，这样方便删除链表中间的chunk。

一般有62个small bins，small bins比large bins速度要快，但是比fast bins慢，每个small bins维护一个双向链表，一般在HEAD插入数据，在TAIL删除数据（先进先出）。跟fast 
bins一样，small bins中每个bin 都包含一样大小的chunk，62个bins chunk大小分别为16,24,...,504字节，如果被释放，那么small chunks被先看是否可以合并，然后再丢入unsorted
 bins中。


**Large**

large bins中的chunk大小不一，对于small bins，你可以挑第一个chunk使用，但对于large bins，你需要挑最合适大小的chunk，很可能你会对该chunk切分成两个chunk，一份是你
所请求的大小，另一份保留他用。

应用进程程序初始化的时候，small和large bins都为空。在bins数组中，每个bin都由两个字段代表，HEAD指针指向列表第一个指针，TAIL指针指向列表最后一个指针，但fast bins是
单链表，所以TAIL指针为空。

一般有63个large bins，每个bin维持一个双向链表，每个bin中的chunk大小不一，逆序排序，插入和删除根据大小处理。对于large chunk的释放，一样看是否可以合并，然后再丢入
unsorted bins中，

```
1st bin: 512 - 568 bytes
2nd bin: 576 - 632 bytes
...
```

最后还有两种特殊的chunk，Top chunk和Last remainder chunk。

Top chunk一般在arena的顶部，对于malloc请求，top chunk一般是最后的响应，如果还不够，需要更大的内存需求，arena可以通过系统调用sbrk扩容。

Last remainder chunk一般从最后一次切分获取，一般对应需求大小的chunk不存在，就要从更大的chunk中切分获取，那么切分剩余的部分就成为了last remainder chunk。


![Free, Unsorted, Small, and Large Bin Chains](https://williammuji.github.io/images/bin-chains.png)

注意在上图以及下图中，所有指针指向一块chunk(mchunkptr)，因为bins不是包含chunks，而是fwd/bck指针的数组，通过mchunkptr可以获得chunk-like object，那么通过mchunkptr中的
fwd和bck字段可以用作访问合适bin。

对于large bins，因为要找到最合适大小的chunk，large chunks一般再包含一个双向链表，链表一般从大到小排序，同样大小的chunk只有第一个chunk会被链接到链表中，
这样malloc库可以快速扫描第一个满足大小的chunk，如果多个chunks同时满足指定大小需求，一般会挑选第二个满足chunk，这样下一个大小的large chunk链表不用修改，
如果有同样大小的chunk被插入，也是同样的插入到符合大小第二个位置，原因相同。

![Large Bin "Next Size" Chains](https://williammuji.github.io/images/large-bin-chains.png)

## 4. Thread Local Cache(tcache)

malloc库是支持多线程的，malloc没有针对NUMA结构进行优化，没有针对线程局部进行协调，没有按core对线程进行排序，它假定内核会有效地处理这些问题。

每个线程有个线程局部变量记录上次使用的arena，如果那个arena此时正被使用中，那么当前线程就会阻塞等待arena锁的释放，如果当前线程未使用过任何arena，那么它会选择
一个未使用过的arena，或者创建一个arena，或者从全局列表挑选下一个arena。

每个线程都有个per-thread cache(tcache)，它包含了chunks的一些组合，使用这些chunk而不用像使用arena上的chunk需要加锁，这些chunks存在链表数组中，像fastbins，但是
这些链表指向的是payload，而不是chunk header，每个bin只包含一个size大小的chunk，所以数据是可以通过chunk size索引，不像fastbins的是，tcache通过tcache_count来控制
chunks数量，如果对于某个大小的请求，tcache指定大小链表为空，那么就第二符合大小的链表也不会被使用（如果被用到就会造成碎片），取而代之的是使用正常的malloc，比如
线程通过加锁从arena获得对应大小的内存。

![Per-thread Cache(tcache)](https://williammuji.github.io/images/per-thread-cache.png)

## 5. Malloc Algorithm

malloc算法如下：

1、如果在tcache里又对应大小的chunk，则返回该chunk，没有也不会使用tcache里更大空间的chunk。

2、如果请求比较大的内存，那么会直接调用mmap从操作系统获取内存。请求多大的内存会直接调用mmap有一个阈值，这个阈值是动态的，除非被M_MMAP_THRESHOLD(查看mallopt文档)重写，
对于mmap这种内存映射，系统可能会有个限制一次允许同时多少个映射。

3、如果找到一个合适的fastbin中有一个chunk，则返回该chunk，如果有额外的chunk可用，则预填充到tcache(FIXME)。

4、如果找到一个合适的smallbin中有一个chunk，则返回该chunk，也可能在这里预填充tcache。

5、如果请求的内存比较大，则需要花点时间把fastbins所有chunk移动到unsorted bin中，如果可以合并就合并起来。

6、把unsorted list中的chunks移出，转到small/large bins中，能合并就合并（只有通过这个途径才能把chunks放入small/large bins中），如果此时有某个chunk符合请求内存大小，
那么直接使用该chunk。

7、如果请求的内存比较大，则从large bin中找到合适的chunk，如果没有则找出更大的chunk，直到一个足够大的chunk。

8、如果此时fastbins还有chunks（这可能发生在small请求的时候)，则合并这些块并重复前面2个步骤。(FIXME)

9、拆分top chunk，可能会预先扩大top chunk。

对一个over对齐大小的malloc，比如vmalloc,pmalloc,或memalign，找到一个超大大小的chunk(使用malloc上面的算法)，并以这种方式分为两个或多个块，其中大部分的块都是对齐的，
并将该对齐部分之前和之后的份额返回给unsorted bin，待以后再用。

## 6. Free Algorithm

一般来说，free不表示内存会直接返回给操作系统，而让别的应用程序可以使用，free函数会标记内存chunk已被释放且可被重新使用，从操作系统来看，这些内存仍然属于应用程序，
但是如果heap中的top chunk，与unmapped内存相邻，变得越来越大，则该内存中部分可能会被unmmap返回给操作系统。

1、如果tcache中有空间，则把chunk保存在tcache中。

2、如果chunk比较小，则把它放入fastbins中。

3、如果chunk是mmap分配的，则munmap返回。

4、查看该chunk是否有相邻的已释放的chunk，有则合并。

5、把chunk放入unsorted bins中，除非它已经是top chunk。

6、如果该chunk足够大，则合并fastbins中的chunk，再看top chunk是否足够大到可以返回一些给操作系统。因为性能原因，这部操作可能会延迟执行，可能会安插在malloc或者其他
调用操作中。

## 7. Realloc Algorithm

对于通过mmap分配的chunks，则通过mremap，它返回的内存地址可能跟原先的不同，这取决于内存。如果系统没有支持munmap，且realloc大小比老的还小，那没有任何操作，且返回原先
的内存地址，如果realloc大小比原先的大，那么就分配一块新的，把老的拷贝过去，再释放老的内存，并返回新的内存地址。

对于在arena里分配的chunks，如果新的分配大小减少到一个适当的数，则该chunk会切分两半，第一部分（老的内存地址）将被返回，另外一半则返回给arena。如果只是减少一点点，
那么会被视为同样大小的chunk。如果新的分配大小是增长的，那么它会检查相邻的chunk是否合适，如果可以或者top chunk足够大，那么这个chunk会和老的chunk合并成一个足够大的
chunk，切分成需要大小的块，这种情况下，老的地址将被返回。如果新的分配大小比老的大，且没有足够的chunk，那么就会malloc-copy-free顺序操作。

## 8. Switching arenas

一般来说一个线程所对应的arena基本不会变，除非在某些场景下：

1、线程从对应的arena分配内存失败。比如尝试合并，查找空闲列表，生成unsorted list等。又比如扩大heap，但是sbrk或者创建新的mapping失败。

2、如果之前使用的是非主arena，即mmap分配的heap，然后当前尝试arena_get_retry尝试切换到主arena（它基于sbrk），或者从主arena切换到非主arena，如果进程是单线程的，那么
从优化的目的，它不会执行成功，而返回ENOMEM。

当然在内存耗尽情况下，线程可能频繁的切换arena。

## 9. Platform-specific Thresholds and Constants

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


## 10. Defect

1、ptmalloc通过top chunk起始来shrink内存，如果top chunk的内存未释放，则top chunk以下的chunk也得不到释放。

2、多线程锁的开销比较大。

3、内存分配通过thread所对应的arena获取，但不能从一个arena移动内存到另外一个arena。这导致多线程没法负载平衡，造成内存浪费。比如说，1号线程使用了300M内存，但是glibc
不会归还给操作系统，然后2号线程又创建了个新的arena，但是1号线程的300M有无法利用到，浪费。

4、每个chunk至少8个字节(64位系统)比较浪费。

5、未规划的长时间驻存的内存分配会导致内存碎片，它不会被回收，64位系统最好分配超过32M内存（使用mmap的阈值)。
