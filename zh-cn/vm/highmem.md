> 原文：[Documentation/vm/highmem.txt](https://www.kernel.org/doc/Documentation/vm/highmem.txt)<br />
> 翻译：[@silenttung](https://github.com/dongfu8107)<br />
> 校订：[@lzufalcon](https://github.com/lzufalcon)<br />

## 目录

-    [什么是高端内存？](#toc_24146_28010_2)
-    [临时虚拟映射](#toc_24146_28010_3)
-    [使用 `KMAP_ATOMIC`](#toc_24146_28010_4)
-    [临时映射的代价](#toc_24146_28010_5)
-    [i386 PAE](#toc_24146_28010_6)


# 高端内存处理

> By: Peter Zijlstra <a.p.zijlstra@chello.nl>

<span id="toc_24146_28010_2"></span>
## 什么是高端内存？


当物理地址大小接近或者达到虚拟内存的最大值时，会使用高端内存( `highmem` )。那时对于内核来说不可能维护所有可用的物理内存映射。这意味着内核开始对要被访问的物理内存片段使用临时映射。

这部分没有被永久映射的(物理)内存就是我们所提及的"高端内存"。高端内存的边界具体位于哪里，有各种各样的体系结构相关的限制。

举个例子，在 `i386` 体系结构中，我们选择将内核映射到每个进程的 `VM` 空间，这样我们不需要在进入内核态或者从内核态退出时无效所有的 `TLB`。这意味可用的虚拟内存空间( `i386` 是 `4G` )需要在用户地址空间和内核地址空间分割。

多个体系结构传统划分方法是 `3:1`， `3GiB` 属于用户空间，高端的 `1GiB` 是内核空间:


		+--------+ 0xffffffff
		| Kernel |
		+--------+ 0xc0000000
		|        |
		| User   |
		|        |
		+--------+ 0x00000000


这意味着内核任意一次最多只能映射 `1GiB` 物理内存，但因为我们需要虚拟地址空间做其他的
事情-包括使用临时映射去访问剩余的物理内存-直接映射实际会更少(通常约 `896MiB` )。

其他用 `TLB` 来标记内存管理的体系结构有独立的内核空间和用户空间映射。然而一些硬件
(比如 `ARM` 系列)，当它们使用内存管理上下文标记时有有限的虚拟空间。


<span id="toc_24146_28010_3"></span>
## 临时虚拟映射

内核包括以下几种创建临时映射的方法：

*  `vmap()` . 该函数用来将多个物理页面长久映射到一个连续的虚拟地址空间。 `unmap` 时需要
做全局的同步。

*  `kmap()` . 这个函数允许做一个独立页面的短时映射。它需要全局同步，但是有所摊销。
 `kmap()` 以嵌入方式使用时易于死锁，所以新写的代码并不推荐使用。

*  `kmap_atomic()` . 这个函数允许对一个独立页面做非常短的映射。因为这种映射局限于
所影响的 `CPU`，所以这个函数工作的很好，但也因此所涉及的任务要求一直在原来的 `CPU` 上
执行直到任务结束, 以免其他任务替换该映射。

     `kmap_atomic()` 也可能被中断上下文所使用，因为该函数不会休眠并且引用者也是直到 `kunmap_atomic()`
被调用时才会休眠. 

      这可能被认为 `k[un]map_atomic()` 不会失败。

<span id="toc_24146_28010_4"></span>
## 使用 `KMAP_ATOMIC`

何时何地使用 `kmap_atomic()` 是简单明了的。当代码想访问从高端内存分配所分配页面的内
容(看 `__GFP_HIGHMEM` )，比如当一个页面”在页缓冲“。有两个相关的API函数，可以像如下方式使用：

    /* 寻找“感兴趣”的页面 */
    struct page *page = find_get_page(mapping, offset);

    /* 获得访问那个页面内容的地址 */
    memset(vaddr, 0, PAGE_SIZE);

    /* 解除映射那个页面 */
    kunmap_atomic(vaddr);

注意 `kunmap_atomic()` 使用的是 `kmap_atomic()` 的结果，而不是 `kmap_atomic()` 的参数.

如果需要映射两个页面因为向从一个页面复制到另一个页面，必须保证 `kmap_aotmic` 调用严格
嵌套，比如：

    vaddr1 = kmap_atomic(page1);
    vaddr2 = kmap_atomic(page2);

    memcpy(vaddr1, vaddr2, PAGE_SIZE);

    kunmap_atomic(vaddr2);
    kunmap_atomic(vaddr1);


<span id="toc_24146_28010_5"></span>
## 临时映射的代价


创建临时映射的代价可谓是很大的。框架必须管理内核的页表，数据 `TLB`，也许或者必须处理
 `MMU`的寄存器。

如果 `CONFIG_HIGHMEM` 没有选中，这是内核将尝试并且简单创建一个算术位的映射，这个映射
将转换页面结构体地址为一个指向页面内容的指针，"而不是篡改的映射"。在这种情况下，
 `unmap` 操作也许可能是一个空操作。

如果 `CONFIG_MMU` 没有被选中，就不可能有临时映射和高端内存。这种情况下，"算术方法"也将被
使用。

<span id="toc_24146_28010_6"></span>
## i386 PAE

`i386` 体系结构在一些情况下允许 `32` 位机器访问到 `64GiB RAM`。这会带来很多后果:

* `Linux` 系统里对每个页要求有一个页帧结构，并且这些页帧要以永久映射方式存在，这
意味着:

* 最多有 `896M/sizeof(struce page)` 个页帧项； 一个页结构体大小为 `32` 字节, 那将
可存储大约 `112G`的页面`; 然而kernel在内存中不仅仅需要保存页表。。。

* `PAE` 可以使得页表变的更大一些 `-` 这会使系统变慢，因为更多的数据要被访问需要 `TLB` 来回
转载等等. 一个优势是 `PAE` 有更多的 `PTE` 位并且能提供更高级的功能如 `NX` 和 `PAT`。

通常建议在 `32` 位机器上不要使用超过 `8GiB` 的内存-尽管更大的内存对你或者你的工作负载有用,
你得自己聪明一点-不要期待内核开发者当系统崩溃时真的会有多在意。
