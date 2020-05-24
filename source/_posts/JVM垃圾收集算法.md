---
title: JVM 垃圾收集算法
date: 2018-10-05 09:10:46
categories:
- 编程语言
tags:
- Java
- JVM
---

垃圾收集（Garbage collection, GC）是 Java 流行的重要原因。GC 是一种能够自动回收不用内存的机制。本质上，GC 追踪所有正在被使用的对象，并将剩余对象标记为「垃圾」（garbage）。因为程序员不需要刻意将对象标记为「可回收」，所以 Java 的 GC 被认为是自动内存管理模式。GC 以低优先级线程运行。

## 对象生命周期

JVM 中对象的生命周期可以被分为3个阶段：

1. 对象创建（Object creation）

创建对象通常使用 `new` 关键字：

```java
Object obj = new Object();

```
当对象创建完成后，操作系统会分配特定大小的内存来保存对象。分配的内存大小与操作系 统体系结构以及 JVM 种类有关。

2. 对象使用（Object in use）

在这个阶段，对象正在被应用程序的其他对象使用（其他对象有指向该对象的引用）。此时，对象被保存在内存中并且可能持有指向其他对象的引用。

3. 对象销毁（Object destruction）

GC 系统检测每个对象，并做引用计数。当没有引用指向某个对象时，挡圈运行的程序没有任何方法可以访问该对象，所以就可以回收该对象所占用的内存。

## 垃圾收集算法

对象有程序员编写的代码，以及为了使用框架所提供的特性而创建，但是不需要显式地回收内存。内存的回收由运行在 JVM 层级的垃圾收集器完成。在 JVM 的进化过程中出现了很多垃圾收集算法。

### 标记-清除（Mark and sweep）

**标记-清除** 是最初也是最基本的垃圾收集算法，它有两个阶段：

1. 标记活跃对象
2. 删除不可达对象

在开始，GC 定义了一些特定对象，被称为**根对象**（Garbage Collection Roots），如：本地变量、当前执行方法、活跃线程、类静态域的输入参数等。GC 从根对象开始，沿着指向其他对象的引用遍历内存中的所有对象，将所有访问到的对象标记为「存活」。

> 在运行标记算法时，应用程序需要暂停运行，因为无法遍历不断变化的引用图。这被称为 **Stop The World pause**。

第二个阶段清理内存。这一步可以用多种方法实现：

1. **普通删除**（Normal deletion）：释放没有被引用的对象所占用的内存，不修改被引用的对象。memory allocator 保存着指向可以创建对象的空闲内存区域。这通常被称为**标记-清除**算法。

![Normal-Deletion.png](/images/2018-10-05/Normal-Deletion-Mark-and-Sweep.png)


2. **删除-整理**（Deletion with compacting）：仅仅删除无用的对象是不够的，因为空闲内存以碎片的形式分布在内存中，如果创建较大的对象，但是却无法找到足够大的连续内存空间，则可能抛出 `OutOfMemoryError` 。
    为了解决这个问题，在删除无用对象后，会将存活的对象所占内存合并，以消除内存碎片。这使得分配新内存更容易也更快。这通常被称为**标记-整理**算法，

![Deletion-with-compacting.png](/images/2018-10-05/Deletion-with-compacting.png)

3. **删除-复制**（Deletion with copying）：这与标记-整理算法相似，它们都移动内存中存活对象的位置。不同之处在于，删除-复制算法将对象移动到不同的区域。

![Deletion-with-copying-Mark-and-Sweep.png](/images/2018-10-05/Deletion-with-copying-Mark-and-Sweep.png)


### 并发标记删除垃圾收集（Concurrent mark sweep，CMS）

CMS 在本质上是对标记清除算法的升级。它使用多个线程扫描堆内存。它可以利用现代计算机多核的结构，在性能上有显著提升。

CMS 通过与应用程序线程并发地进行垃圾收集，尝试最小化程序的停顿时间。它在新生代使用并行的**标记-复制**算法，在老年代使用并发的**标记-清除**算法。

为了开启 CMS ，需要设置 JVM 的参数：

```shell
-XX:+UserConcMarkSweepGC
```

#### CMS 的优化选项


| FLAG | DESCRIPTION |
| --- | --- |
| -XX:+UseCMSInitiating\OccupancyOnly | Indicates that you want to solely use occupancy as a criterion for starting a CMS collection operation. |
| -XX:CMSInitiating\OccupancyFraction=70 | Sets the percentage CMS generation occupancy to start a CMS collection cycle. |
| -XX:CMSTriggerRatio=70 | This is the percentage of MinHeapFreeRatio in CMS generation that is allocated prior to a CMS cycle starts. |
| -XX:CMSTriggerPermRatio=90 | Sets the percentage of MinHeapFreeRatio in the CMS permanent generation that is allocated before starting a CMS collection cycle. |
| -XX:CMSWaitDuration=2000 | Use the parameter to specify how long the CMS is allowed to wait for young collection. |
| -XX:+UseParNewGC | Elects to use the parallel algorithm for young space collection. |
| -XX:+CMSConcurrentMTEnabled | Enables the use of multiple threads for concurrent phases. |
| -XX:ConcGCThreads=2 | Sets the number of parallel threads used for the concurrent phases. |
| -XX:ParallelGCThreads=2 | Sets the number of parallel threads you want used for stop-the-world phases. |
| -XX:+CMSIncrementalMode | Enable the incremental CMS (iCMS) mode. |
| -XX:+CMSClassUnloadingEnabled | If this is not enabled, CMS will not clean permanent space. |
| -XX:+ExplicitGCInvokes\Concurrent | This allows System.gc() to trigger concurrent collection instead of a full garbage collection cycle. |


### G1 垃圾收集器

G1（Garbage First）从 Java7 开始可用，并希望在未来逐渐替代 CMS。目前，在 Java8中，G1 是默认的垃圾收集器。G1 是并行、并发、分代收集、低停顿的垃圾收集器。

G1 将按区（region）将堆分割，每个 region 的大小通常为 2048 字节。每个 region 可能是新生代或老年代（新生代又被分为 eden 和 survivor region）。这允许 GC 不需要一次对整个堆进行垃圾回收，而是可以增量地进行。这意味着一次只在一部分 region 上进行垃圾回收。

![Memory-regions-marked-G1.png](/images/2018-10-05/Memory-regions-marked-G1.png)

G1 追踪每个 region 中包含的存活数据的数量。追踪的结果被用于觉得那些 region 中 garbage 最多，最多的被最先回收。

和其他算法一样，整理的操作也会暂停程序的运行。但是我们可以配置暂停时间。G1将尽可能的满足配置。

