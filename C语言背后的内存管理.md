# 程序运行背后的内存管理机制

:::success
 contributed by <[Felizs](https://hackmd.io/@Felizs)>
:::


## 参考资料 
[你所不知道的C Jserv](https://hackmd.io/@sysprog/c-memory?type=view) 55min
\
[What a C programmer should know about memory](https://marek.vavrusa.com/memory/
)（[札记](http://wen00072.github.io/blog/2015/08/08/notes-what-a-c-programmer-should-know-about-memory/)）
\
 [Anatomy of a Program in Memory](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)



---



可能很多人都知道[每个程序员都应该了解的内存知识](https://akkadia.org/drepper/cpumemory.pdf)这篇文章。那篇文章我也草草浏览过，写的太深入(底层过于详细)了，以我当前的知识背景只能囫囵吞枣，读完也不清楚重点。所以梳理了一下与日常工作更相关的部分。

行文主要由以下五个部分组成：
- 虚拟内存（内存空间布局介绍）
- 栈分配
- 堆分配
- 内存映射
- 内存消耗

引申出的主题会非常多，也在此处做一个记录，日后可以整理成博文。
- [ ] 指针与内存分配
- [ ] 函数与内存分配
- [ ] Cache原理
- [ ] Linux中的内存管理 


---

## 杂记
[What Every Programmer Should Know A bout Memory](https://akkadia.org/drepper/cpumemory.pdf)
非常大的主题 Drepper写了这篇文章，glibc的维护者 



## 背景知识
C语言指针
C语言函数调用

---


## 虚拟内存

我们一般在保护模式下工作，程序拥有各自的**虚拟**地址空间。**虚拟**意味着我们并不受可用内存的束缚，同时也无权拥有它们。如果需要使用虚拟地址空间，我们需要os的提供真实的物理支持，这就叫做mapping。支持可以是物理内存（不一定是RAM） 或者持久性存储。 前者称为'匿名映射'。

:::info
:warning:VMA可能会分配它并不拥有的空间，给出过度承诺（overcommiting）。每次分配后进行NULL检查是由必要的，但不能确保不出错。 因为过度承诺的存在，os可能给了你足够多的指向内存的有效指针，但当你试图访问的时候，就Out of memory了。和银行有些像。
:::

---

### 虚拟内存-整体视图
多任务操作系统的每个进程都在它自己的内存沙盒中运行。这个沙盒就是**虚拟地址空间**，在32位的模式下，它就是4GB的内存地址块。这些虚拟地址通过页表映射到物理内存，这些页表是由操作系统维护，由处理器进行查询的。 每个一个进程都有他自己的一组页表，但是存在一个陷阱。一旦虚拟空间被启用了，它们会应用于及其中运行的所有软件，包括内核本身。所以虚拟空间地址的一部分必须保留给内核。
![](https://i.imgur.com/m0xxSt0.png)
This does **not** mean the kernel uses that much physical memory, only that it has that portion of address space available to map whatever physical memory it wishes. 内核空间在页表中被标记为特权代码专有，所以当用户模式的程序想访问它的时候，会出发页错误。在Linux中，内核空间一直存在，并在所有进程中映射相同的物理内存。内核代码和数据一直是可寻址的，随时可以处理中断和系统调用。相反，每当进程切换发生时，地址空间的用户模式部分的映射都会改变。
![](https://i.imgur.com/rVC76wG.png)
above, Firefox has used far more of its virtual address space due to its legendary memory hunger

下图是Linux进程的标准段布局：
![](https://i.imgur.com/coIMHIV.png)
当计算是安全友好的时候，上图显示的段的起始地址堆几乎堆计算机中的每个进程都是一样的。而漏洞常常引用绝对的内存地址：堆栈上的地址，库函数的地址等，攻击者可以闭着眼睛找到这些位置。因此地址空间随机化已经变得非常流行。Linux会通过加入偏移量的方法随机化栈，内存映射段和堆（不幸的是32的地址空间已经非常紧凑了）

---

### 虚拟内存-栈
进程最上边的段称作栈（向下增长），存储的是本地变量和函数参数（在大部分语言中）。调用函数的时候会在栈中push一个栈帧，栈帧在函数返回的时候被摧毁。这种简单的设计是因为数据遵循严格的LIFO(Last In First Out)顺序。所以跟踪栈内容只需要简单地改变栈指针，push和pop因此非常快速且拥有确定性。此外，对栈的重用会倾向于在cpu缓存里保留活跃的栈内存。

push超过其适合范围的数据可能会耗尽栈内存，这会触发页错误。该错误在Linux中[expand_stack（）](http://lxr.linux.no/linux+v2.6.28/mm/mmap.c#L1716)处理，该错误又调用[acct_stack_growth（）](http://lxr.linux.no/linux+v2.6.28/mm/mmap.c#L1544)来检查是否适合增加堆栈。栈大小会根据需求进行调整：当栈小于RLIMIT_STACK(通常是8M)，那么它会在程序没有感知的情况下正产高增长。但是如果达到了栈的最大限制，会出现stack overflow，程序会受到段错误。

>当栈变小时，它不会收缩。// 和联邦预算一样

动态栈增长是访问未映射内存区域的唯一有效的情况。其他的访问会触发页错误->段错误。一些只读的内存区域遇到写入操作，也会发生段错误。

---

### 虚拟内存-堆
堆提供了程序运行时的内存分配，和栈一样。不同的是，它存的数据活的比函数分配的时候更长。

>原文：meant for data that must outlive the function doing the allocation

大部分语言为程序提供了堆管理。因此，满足内存请求是内核和程序的公共事务。

In C, the interface to heap allocation is [malloc()](http://www.kernel.org/doc/man-pages/online/pages/man3/malloc.3.html) and friends, whereas in a garbage-collected language like C# the interface is the `new` keyword.

如果堆有足够的空间去满足分配内存的请求时，它可以直接由语言运行时处理，而不需要内核介入。否则堆就通过brk系统调来生成空间。堆的管理是很复杂的，需要复杂的算法来应对应用程序混乱的分配模式，以提高速度和有效的内存访问。实时系统拥有专门的分配器去处理这个问题。 堆同时也是碎片化的，如下所示：
![](https://i.imgur.com/4PmzNsP.png)

---

### 虚拟内存-内存映射段
在这里，内核将文件的内容直接映射到内存。 任何Linux的程序都可以通过mmap()系统调用来请求（创建？）此类映射。内存映射是一种执行文件IO的便捷且高效能的方式，因此它用来加载动态库。也可以在这个区域创建不应用于任何文件的匿名内存映射，将其用于程序数据。 在Linux中，如果你通过malloc()请求大块内存，则C库会创建此类匿名映射，而不使用堆内存。

---

### BSS data Program text
BSS和数据段都是用来存静态（全局）变量的内容。区别在于BSS存的是未初始化的静态变量的内容。 BSS的内存区域是匿名的，它不映射任何文件。 比如声明了static int a , a就在BSS段。

![](https://i.imgur.com/gJr6kKL.png)

据段存储了指针，已被初始化的变量。这个存储区域不是匿名的，映射了程序二进制图像的一部分。即使数据段映射了一个文件，它也是一个私有的内存映射，意味着不会反应全局变量的更新。

指针指向的实际字符串，二进制文件存储在text段。
 



## 堆分配

### slab
如何最佳利用cache memory

### Demand paging 
:::info
Linux系統提供一系列的内存管理API
- 分配和释放
- 内存管理API
    - mlock 禁止换出
    - madvise - give advice about use of memory 
      他不是标准。
:::
### copy-on-write





## 栈分配
- alloca() -> automatically freed 用的是栈空间 
- VLA是指viable-length array,的实现和alloca也是在栈空间。存在安全隐患



## 内存映射

## 内存

## 内存消耗








