---
layout: post
title:  "TCMalloc JEMalloc"
date: 2020-09-09 10:00:00 +0800
categories: jekyll update
---

1. TCMalloc
2. JEMalloc


## 1. TCMalloc

### 1.1 Motivation

TCMalloc是一个内存分配器，它是系统自带内存分配器的很好替代，因为它有以下特征：

1、快，对于大部分对象都可以做到无争用分配和释放，对象根据线程或者CPU模式被缓存起来，大部分分配都不需要加锁，对于多线程应用程序，它可以做到低线程争用，且
做到了很好的扩展性。

2、内存灵活使用，已被释放的内存可以被不同大小的对象重用，或者归还给操作系统。

3、通过分配"page"给相同大小的对象，降低了每个对象的内存开销，对于小对象，内存空间占用更加高效。

4、低采样损耗，可以让应用程序方便的查看分配器内部状态。


### 1.2 Overview

![TCMalloc internals](https://williammuji.github.io/images/tcmalloc-internals.png)

1、front-end为应用程序快速分配释放内存提供缓存。可以通过线程模式，或者CPU模式。

2、middle-end负责重填front-end缓存。

3、back-end负责从操作系统获取内存。支持hugepage的pageheap，或旧式pageheap。


### 1.3 The TCMalloc Front-end

front-end负责对特定大小的内存请求，front-end有一个内存缓存来应对内存分配或回收释放内存。这个cache一次只能由一个线程访问，所以不需要加锁，因而对于大多数的内存
分配和释放都是快速的。

如果front-end上的cache有适合请求大小的内存，那么就会直接返回，如果没有，则front-end就会从middle-end请求一批内存来重填到cache上。middle-end由CentralFreeList和TransferCache
构成。

如果middle-end也用完了，或者所请求大小超过front-end caches所限制的最大大小，那么分配器就会由back-end来负责大块内存分配，或者通过重填内存到middle-end。back-end也被称为PageHeap。

front-end一般有两种实现：

1、起初，它支持每个线程一个cache，但是如果线程数上去了，内存占用将会显著增加，而现代应用程序可以支持很多线程数，这将导致所有线程占用内存总数变大，或者导致每个线程所分配的
cache变少。

2、现在可以支持per-CPU模式，系统中每个逻辑CPU都有一个cache，内存请求可以从中分配，x86架构上，逻辑CPU等效于超线程。

per-Thread和per-CPU模式的区别在与malloc/new和free/delete的实现。


### 1.4 Small and Large Object Allocation

小对象的分配一般对应60-80种不同大小的classes，比如要分配12个字节大小的一般会从16字节大小的class中获取，size-classes设计目的就是为了减少内存浪费。

可以通过变量设置对齐大小小等于8，分配器对应::operator new会分配一组8个字节对齐的raw内存。一般对齐的大小是16字节，那么通过设置这个变量对于小字节对齐有效减少内存浪费
（比如对于申请24,40个字节大小的请求等等），在很多编译器，可以通过-fnew-alignment来设置。但是对应低于16字节的请求分配，tcmalloc可以返回一个更小对齐大小的对象，因为无法在
空间中分配对齐要求较大的对象。

当请求一个特定大小的对象，这个请求会映射到一个特定大小的class-size，并从该class-size返回内存，这意味着所返回的内存至少大于请求大小，这些class-sized都是由front-end分配。

如果请求对象大小超过kMaxSize，那么会直接由backend分配，因为这样大小的对象在front-end,middle-end都不会有缓存，大对象的分配请求将匹配到TCMalloc的page大小。

### 1.5 Deallocation

如果有对象释放，编译器会在编译期间提供该对象的大小，如果不知道大小，可以通过pagemap查找，如果对象比较小，那么会放入front-end的缓存末端，如果对象比kMaxSize还大，将会直接返回
到pageheap中。

**Per-CPU Mode**

在per-CPU模式下，一次性会分配一大块内存，下图展示了slab内存如何根据CPU切分，以及每个CPU使用少量slab内存保存元数据、指向可用对象的指针。

![per-CPU cache internals](https://williammuji.github.io/images/per-cpu-cache-internals.png)

每个逻辑CPU分配有一段内存，以保存有元数据和指向可用特定大小对象的指针。元数据由每个size-class的header、block构成。header有一个指针指向每个size-class指针数组的开头，也
包含了一个当前、动态指针，最大容量以及当前位置的指针。每个按大小分类的指针数组的静态最大容量，在开始时由此大小分类的数组开始与下一个大小分类的数组开始之间的差确定。

在运行时，每个cpu块中可以存储的特定类大小的项目的最大数量将有所不同，但决不能超过启动时分配的静态确定的最大容量。

当请求特定类大小的对象时，将其从此数组中删除；释放该对象时，会将其添加到数组中。如果数组已耗尽，则使用middle-end的一批对象重新填充数组。如果数组溢出，则会从数组中删除一批对象，
然后返回到middle-end。

每个CPU通过MallocExtension::SetMaxPerCpuCacheSize参数可以设置单CPU缓存内存大小，这意味着总的缓存数量跟每个活动的CPU缓存相关。所以拥有更多CPU数的机器会有更多的cache内存。

为避免在不再运行应用程序的CPU上保留内存，请 MallocExtension::ReleaseCpuMemory释放保留在指定CPU缓存中的对象。

在CPU内部，内存通过各种size-classes来保证缓存的内存小于限制大小，注意到的是管理缓存的最大值，而不是缓存现有的量，一般管理的只有限制的一半大小。

当一个size-class的对象用完时，最大容量会增加，并且要获取更多的对象时，它会考虑增加该类大小的容量。它可以增加大小类的容量，直到cache可以容纳的总内存（对于所有类大小）
达到per-CPU限制，或者直到该大小类的容量达到该大小的硬编码大小限制为止。如果大小级别尚未达到硬编码限制，则为了增加容量，它可以从同一CPU上的另一个大小级别窃取容量。

**Legacy Per-Thread mode**

per-thread模式下，TCMalloc为每个线程分配一个线程局部cache，小的内存需求直接由线程局部cache获取，如果不够就从middle-end获取。

线程缓存由不同大小的单链表构成，如果有N个不同class-size，那么就会有N个相应的链表，如下图所示：

![per-thread cache](https://williammuji.github.io/images/per-thread-structure.png)

分配内存的时候从缓存中对应大小的size-class获取，释放的时候对象插入到对应size-class前面。通过访问middle-end以获取更多对象或返回一些对象来处理下溢和上溢。

每个线程缓存的最大容量由参数设置 MallocExtension::SetMaxTotalThreadCacheBytes。但是，总大小可能会超过该限制，因为每个每个线程缓存的最小大小KMinThreadCacheSize
通常为512KiB。如果某个线程希望增加其容量，则需要从其他线程中清除容量。

当线程退出时，其缓存的内存将返回给middle-end。

**Runtime Sizing of Front-end Caches**

优化front-end cache可用列表的大小是非常重要的。如果空闲列表太小，我们将需要经常访问middle-end中央空闲列表。如果空闲列表太大，由于对象闲置在那里，我们将浪费内存。

缓存对分配和释放一样重要作用。如果没有缓存，则每个重新分配都需要将内存移至中央空闲列表。

per-CPU和per-thread模式具有不同的动态缓存大小调整算法实现。

1、在per-thread模式下，每当需要从middle-end获取更多对象时，可存储的最大对象数量将增加到一个限制。同样，当我们发现cache了太多对象时，容量也会减少。如果cache的对象的
总大小超过每个线程的限制，缓存的大小也会减少。

2、在per-CPU模式下，空闲列表的容量会随着我们是否在下溢和上溢之间交替而增加（表示更大的缓存可能会阻止这种交替）。一段时间未增长时，容量会减少，因此可能会超出容量。


### 1.6 TCMalloc Middle-end

middle-end为front-end提供内存，并归还内存给back-end。它由Transfer cache和Central free list构成。一般这些都是单数的，对应每一个size-class，都有一个transfer cache
和一个central free list。这些cache需要锁来保证串行化访问。

**Transfer Cache**

当front-end请求内存，或释放内存，都会涉及到transfer cache。

transfer cache持有一个已释放内存的指针数组。它可以快速添加对象到这个数组，快速的从数组获取内存。

可以从一个线程获取内存，而从另外一个线程释放，这就是transfer cache得名。transfer cache允许不同线程之间快速移动。

transfer cache无法满足对应大小的内存请求，或者没有足够的空间来回收对象，那么就会访问到central free list。

**Central Free List**

Central Free List以span方式管理内存，它代表一个或多个"TCMalloc pages"。

请求一个或多个对象可以通过Central free list的span获取，如果一个span不够，就通过back-end获得多个span获取。

当对象要返回给Central free list时，每个对象都映射到它所属的span（通过pagemap释放到span)，如果一个特定span上的所有对象返回，那么整个span将会返回给back-end。

**Pagemap and Spans**

编译期间会定page的大小，TCMalloc通过page来管理heap内存，一段连续的pages组成一个span。一个span可以用来管理一个已分配给应用程序的大对象，或者它所包含的pages被切分成
一组小对象。如果span管理这些小对象，那么对象对应的size-class会被span登记。

pagemap用来查找对象属于哪个span，或标识一个指定对象的class-size。

TCMalloc使用2层或3层基数树映射span到内存。

下图展示了2级radix pagemap，它映射了对象和span之间的地址，span控制相关对象所在的pages。下图所示span A有两个pages，span B有3个pages。

![pagemap](https://williammuji.github.io/images/pagemap.png)

**Storing Small Objects in Spans**

span包含指向该span控制的TCMalloc page的底部的指针。对于小对象，这些页面最多分为pow(2,16)个对象。选择此值是为了在span内我们可以通过两个字节的索引来引用对象。

这意味着我们可以使用展开的链表来持有对象。例如，如果我们有八个字节的对象，则可以存储以3索引的即用型对象，并使用第四个插槽存储链中下一个对象的索引。这种数据结构
减少了完全链接列表上的高速缓存未命中。

使用两个字节索引的另一个优点是我们可以在span本身中使用备用容量来缓存四个对象。

当我们没有一个class-size的可用对象时，我们需要从pageheap中获取一个新的span并进行填充 。


### 1.7 TCMalloc Page Sizes

TCMalloc可以有多个page size，它跟硬件TLB所指的page不同，TCMalloc的page size可以为4KiB,8KiB,32KiB,256KiB。

TCMalloc页要么包含特定大小的多个对象，要么被用作组的一部分，以容纳大于单个页面的对象。如果整个页面都空闲了，它将返回到后端（页面堆），以后可以重新用于容纳不同
大小的对象（或返回到OS）。

对于应用程序的内存请求，小页面使用较少的开销，比如，4KiB使用一半将剩下2KiB，32KiB使用剩下16KiB，小页面也很有可能被释放掉，举个例子，4KiB页面大小可以持有8个512字节大小
的对象，32KiB页面大小可以持有64个512字节大小的对象。一般比较少见32个对象同时被回收。

如果page比较大，那么会减少到back-end分配和释放内存的操作，一个32KiB大小的page可以持有8个4KiB大小的对象，可以使用较少的large pages映射虚拟地址空间，page大小变大也使得
pagemap所需要的条目减少。

因此，对于内存占用量较小或对内存占用量大小敏感的应用程序，使用较小的TCMalloc页面大小是有意义的。具有较大内存占用空间的应用程序可能会受益于更大的TCMalloc页面大小。


### 1.8 TCMalloc Backend

back-end主要有3个作用：

1、它管理large chunks未使用的内存。

2、如果没有适合大小的内存可用，那么它负责从操作系统获取内存。

3、释放不需要的内存返回给操作系统。

TCMalloc有两种back-end。

1、传统的pageheap根据TCMalloc page大小的chunks来管理内存。

2、hugepage相关的pageheap根据hugepage大小的chunks来管理内存。使用hugepage大小的chunks使得分配器更加高效，因为它减少了TLB misses。

**Legacy pageheap**

leagacy pageheap是一个特定大小的连续的pages的free lists数组，如果k<256，数组第k个free list包含k个TCMalloc page。第256个free list则包含超过256个pages大小。

![Legacy pageheap](https://williammuji.github.io/images/legacy_pageheap.png)

如果需要分配k个pages大小，那么就可以索引数组第k个位置获取，如果那个free list为空，则查找下一个索引直到最后一个索引，实在没有就调用mmap获取内存。

如果申请k个pages大小的内存，而实际有更大pages索引的free list满足，那么剩余的pages会被重新插入到合适的pageheap位置。

如果一个区间的pages返回给pageheap，那么检查相邻的pages是否是一段连续的区间，如果是，则把这些pages合并起来并且重新插入到pageheap中。

**Hugepage Aware Allocator**

hugepage的目的就是了保持hugepage大小的chunks。x86可以支持2MiB大小的hugepage，back-end有3个不同的caches：

1、The filler cache持有了一些已分配好内存的hugepages，这有点像pageheap，它持有一些特定数量的TCMalloc pages的链表，如果一个请求小于一个hugepage，那么直接
通过the filler cache返回，如果the filler cache没有足够的内存，那么它会再申请额外的hugepages。

2、The regioncache处理大于一个hugepage的内存请求，这种cache允许跨越多个hugepages的分配请求，并且把这些分配内存打包成一个连续的region，这种cache适合于
超过一个hugepage的内存请求（比如请求2.1MiB）。

3、The hugepage cache处理至少需要一个hugepage大小的内存分配，这跟the region cache有点交叉，但是the region cache只有在运行时才有效，


### 1.9 Caveats

TCMalloc在启动时会保存一些元数据，这些元数据随着heap需求增多而扩大，特别是pagemap，它会随着虚拟地址空间增加而增加，spans也会随着active 内存pages增加而增加。
在per-CPU模式下，TCMalloc会对每个CPU保存一个slab内存，如果系统有多个逻辑CPU，那么会引起多MB空间的内存占用。

值得注意的是，TCMalloc从操作系统中以大块（通常为1 GiB区域）请求内存。地址空间是保留的，但直到使用后才由物理内存支持。由于这种方法，应用程序的VSS可能比RSS更大。
这样做的副作用是，通过限制VSS来限制应用程序的内存使用将在应用程序使用那么多物理内存之前很长时间就失败了。

不要尝试将TCMalloc加载到正在运行的二进制文件中（例如，在Java程序中使用JNI）。应用程序使用系统malloc分配一些对象，并可能尝试将它们传递给TCMalloc进行释放。
TCMalloc将无法处理此类请求。
