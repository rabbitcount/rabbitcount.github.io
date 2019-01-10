---
title: 垃圾回收算法
date: 2018-03-11 19:07:31
tags:
- gc
---

# 前人的肩膀
- [ref 常用垃圾回收算法](https://www.cnblogs.com/guozhenqiang/p/5621665.html)
- ref 《垃圾回收的算法与实现》 by 中村成洋
- [ref JVM中CMS收集器（示例图片靠谱）](https://segmentfault.com/a/1190000007765676)

# key word

对象、指针、mutator、堆、活动对象/非活动对象、分配、分块、根（GC root）

## mutator
> from： 《垃圾回收的算法与实现》1.3 mutator

mutator 是 Edsger Dijkstra琢磨出来的词，有“改变某物”的意思。
mutator 的实际操作有一下两种：
- 生成对象
- 更新指针

mutator 在进行这些操作时，会同时为应用程序的用户进行一些处理（数值计算、浏览网页、编辑文章等）。对着这些处理的逐步推进，对象间的引用关系也会“改变”，而负责回收这些垃圾的机制就是GC。

## 评价标准
- 吞吐量（throughput）
- 最大暂停时间
- 堆使用效率
- 访问的局部性

# 垃圾回收算法

## Reference Counting 引用计数法

这个方法是最经典点的一种方法。具体是对于对象设置一个引用计数器，每增加一个变量对它的引用，引用计数器就会加1，每减少一个变量的引用，引用计数器就会减1，只有当对象的引用计数器变成0时，该对象才会被回收。

RC仅可远观，因为，坑很多：
- 采用这种方法后，每次在增加变量引用和减少引用时都要进行加法或减法操作，如果频繁操作对象的话，在一定程度上增加的系统的消耗。
- 这种方法无法处理循环引用的情况（假设有两个对象 A和B，A中引用了B对象，并且B中也引用了A对象，那么这时两个对象的引用计数器都不为0，但是由于存在相互引用导致无法垃圾回收A和 B，导致内存泄漏）

### RC 优点
- 可立刻回收垃圾：直接可以通过计数判断
- 最大暂停时间短
- 没有必要沿指针查找

### RC 缺点
- 计数器值的增减处理繁重
- 计数器需要占用额外的位
用于已用计数的计数器最大必须能数完堆中的所有对象的引用数。比如用32位机器，就有可能要让2的32次方个对象同时引用一个对象。考虑到这种情况，就有必要确保各对象的计数器有32位大小。也就是说，对于所有对象，必须留有32位的空间。会使空间利用率大大降低
- 实现繁琐复杂
- 循环引用无法回收

## Mark Sweep 标记清除法

这个方法是将垃圾回收分成了两个阶段：标记阶段和清除阶段。
- **标记阶段**：通过跟对象，标记所有从根节点开始可达的对象，那么未标记的对象就是未被引用的垃圾对象。
- **清除阶段**：清除掉所以的未被标记的对象。

### MS 优点

- 算法简单，实现容易；
  - vs 引用计数中切实管理计数器的增减复杂，实现也很困难
  - 由于标记清除实现简单，所以更容易与其他算法结合使用，比如JVM中与Mark Compact混合使用的CMS
- 不移动对象（因此也可以与保守式GC兼容）

### MS 缺点

> 碎片化（fragmentation）

垃圾回收后可能存在大量的磁盘碎片，准确的说是内存碎片。因为对象所占用的地址空间是固定的。

## Mark Compact 标记压缩清除法（Java中老年代采用）

在 Mark Sweep 的基础上做了一个改进，可以说这个算法分为三个阶段：
- 标记阶段（不变）
- 压缩阶段（新增）
- 清除阶段（不变）

将这些标记过的对象集中放到一起，确定开始和结束地址，比如全部放到开始处，这样再去清除，将不会产生磁盘碎片。但是我们也要注意到几个问题，压缩阶段占用了系统的消耗，并且如果标记对象过多的话，损耗可能会很大，在标记对象相对较少的时候，效率较高。

### MC优点

- 可有效利用堆

### MC缺点

- 压缩话费计算成本

## Copying GC复制算法（Java中新生代采用）。

> 分块，将存活对象复制到另一个块中

核心思想是将内存空间分成两块，同一时刻只使用其中的一块，在垃圾回收时将正在使用的内存中的存活的对象复制到未使用的内存中，然后清除正在使用的内存块中所有的对象，然后把未使用的内存块变成正在使用的内存块，把原来使用的内存块变成未使用的内存块。很明显如果存活对象较多的话，算法效率会比较差，并且这样会使内存的空间折半，但是这种方法也不会产生内存碎片。

### copy优点

- 优秀的吞吐量：仅需搜索并复制活动对象；
- 可实现高速分配：分块是连续的内存空间（不使用空闲链表），因此仅需调查分块的大小，只要分块大小不小于所申请的大小，仅需要移动$free指针可以进行分配；
- 不会发生碎片化：复制后的存活对象是内存连续的；
- 与缓存兼容：
在GC复制算法中有引用关系的对象会被安排在堆里离彼此较近的位置。这种情况有一个优点，就是mutator执行速度极快

### copy缺点

- 堆使用效率低下：堆二分，一次只能利用一半
- 不兼容保守式GC算法：保守式GC不移动对象
- 递归调用函数：在复制某个对象时，需要递归复制它的子对象，在每次复制的时候都要调用函数，会带来不容忽视的额外负担

## 分代法（Java堆采用）。

主要思想是根据对象的生命周期长短特点将其进行分块，根据每块内存区间的特点，使用不同的回收算法，从而提高垃圾回收的效率。
比如Java虚拟机中的堆就采用了这种方法分成了新生代和老年代。然后对于不同的代采用不同的垃圾回收算法。新生代使用了复制算法，老年代使用了标记压缩清除算法。

老年代：old 或称 tenuring
minGC、majorGC

## 分区算法。

这种方法将整个空间划分成连续的不同的小区间，每个区间都独立使用，独立回收，好处是可以控制一次回收多少个小区间。

# VM 

分代式GC里，年老代常用mark-sweep；或者是mark-sweep/mark-compact的混合方式，一般情况下用mark-sweep，统计估算碎片量达到一定程度时用mark-compact。这是因为传统上大家认为年老代的对象可能会长时间存活且存活率高，或者是比较大，这样拷贝起来不划算，还不如采用就地收集的方式。[Mark-sweep](http://www.memorymanagement.org/glossary/m.html#mark-sweep)、[mark-compact](http://www.memorymanagement.org/glossary/m.html#mark-compact)、[copying](http://www.memorymanagement.org/glossary/c.html#copying.garbage.collection)这三种基本算法里，只有mark-sweep是不移动对象（也就是不用拷贝）的，所以选用mark-sweep。


简要对比三种基本算法：

|  mark-sweep |   mark-compact |   copying             
------- | ------------ | -------------- | ---------------------
 速度    |   中等         |   最慢           |   最快                  
 空间开销  |   少（但会堆积碎片）  |   少（不堆积碎片）     |   通常需要活对象的2倍大小（不堆积碎片） 
 移动对象？ |   否          |   是            |   是                     

关于时间开销：

- mark-sweep：mark阶段与活对象的数量成正比，sweep阶段与整堆大小成正比
- mark-compact：mark阶段与活对象的数量成正比，compact阶段与活对象的大小成正比
- copying：与活对象大小成正比

如果把mark、sweep、compact、copying这几种动作的耗时放在一起看，大致有这样的关系：
> compaction >= copying > marking > sweeping

还有 
> marking + sweeping > copying

（虽然compactiont与copying都涉及移动对象，但取决于具体算法，compact可能要先计算一次对象的目标地址，然后修正指针，然后再移动对象；copying则可以把这几件事情合为一体来做，所以可以快一些。

另外还需要留意GC带来的开销不能只看collector的耗时，还得看allocator一侧的。如果能保证内存没碎片，分配就可以用pointer bumping方式，只有挪一个指针就完成了分配，非常快；而如果内存有碎片就得用freelist之类的方式管理，分配速度通常会慢一些。）


在分代式假设中，年轻代中的对象在minor GC时的存活率应该很低，这样用copying算法就是最合算的，因为其时间开销与活对象的大小成正比，如果没多少活对象，它就非常快；而且young gen本身应该比较小，就算需要2倍空间也只会浪费不太多的空间。

而年老代被GC时对象存活率可能会很高，而且假定可用剩余空间不太多，这样copying算法就不太合适，于是更可能选用另两种算法，特别是不用移动对象的mark-sweep算法。


不过HotSpot VM中除了CMS之外的其它收集器都是会移动对象的，也就是要么是copying、要么是mark-compact的变种。

# Hotspot

> [参考 JVM cms](http://hllvm.group.iteye.com/group/topic/38223#post-248757)

HotSpot VM的serial GC（UseSerialGC）、parallel GC（UseParallelGC）中，只有full GC会收集年老代（实际上收集整个GC堆，包括年老代在内）。它用的算法是[mark-compact](http://www.oracle.com/technetwork/java/whitepaper-135217.html)（"Mark-Compact Old Object Collector"那一节），具体来说是典型的单线程（串行）的[LISP 2算法](http://www.memorymanagement.org/glossary/m.html#mark-compact)。虽然在HotSpot VM的源码里，这个full GC的实现类叫做[MarkSweep](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/tip/src/share/vm/gc_implementation/shared/markSweep.hpp)，而许多资料上都把它称为[mark-sweep-compact](http://www.oracle.com/technetwork/java/javase/memorymanagement-whitepaper-150215.pdf)，但实际上它是典型的mark-compact而不是mark-sweep，请留意不要弄混了。出现这种情况是历史原因，十几二十年前GC的术语还没固定到几个公认的用法时mark-sweep-compact和mark-compact说的是一回事。


我不太清楚当初HotSpot VM为何选择先以mark-compact算法来实现full GC，而不像后来微软的[CLR](http://msdn.microsoft.com/en-us/library/8bs2ecf4.aspx)那样先选择使用mark-sweep为基本算法来实现[Gen 2 GC](http://msdn.microsoft.com/en-us/library/ee787088.aspx)。但其背后的真相未必很复杂：

HotSpot VM的前身是[Strongtalk VM](http://strongtalk.org/)，它的full GC也是mark-compact算法的，虽说具体算法跟HotSpot VM的不同，是一种threaded compaction算法。这种算法比较省空间，但限制也挺多，实现起来比较绕弯，所以后来出现的HotSpot才改用了更简单直观的LISP 2算法吧，而这个决定又进一步在V8上得到体现。

而Strongtalk VM的前身是[Self VM](http://selflanguage.org/)，同样使用mark-compact算法来实现full GC。可以看到mark-compact是这一系列VM一脉相承的，一直延续到更加新的[Google V8](https://code.google.com/p/v8/)也是如此。或许当初规划HotSpot VM时也没想那么多就先继承下了其前身的特点。


如果硬要猜为什么，那一个合理的推断是：如果不能整理碎片，长时间运行的程序终究会遭遇内存碎片化，导致内存空间的浪费和内存分配速度下降的问题；要解决这个问题就得要能整理内存。如果决定要根治碎片化问题，那么可以直接选用mark-compact，或者是主要用mark-sweep外加用mark-compact来备份。显然直接选用mark-compact实现起来更简单些。所以就选它了。

（而CLR就选择了不根治碎片化问题。所有可能出问题的地方都最终会出问题，于是现在就有很多.NET程序受碎片化的困扰）


后来HotSpot VM有了[parallel old GC](http://docs.oracle.com/javase/6/docs/technotes/guides/vm/par-compaction-6.html)（UseParallelOldGC），这个用的是多线程并行版的mark-compact算法。这个算法具体应该叫什么名字我说不上来，因为并没有专门的论文去描述它，而大家有许多不同的办法去并行化LISP 2之类的经典mark-compact算法，各自取舍的细节都不同。无论如何，这里要关注的只是它用的也是mark-compact而不是mark-sweep算法。


================================================================


那CMS为啥选用mark-sweep为基本算法将其并发化，而不跟HotSpot VM的其它GC一样用会移动对象的算法呢？


一个不算原因的原因是：当时设计和实现CMS是在Sun的另外一款JVM，[Exact VM（EVM）](http://www.cs.rit.edu/~swm/gc/smli_tr-98-67.pdf%E2%80%8E)上做的。后来EVM项目跟HotSpot VM竞争落败，CMS才从EVM移植到HotSpot VM来。因此它没有HotSpot VM的初始血缘。 <- 真的算不上原因（逃


真正的原因请参考CMS的原始论文：[A Generational Mostly-concurrent Garbage Collector](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.22.8915)（可恶，Oracle Labs的链接挂了。用CiteSeerX的链接吧）


把GC之外的代码（主要是应用程序的逻辑）叫做mutator，把GC的代码叫做collector。两者之间需要保持同步，这样才可以保证两者所观察到的对象图是一致的。


如果有一个串行、不并发、不分代、不增量式的collector，那么它在工作的时候总是能观察到整个对象图。因而它跟mutator之间的同步方式非常简单：mutator一侧不用做任何特殊的事情，只要在需要GC时同步调用collector即可，就跟普通函数调用一样。


如果有一个分代式的，或者增量式的collector，那它在工作的时候就只会观察到整个对象图的一部分；它观察不到的部分就有可能与mutator产生不一致，于是需要mutator配合：它与mutator之间需要额外的同步。Mutator在改变对象图中的引用关系时必须执行一些额外代码，让collector记录下这些变化。有两种做法，一种是[write barrier](http://www.ravenbrook.com/project/mps/master/manual/html/glossary/w.html#term-write-barrier)，一种是[read barrier](http://www.ravenbrook.com/project/mps/master/manual/html/glossary/r.html#term-read-barrier)。

Write barrier就是当改写一个引用时：

Java代码 

<embed wmode="transparent" src="/javascripts/syntaxhighlighter/clipboard_new.swf" width="14" height="15" flashvars="clipboard=a.x%20%3D%20b" quality="high" allowscriptaccess="always" type="application/x-shockwave-flash" pluginspage="http://www.macromedia.com/go/getflashplayer">[![收藏代码](http://hllvm.group.iteye.com/images/icon_star.png)![](http://hllvm.group.iteye.com/images/spinner.gif)](javascript:void() "收藏这段代码")

1.  a.x = b  

<pre name="code" class="java" codeable_id="post-248757" codeable_type="GroupPost" source_url="http://hllvm.group.iteye.com/group/topic/38223#post-248757" pre_index="0" title="并发垃圾收集器（CMS）为什么没有采用标记-整理算法来实现？" style="display: none;">
a.x = b
</pre>  
插入一块额外的代码，变成：

C代码 

<embed wmode="transparent" src="/javascripts/syntaxhighlighter/clipboard_new.swf" width="14" height="15" flashvars="clipboard=write_barrier(a%2C%20%26(a-%3Ex)%2C%20b)%3B%0Aa-%3Ex%20%3D%20b%3B" quality="high" allowscriptaccess="always" type="application/x-shockwave-flash" pluginspage="http://www.macromedia.com/go/getflashplayer">[![收藏代码](http://hllvm.group.iteye.com/images/icon_star.png)![](http://hllvm.group.iteye.com/images/spinner.gif)](javascript:void() "收藏这段代码")

1.  write_barrier(a, &(a->x), b);  
2.  a->x = b;  

<pre name="code" class="c" codeable_id="post-248757" codeable_type="GroupPost" source_url="http://hllvm.group.iteye.com/group/topic/38223#post-248757" pre_index="1" title="并发垃圾收集器（CMS）为什么没有采用标记-整理算法来实现？" style="display: none;">
write_barrier(a, &(a->x), b);
a->x = b;
</pre>  
Read barrier就是当读取一个引用时：

Java代码 

<embed wmode="transparent" src="/javascripts/syntaxhighlighter/clipboard_new.swf" width="14" height="15" flashvars="clipboard=b%20%3D%20a.x" quality="high" allowscriptaccess="always" type="application/x-shockwave-flash" pluginspage="http://www.macromedia.com/go/getflashplayer">[![收藏代码](http://hllvm.group.iteye.com/images/icon_star.png)![](http://hllvm.group.iteye.com/images/spinner.gif)](javascript:void() "收藏这段代码")

1.  b = a.x  

<pre name="code" class="java" codeable_id="post-248757" codeable_type="GroupPost" source_url="http://hllvm.group.iteye.com/group/topic/38223#post-248757" pre_index="2" title="并发垃圾收集器（CMS）为什么没有采用标记-整理算法来实现？" style="display: none;">
b = a.x
</pre>  
插入一块额外的代码，变成：

C代码 

<embed wmode="transparent" src="/javascripts/syntaxhighlighter/clipboard_new.swf" width="14" height="15" flashvars="clipboard=read_barrier(%26(a-%3Ex))%3B%0Ab%20%3D%20a-%3Ex%3B" quality="high" allowscriptaccess="always" type="application/x-shockwave-flash" pluginspage="http://www.macromedia.com/go/getflashplayer">[![收藏代码](http://hllvm.group.iteye.com/images/icon_star.png)![](http://hllvm.group.iteye.com/images/spinner.gif)](javascript:void() "收藏这段代码")

1.  read_barrier(&(a->x));  
2.  b = a->x;  

<pre name="code" class="c" codeable_id="post-248757" codeable_type="GroupPost" source_url="http://hllvm.group.iteye.com/group/topic/38223#post-248757" pre_index="3" title="并发垃圾收集器（CMS）为什么没有采用标记-整理算法来实现？" style="display: none;">
read_barrier(&(a->x));
b = a->x;
</pre>  

通常一个程序里对引用的读远比对引用的写要更频繁，所以通常认为read barrier的开销远大于write barrier，所以很少有GC使用read barrier。

如果只用write barrier，那么“移动对象”这个动作就必须要完全暂停mutator，让collector把对象都移动好，然后把指针都修正好，接下来才可以恢复mutator的执行。也就是说collector“移动对象”这个动作无法与mutator并发进行。


如果用到了read barrier（虽少见但不是不存在，例如[Azul C4 Collector](http://www.azulsystems.com/technology/c4-garbage-collector)），那移动对象就可以单个单个的进行，而且不需要立即修正所有的指针，所以可以看作整个过程collector都与mutator是并发的。


CMS没有使用read barrier，只用了write barrier。这样，如果它要选用mark-compact为基本算法的话，就只有mark阶段可以并发执行（其中root scanning阶段仍然需要暂停mutator，这是initial marking；后面的concurrent marking才可以跟mutator并发执行），然后整个compact阶段都要暂停mutator。回想最初提到的：compact阶段的时间开销与活对象的大小成正比，这对年老代来说就不划算了。

于是选用mark-sweep为基本算法就是很合理的选择：mark与sweep阶段都可以与mutator并发执行。Sweep阶段由于不移动对象所以不用修正指针，所以不用暂停mutator。


（题外话：但现实中我们仍然可以看到以mark-compact为基础算法的增量式/并发式年老代GC。例如Google V8里的年老代GC就可以把marking阶段拆分为非并发的initial marking和增量式的[incremental marking](https://code.google.com/p/v8/source/browse/trunk/src/incremental-marking.h)；但真正比较耗时的compact阶段仍然需要完全暂停mutator。它要降低暂停时间就只能想办法在年老代内进一步选择其中一部分来做compaction，而不是整个年老代一口气做compaction。这在V8里也已经有实现，叫做incremental compaction。再继续朝这方向发展的话最终会变成region-based collector，那就跟G1类似了。）


那碎片堆积起来了怎么办呢？HotSpot VM里CMS只负责并发收集年老代（而不是整个GC堆）。如果并发收集所回收到的空间赶不上分配的需求，就会回退到使用serial GC的mark-compact算法做full GC。也就是mark-sweep为主，mark-compact为备份的经典配置。但这种配置方式也埋下了隐患：使用CMS时必须非常小心的调优，尽量推迟由碎片化引致的full GC的发生。一旦发生full GC，暂停时间可能又会很长，这样原本为低延迟而选择CMS的优势就没了。


所以新的Garbage-First（G1）GC就回到了以copying为基础的算法上，把整个GC堆划分为许多小区域（region），通过每次GC只选择收集很少量region来控制移动对象带来的暂停时间。这样既能实现低延迟也不会受碎片化的影响。

（注意：G1虽然有concurrent global marking，但那是可选的，真正带来暂停时间的工作仍然是基于copying算法而不是mark-compact的）
