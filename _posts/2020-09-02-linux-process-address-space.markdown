---
layout: post
title:  "Linux process address space"
date: 2020-09-02 15:00:00 +0800
categories: jekyll update
---

1. 进程的地址空间
2. 线性区
3. 缺页异常处理程序


内核中的函数通过get_free_pages或alloc_pages从分区页框分配器中获得页框，kmem_cache_alloc或kmalloc使用slab分配器为专用或通用
对象分配块，而vmalloc或vmalloc_32获得一块非连续的内存区。如果所请求的内存区得以满足，这些函数都返回一个页描述符地址或线性
地址。

使用这些简单方法基于以下两个原因：

1、内核是操作系统中优先级最高的部分。2、内核信任自己。


给用户态进程分配内存时，情况则不同：

1、进程对动态内存请求被认为是不紧迫的。2、用户进程是不可信任的，内核必须能随时捕获用户态进程引起的寻址错误。

内核使用新的资源成功实现对进程动态内存的推迟分配，当用户态进程请求动态内存时，并没有获得请求的页框，而仅仅获得对一个新的
线性地址区间的使用权，而这一线性地址空间就成为进程地址空间的一部分。这一区间叫做线性区(memory region)。

## 1. 进程的地址空间

address space由允许进程使用的全部线性地址组成。每个进程所看到的线性地址集合是不同的，一个进程所使用的地址与另外一个进程所
使用的地址之间没有什么关系。

内核通过线性区的资源来表示线性地址区间，线性区是由起始线性地址、长度和一些访问权限来描述。


## 2. 线性区

当一个新的线性地址区间加入到进程的地址空间时，内核检查一个已经存在的线性区是否可以扩大，如果不能，就创建一个新的线性区，类似，
如果从进程的地址空间删除一个线性地址空间，内核就要调整受影响的线性区大小。有些情况下，调整大小迫使一个线性区被分成两个更小
的部分。

进程所拥有的所有线性区是通过一个简单的链表链接在一起，出现在链表中的线性区是按内存地址的升序排列的，不过每两个先行区可以由未用
的内存地址区隔开。每个vm_area_struct元素的vm_next字段指向链表的下一个元素。内核通过进程的内存描述符的mmap字段来查找线性区，其中
mmap字段指向链表中的第一个线性区描述符。

内核频繁执行的一个操作就是查找包含指定线性地址的线性区。由于链表是经过排序的，因此只要在指定线性地址之后找到一个线性区，搜索就
可以结束。然而当进程的线性区非常少的时候使用这种链表才是方便的，在链表中查找元素、插入元素、删除元素涉及到许多操作，这些操作所
花费时间与链表的长度成线性比例。

有些面向对象的数据库，或malloc的专用调试器，可能会有成百上千的线性区，这种情况下，线性区链表的管理就会变得低效，因此与内存相关
的系统调用的性能就会降低到令人无法忍受的程度。因此，Linux2.6把内存描述符存放在红黑树的数据结构中，这种结构中插入和删除一个元素
是高效的。Linux既使用了链表，也使用了红黑树。这两种数据结构包含指向同一线性区描述符的指针，当插入或删除一个线性区描述符时，内核
通过红黑树搜索前后元素，并用搜索结果快速更新链表而不用扫描链表。

![Adding or removing a linear address interval](https://williammuji.github.io/images/adding-removing-linear-address-interval.jpg)

![Descriptors releated to the address space of a process](https://williammuji.github.io/images/descriptors-process-address-space.jpg)


## 4. 缺页异常处理程序

Page Fault异常处理程序必须区分以下两种情况：由编程错误引起的异常，由引用属于进程地址空间但还尚未分配物理页框的页引起的异常。

**处理地址空间以外的错误地址**

如果address不属于进程的地址空间，那么do_page_fault函数继续执行bad_area标记处的语句。如果错误发生在用户态，则发送一个SIGSEGV信号给current
进程并结束函数。force_sig_info确信进程不忽略或阻塞SIGSEGV信号，并通过info局部变量传递附加信息的同时把该信号发送给用户态进程。info.si_code
字段已被置为SEGV_MAPPER（如果异常是由一个不存在的页框引起），或置为SEGV_ACCERR（异常是由于对现有页框的无效访问引起）。

如果异常发生在内核态：

1、异常的引起是由于把某个线性地址作为系统调用的参数传递给内核。代码会跳到一段修正代码处，这段代码的典型操作就是向当前进程发送SIGSEGV信号，
或用一个适当的出错码终止系统调用处理程序。

2、异常是因一个真正的内核缺陷所引起的。函数把CPU寄存器和内核态堆栈全部转储打印到控制台，并输出一个系统消息缓冲区，然后调用函数do_exit杀死
当前进程，这就是按所显示的消息命名的内核漏洞(kernel oops)的错误。

**处理地址空间内的错误地址**

如果address属于进程的地址空间，如果这个线性区的访问权限与引起异常的访问类型相匹配，则调用handle_mm_fault函数分配一个新的页框。如果handle_mm_fault
函数成功地给进程分配一个页框，则返回VM-FAULT-MINOR或VM-FAULT-MAJOR。VM-FAULT-MINOR表示在没有阻塞当前进程的情况下处理了缺页，这种缺页叫做minor fault。
VM-FAULT-MAJOR表示缺页迫使当前进程睡眠，阻塞当前进程的缺页就叫做主缺页major fault。函数也会返回VM-FAULT-OOM没有足够的内存，或VM-FAULT-STGBOS其他任何
错误。



![Overall scheme for the Page Fault handler](https://williammuji.github.io/images/page-fault-handler.jpg)

![The flow diagram of the Page Fault handler](https://williammuji.github.io/images/page-fault-handler-diagram.gif)
