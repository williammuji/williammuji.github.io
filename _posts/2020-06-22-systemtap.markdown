---
layout: post
title:  "Systemtap 分析heap占用"
date: 2020-06-22 17:03:07 +0800
categories: jekyll update
---

1. 安装
2. 测试
3. 效果图
4. 分析
5. 进程内存分布

## 1.安装

 - 1.1 官方安装

[Systemtap Get Involved](https://sourceware.org/systemtap/getinvolved.html)

```shell
[root@williammuji williammuji]# uname -r
2.6.32-642.el6.x86_64

[root@williammuji williammuji]# yum install systemtap kernel-devel yum-utils
[root@williammuji williammuji]# debuginfo-install kernel

[root@williammuji williammuji]# stap -V
Systemtap translator/driver (version 2.9/0.164, rpm 2.9-4.el6)
Copyright (C) 2005-2015 Red Hat, Inc. and others
This is free software; see the source for copying conditions.
enabled features: AVAHI LIBRPM LIBSQLITE3 NLS NSS TR1_UNORDERED_MAP
```

 - 1.2 手动安装

[https://sourceware.org/systemtap/ftp/releases/](https://sourceware.org/systemtap/ftp/releases/)

[Build Systemtap](https://openresty.org/en/build-systemtap.html)

```shell
wget https://sourceware.org/systemtap/ftp/releases/systemtap-4.3.tar.gz
tar -xvf systemtap-4.3.tar.gz
cd systemtap-4.3/
./configure --prefix=/opt/stap --disable-docs \
            --disable-publican --disable-refdocs CFLAGS="-g -O2"
make -j8   # the -j8 option assumes you have about 8 logical CPU cores available
sudo make install
```

 - 1.3 遇到问题:

[systemtap如何跟踪libc.so](https://blog.yufeng.info/archives/2033)

若没有内核 symbol，需要手动找到对应$(uname -r)的debuginfo
```
# sudo rpm -i kernel-debuginfo-xxx.x86_64.rpm
```
安装glibc symbol
```
# sudo rpm -i glibc-debuginfo-xxx.x86_64.rpm
```

## 2.测试

 - 2.1 leaks.stp

[leaks.stp](http://agentzh.org/misc/leaks.stp)

leak = malloc - free 
服务器启动时，统计的就是heap占用。服务器运行时，统计的就是内存泄漏。

```C
global ptr2bt
global ptr2size
global bt_stats
global quit

probe begin {
    warn("Start tracing. Wait for 10 sec to complete.\n")
}

probe process("/lib*/libc.so*").function("malloc").return {
    if (pid() == target()) {
        if (quit) {
            foreach (bt in bt_stats) {
                print_ustack(bt)
                printf("\t%d\n", @sum(bt_stats[bt]))
            }

            exit()

        } else {

            //printf("malloc: %p (bytes %d)\n", $return, $bytes)
            ptr = $return
            bt = ubacktrace()
            ptr2bt[ptr] = bt
            ptr2size[ptr] = $bytes
            bt_stats[bt] <<< $bytes
        }
    }
}

probe process("/lib*/libc.so*").function("free") {
    if (pid() == target()) {
        //printf("free: %p\n", $mem)
        ptr = $mem

        bt = ptr2bt[ptr]
        delete ptr2bt[ptr]

        bytes = ptr2size[ptr]
        delete ptr2size[ptr]

        bt_stats[bt] <<< -bytes
        if (@sum(bt_stats[bt]) == 0) {
            delete bt_stats[bt]
        }
    }
}

probe timer.s(10) {
    quit = 1
    delete ptr2bt
    delete ptr2size
}

```
 
 - 2.2 火焰图

[FlameGraph](https://github.com/brendangregg/FlameGraph)

[http://www.brendangregg.com/flamegraphs.html](http://www.brendangregg.com/flamegraphs.html)


 - 2.3 测试TestServer

```
# /opt/stap/bin/stap leaks.stp -v --all-modules --ldd -d TestServer \
 -DSTP_NO_OVERLOAD -DMAXACTION=1024000 -DMAXTRYACTION=1024000 -DMAXMAPENTRIES=10240000 \
 -DSTP_NO_BUILD_CHECK -c "TestServer -f /log/testserver.log" > stack_count

# stackcollapse-stap.pl < stack_count | c++filt | flamegraph.pl --color="mem" \
 --title="malloc() Flame Graph" --countname="bytes" > memory_leaks.svg
```

参数说明：
> --ldd
>
> Add symbol/unwind information for all user-space shared libraries suspected by ldd to be necessary for user-space binaries being probed or listed  with  the  -d
> option.  Caution: this can make the probe modules considerably larger.  Note that this option does not deal with kernel-space modules: see instead --all-modules
> below.

> --all-modules
>
> Equivalent to specifying "-dkernel" and a "-d" for each kernel module that is currently loaded.  Caution: this can make the probe modules considerably larger.


> MAXACTION
>
> Maximum  number  of  statements  to execute during any single probe hit (with interrupts disabled), default 1000.  Note that for straight-through probe handlers
> lacking loops or recursion, due to optimization, this parameter may be interpreted too conservatively.

> MAXMAPENTRIES
>
> Maximum number of rows in any single global array, default 2048.  Individual arrays may be declared with a larger or smaller limit instead:
>
> global big[10000],little[5]
>
> or denoted with % to make them wrap-around (replace old entries) automatically, as in
>
> global big%
>
> or both.

> MAXSKIPPED
> Maximum  number  of  skipped  probes before an exit is triggered, default 100.  Running systemtap with -t (timing) mode gives more details about skipped probes.
> With the default -DINTERRUPTIBLE=1 setting, probes skipped due to reentrancy are not accumulated against this limit.  Note that with the  --suppress-handler-errors
> option, this limit is not enforced.

> STP_NO_OVERLOAD
>
> By default, overload processing is turned on for all modules. If you would like to disable overload processing, define STP_NO_OVERLOAD

> STP_NO_BUILDID_CHECK
>
> To disable build-id verification errors, if one is confident that they are an artefact of build accidents rather than a real mismatch,
> one might try the -DSTP_NO_BUILDID_CHECK option.

## 3.效果图

![Image of memory_leaks](http://williammuji.github.io:4000/images/memory_leaks_flame_graph.jpg)

## 4.分析

```
PID     USER            PR      NI      VIRT    RES     SHR     S       %CPU    %MEM    TIME+           COMMAND
14170   williammuji     20      0       2413080 1.216g  19932   S       20.0    1.0     00:10:12        TestServer
```

```bash
[root@williammuji williammuji]# pmap -X 14170
Address         Perm    Offset	  Inode              Size    Rss  Mapping
00200000        r--p    00000000  1739229210        11252    1308 TestServer
00cfd000        r-xp    00afd000  1739229210        26716   16564 TestServer
02714000        r--p    02514000  1739229210           60      60 TestServer
02723000        rw-p    02523000  1739229210           36      16 TestServer
0272c000        rw-p    00000000  	   0         2784    1480
035bc000        rw-p    00000000           0          732     668 [heap]
03673000        rw-p    00000000           0      1044212 1044132 [heap]

7f1b8934e000    rw-p    00000000           0         8192      20 [stack:14444]
7f1b89b4f000    rw-p    00000000           0        21176   12596 [stack:14443]
7f1b8b800000    rw-p    00000000           0         8192    2048 [stack:14417]
7f1b945fb000    rw-p    00000000           0        59412   53248 [stack:14403]
7f1ba41a1000    rw-p    00000000           0        28680   22156 [stack:14402]
7f1ba5da4000    rw-p    00000000           0         8192      16 [stack:14367]
7f1ba65a5000    rw-p    00000000           0        10596    4088 [stack:14366]
7f1ba6fff000    rw-p    00000000           0         8192      24 [stack:14361]
7f1ba7800000    rw-p    00000000           0         8192    2048 [stack:14325]
7f1bc463b000    rw-p    00000000           0         8192       8 [stack:14178]
7f1bc4e3c000    rw-p    00000000           0         8192    2064 [stack:14177]
7f1bc563d000    rw-p    00000000           0         8192       8 [stack:14176]
7f1bc5e3e000    rw-p    00000000           0         8192       8 [stack:14175]
7f1bc663f000    rw-p    00000000           0         8192       8 [stack:14174]
7f1bc6e40000    rw-p    00000000           0         8192       8 [stack:14173]
7f1bc7641000    rw-p    00000000           0         8192    2184 [stack:14172]

7f1bc8aba000    r-xp    00000000  8053067781         1804     796 libc-2.17.so
7f1bc8c7d000    ---p    001c3000  8053067781         2048       0 libc-2.17.so
7f1bc8e7d000    r--p    001c3000  8053067781           16      16 libc-2.17.so
7f1bc8e81000    rw-p    001c7000  8053067781            8       8 libc-2.17.so

7fff101a6000    rw-p    00000000  	   0          304     116 [stack]
7fff101f2000	r-xp	00000000	   0	        8       8 [vdso]
ffffffffff600000        00000000           0            4       0 [vsyscall]
                                    		 ======== =======
		                                  1275356 1256992 KB
```

- top 查看 TestServer RES常驻内存1.216GiB = 1,305,670,057Bytes

- pmap 查看 TestServer Rss常驻内存1256992KB = 1,256,992,000Bytes 

- low address是应用程序空间，再往上是heap空间，再往上是mmap空间，其中包含了非主线程的stack:xxxxx，以及应用程序
所依赖的libxxx.so空间，再往上是主线程的stack。

- The vdso and vsyscall segments are two mechanisms used to accelerate certain system calls in Linux.

> [What are vdso and vsyscall](https://stackoverflow.com/questions/19938324/what-are-vdso-and-vsyscall) 
> 
> vsyscall, which was added as a way to execute specific system calls which do not need any real level of 
privilege to run in order to reduce the system call overhead.
> 
> The vDSO (Virtual Dynamically linked Shared Objects) is a memory area allocated in user space which exposes
some kernel functionalities at user space in a safe manner. 

- vsyscall 位于kernel空间

ffffffffff600000 |  -10    MB | ffffffffff600fff |    4 kB | legacy vsyscall ABI

- heap占用 1,044,212KB + 668KB

- stack共占用 100,648KB，17个线程stack，有的线程stack超过8MiB

> [Default stack size for pthreads](https://unix.stackexchange.com/questions/127602/default-stack-size-for-pthreads)
>
> On UNIX/Linux, getrlimit(RLIMIT_STACK) is only guaranteed to give the size of the main thread\'s stack.
> 
> 其他线程栈空间是在pthread_create创建时，mmap分配。Allocations performed using mmap(2) are unaffected by the RLIMIT_DATA resource limit (see getrlimit(2)).
> 
> Under the NPTL threading implementation, if the RLIMIT_STACK soft resource limit at the time the program started has any value other than "unlimited", then it determines the default stack size of new threads.


```shell
[root@williammuji williammuji]# ulimit -s
8192
[root@williammuji williammuji]# cat /proc/14170/limits
Limit           Soft Limit      Hard Limit      Units
Max stack size  8388608         unlimited       bytes
```

## 5.进程内存分布

 - x86_32

用户空间3G，kernel空间1G

[Anatomy of a Program in Memory](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)

[How The Kernel Manages Your Memory](https://manybutfinite.com/post/how-the-kernel-manages-your-memory/)


![x86_32](https://williammuji.github.io:4000/images/x86_32_memory_layout.png)


 - x86_64

用户空间128TB，kernel空间128TB(4-level page tables,48-bits addresses)

用户空间64PB，kernel空间64PB(5-level page tables,56-bits addresses)

[Memory Management](https://www.kernel.org/doc/html/latest/x86/x86_64/mm.html)

下图为48-bits addresses:

![x86_64](https://williammuji.github.io:4000/images/x86_64_memory_layout.png)

48-bits 
![x86_64 48-bits](https://williammuji.github.io:4000/images/x86_64_48_bit.png)
56-bits
![x86_64 56-bits](https://williammuji.github.io:4000/images/x86_64_56_bit.png)
