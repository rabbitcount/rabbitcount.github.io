---
title: jvm_garbage_collect
date: 2017-12-20 23:58:49
tags:
---

Java和C++之间有一堵由内存分配和垃圾回收技术所围成的高墙，在里面的人想出来，不在里面的人想进去。C++程序员必须承担每一个对象生命开始到终结的责任，而Java程序员无须为每一个new 出来的对象执行 delete/free 操作，不容易出现内存泄漏和内存溢出问题。

# 1. JVM内存模型

## 1.1. 内存结构

根据《Java虚拟机规范（Java SE 7版本）》规定，JVM包括下面几个运行时的内存区域：
<div align=center>
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_memory_structure.gif)
</div>

共分为下面五个部分：
- 程序计数器：当前线程所执行的字节码的行号指示器；
- 方法区：方法区用于存储已经被虚拟机加载的类信息、final常量、静态变量、编译器即时编译后的代码等数据；
- 本地方法栈：执行 Native 方法时的栈存储区域；
- JVM 虚拟机栈：Java方法执行时的栈帧，存储局部变量表、操作数栈、动态链接、方法接口 等信息；
- 堆区：所有的对象实例以及数组。

下图是 Hot-Spot 虚拟机的内存划分：

<div align=center>
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/hot_spot_jmm.gif)
</div>

**注意**：
1. HotSpot虚拟机把本地方法栈和虚拟机栈合二为一；
2. 方法区 ≠ 永久代，后者是HotSpot虚拟机中的特定概念，是分代算法的延伸。
3. **JAVA进程内存 = JVM进程内存 + Heap内存 + 永久代内存 + 本地方法栈内存 + 线程栈内存 + 堆外内存 + Socket 缓冲区内存**。

方法区在JDK各个版本中的演进？
- 在 Java 6 中，方法区中包含的数据，除了 JIT 编译生成的代码存放在 Native memory 的 CodeCach 区域，其他都存放在永久代；
- 在 Java 7 中，符号引用迁移至系统内存(Native Memory)，字符串字面量迁移至Java堆(Java Heap)，二者均属于常量池的内容；
- 在 Java 8 中，永久代被彻底移除，取而代之的是另一块与堆不相连的本地内存——元空间（Metaspace）,‑XX:MaxPermSize 参数失去了意义，取而代之的是-XX:MaxMetaspaceSize。

## 1.2. 内存溢出
程序计数器是JMM中唯一不会发生 OOM 的地方。OOM的种类、根源及解决方法：

- 堆内存溢出（关键字：OutOfMemoryError，Java heap space）
	产生原因：堆中无法存放新的对象，同时垃圾回收机制也无法回收对象；
	解决方法：通过Dump内存，明确是内存溢出还是内存泄漏，然后确定方案；
- 栈内存溢出（关键字：StackOverflowError 或 StackOutOfMemoryError）
	产生原因：栈深度大于虚拟机锁允许的最大深度 或 拓展栈时无法申请到足够的内存空间；
	解决方法：配置 -Xss 增大栈内存容量，但这会减少工作线程数，因此需要权衡。
- 方法区溢出（关键字：OutOfMemoryError，PermGen space）
	产生原因：常量池溢出或动态生成了大量的Class而未卸载；
	解决方法：JDK6 中谨慎使用intern()，卸载不使用的类数据。

# 2. 垃圾回收策略
## 2.1. 基本概念
### 2.1.1. 关键术语
- **并行（Parallel）**：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
- **并发（Concurrent）**：指用户线程与垃圾收集线程同时执行（但不一定是并行的，可能会交替执行），用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。
- **STW（Stop The World）**：在执行垃圾收集算法时，除了垃圾收集线程外，Java应用程序的其他线程都被挂起的现象（Native 代码可以执行）。
- **引用（Reference）**：从JDK 1.2版本开始，把对象的引用分为4种级别，从而使程序能更加灵活地控制对象的生命周期。这4种级别由高到低依次为：强引用、软引用、弱引用和虚引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它，其他的引用类型都会被回收。

### 2.1.2. 垃圾的定义
GC 把程序不用的内存空间视为垃圾, 是管理堆中已分配对象的机制， GC 要做的有两件事：
- 查找内存中不再使用的对象
- 释放这些对象占用的内存

怎么确保内存空间已经不被使用？
- 引用计数法
- 可达性分析

#### 2.1.2.1. 引用计数法
给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值加1；当引用失效时，计数器减1；任何时刻计数器都为0的对象就是不可能再被使用的。下图为 Python 中通过引用计数法定义的核心结构体。

```
typedef struct_object {
    int ob_refcnt;
    struct_typeobject *ob_type;
} PyObject;
```

引用计数算法的实现简单，判断效率也很高，在大部分情况下它都是一个不错的算法。但是Java语言中没有选用引用计数算法来管理内存，其中最主要的一个原因是它很难解决对象之间相互循环引用的问题。Python 就通过通过标记-清除和分代收集两种机制补充引用计数的不足。

#### 2.1.2.2. 可达性分析
在主流的商用程序语言中(Java和C#)，都是使用可达性分析算法判断对象是否存活的。这个算法的基本思路是通过一系列名为 GC Roots 的对象作为起始点，从这些根节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到 GC Roots 没有任何引用链相连时，则证明此对象是不可用的，就可以纳入可回收的范围。

在 Java 语言里，可作为 GC Roots 对象的包括如下几种：
- 虚拟机栈(栈桢中的本地变量表)中的引用的对象；
- 方法区中的类静态属性引用的对象；
- 方法区中的常量引用的对象；
- 本地方法栈中 JNI 的引用的对象。

### 2.1.3. 垃圾回收的内存区域
垃圾回收主要是在回收堆（Heap）内存，下面将详细叙述。

对于属于堆外内存（Non-Heap）的永久代，Java虚拟机规范中没有规定要回收，但是永久代也是需要回收的，不过频率较低，主要做的工作是常量池回收和类型卸载。常量池回收比较简单，通过判断字面量是否有对象引用即可；对于类型卸载，可是通过以下三条规则判断一个类是否是无用的类：
- 该类所有的实例都已经被回收，也就是java堆中不存在该类的任何实例。
- 加载该类的ClassLoader已经被回收。
- 该类对应的java.lang.Class对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法。

`使用堆外内存时的注意点?`

堆外内存就是把内存对象分配在Java虚拟机的堆以外的内存，包括JVM本身在运行过程中分配的内存，CodeCache，JNI 里分配的内存、DirectByteBuffer 分配的内存等等，这些内存直接受操作系统管理（而不是 JVM ），如 Netty 中使用 java.nio.DirectByteBuffer 创建的内存。
- 优点：结果就是能够在一定程度上减少垃圾回收对应用程序造成的影响，执行 Flush 到远端的操作时，也节省了**复制到直接内存**这部分的时间。
- 缺点：JVM 不会直接管理这些堆外内存，存在 OOM 的风险。可以在 JVM 启动参数里加上 -XX:MaxDirectMemorySize，对可申请的堆外内存大小进行限制（注意：这个参数的配置直接会影响 Full GC 的频率）。
**不直接管理的含义**：JDK 中使用 DirectByteBuffer 对象来表示堆外内存，DirectByteBuffer 对象里持有 Cleaner 对象，后者唯一保存了堆外内存的数据（开始地址、大小和容量。在创建完成后的下一次 FGC 时，Cleaner对象会进行堆外内存的回收。

申请堆外内存时，如果申请的内存大小超出限制，则会调用 System.gc() 以期望触发垃圾回收，并将当前线程 sleep 100毫秒，之后再尝试申请，如果此时申请失败，则抛出 OOM 报错，这种场景多出现在禁用了显式GC（-XX:+DisableExplicitGC）的环境下。

### 2.1.4. 垃圾回收的分类
## 2.2. 垃圾回收算法
### 2.2.1. 垃圾回收算法性能
- 吞吐量（Throughput）：应用程序线程用时 / 程序总用时的比例。吞吐量越高，则算法越好。
- 最大暂停时间（pause times）：因执行 GC 而暂停应用程序线程线程的最长时间。暂停时间越短，则算法越好。
- 堆使用效率：堆使用效率和吞吐量，以及最大暂停时间不可兼得。简单地说就是：可用的堆越大，GC 运行越快；相反，越想有效地利用有限的堆，GC 花费的时间就越长。
- 访问的局部性：具有引用关系的对象之间通常很可能存在连续访问的情况。这在多数程序中都很常见，称为“访问的局部性”。考虑到访问的局部性，把具有引用关系的对象安排在堆中较近的位置，就能提高在缓存中读取到想利用的数据的概率，令mutator 高速运行。

高吞吐量和低暂停时间是竞争关系，为了获得最大吞吐量，JVM 必须尽可能少地运行 GC，只有在迫不得已的情况下(比如新生代或者老年代已经满了）才运行GC。但是，推迟运行 GC 的结果是，每次运行GC时需要做的事情会很多，比如有更多的对象积累在堆上等待回收，每次的GC时间会很高，由此引起的平均和最大暂停时间也会很高，这就要求 GC 不能推迟运行的时机。

### 2.2.2. 常用垃圾回收算法
推荐阅读《[垃圾回收的算法与实现](https://book.douban.com/subject/26821357/)》这一本书，这里有我的读书笔记：垃圾回收的算法与实现。

- **标记-清除算法（Mark-Sweep）**：GC 标记- 清除算法由标记阶段和清除阶段构成。标记阶段是把所有活动对象都做上标记的阶段。清除阶段是把那些没有标记的对象，也就是非活动对象回收的阶段。通过这两个阶段，就可以令不能利用的内存空间重新得到利用。
- **标记-压缩算法（Mark-Compact）**：GC 标记-压缩算法由标记阶段和压缩阶段构成。标记阶段和 GC 标记-清除算法段完全一样，压缩阶段通过数次搜索堆来重新装填活动对象。压缩阶段并不会改变对象的排列顺序，只是缩小了它们之间的空隙，把它们聚集到了堆的一端。
- **复制算法（Copying）**：GC 复制算法是将可用内存划分为两块区域(通常为相等大小)：From、to，当有新的活动对象加入空闲内存，利用 From 空间进行分配，当From 空间被完全占满时，GC 会将活动对象全部复制到 To 空间。当复制完成后，该算法会把From 空间和 To 空间互换，GC 也就结束了。From 空间和 To 空间大小必须一致。这是为了保证能把 From 空间中的所有活动对象都收纳到 To 空间里。
- **分代算法(Generational GC)**：根据对象的不同生命周期分别管理，HotSpot JVM 中将对象分为我们熟悉的新生代、老年代和永久代分别管理。这样做的好处就是可以根据不同类型对象进行不同策略的管理，例如新生代中对象更新速度快，就会使用效率较高的复制算法。老年代中内存空间相对分配较大，而且时效性不如新生代强，就会常常使用Mark-Sweep-Compact(标记-清除-压缩)算法。

比较前三种算法：
<div align=center>
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_garbage_collection.png)
</div>

注意：Mark-Compact 与 Copying 都涉及移动对象，但取决于具体算法， Mark-Compact 可能要先计算一次对象的目标地址，然后修正指针，然后再移动对象；Copying 则可以把这几件事情合为一体来做，所以可以快一些。

## 2.3. 垃圾回收器
### 2.3.1. 垃圾回收器分类
垃圾回收算法是垃圾回收的方法论，垃圾回收器是垃圾回收算法的具体实践。Java虚拟机规范中对垃圾回收器该如何实现并没有任何规定，因此不同的厂商、不同的版本虚拟机提供的垃圾回收器都有很大的差别。下图是JDK 7 Update 4中的垃圾回收器。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_garbage_collect_category.png)

组合起来有以下几种:
- **Serial + Serial Old (+XX:+UseSerialGC)**: GC 线程在做事情时, 其他所有的用户线程都必须停止 (即 STW, stop the world)；
- **Serial + CMS**: 一般不会这样配合使用；
- **ParNew + CMS (+XX:+UseConcMarkSweepGC)**: 新生代的 GC 使用 ParNew, 有多个 GC 线程同时进行 Young GC (主要是在多核的环境用多线程效果会好); 而老生代使用 CMS；
- **ParNew + Serial Old (+XX:+UseParNewGC)**: 新生代用 ParNew 的时候, 也可以选择老生代不用 CMS, 而用 Serial Old, 这个组合也不太常用；
- **Parallel Scavenge + Serial Old (+XX:+UseParallelGC)**: Parallel Scavenge 收集器的目的是达到一个可控制的吞吐率 (适用于各种计算任务); 这个组合中老生代仍旧使用 Serial Old；
- **Parallel Scavenge + Parallel Old (+XX:+UseParallelOldGC)**: 新生代使用 Parallel Scavenge, 而 Parallel Old 是老年代版本的 Parallel Scavenge；
- **G1 (-XX:+UseG1GC)**：新生代和老年代都使用 G1 垃圾回收器。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_garbage_collector.gif)

垃圾收集器搭配注意事项:
- CMS 只能配 Serial 或 ParNew；
Pa- rallel Scavenge 只能配 Serial Old 或 Parallel Old；
- Serial 不能配 Parallel Old；
- UseParallelGC vs. UseParallelOldGC, 如果没有调好配置, UseParallelOldGC 有可能比 UseParallelGC 的性能还要差 (参考)。

### 2.3.2. 垃圾回收器的选择
我应该选用哪一种垃圾回收器？
1. 客户端程序: 一般使用 -XX:+UseSerialGC (Serial + Serial Old). 特别注意, 当一台机器上起多个 JVM, 每个 JVM 也可以采用这种 GC 组合；
2. 吞吐率优先的服务端程序（eg. 计算密集型）: -XX:+UseParallelGC 或者 -XX:+UseParallelOldGC；
3. 响应时间优先的服务端程序: -XX:+UseConcMarkSweepGC；
4. 响应时间优先同时也要兼顾吞吐率的服务端程序：-XX:+UseG1GC。

### 2.3.3. 新生代垃圾回收器
新生代 GC（Young GC），主要通过复制算法来实现，利用了复制算法时间开销较低的特点。

需要注意，新生代垃圾回收大多数是 STW 的， 停顿时间与 GC 后存活的对象成正比。

### 2.3.4. 老年代垃圾回收：CMS
CMS 全称为 Concurrent Mark Sweep，是现在非常主流的一款老年代的垃圾回收器，CMS 是以牺牲吞吐量为代价来获得最短回收停顿时间的垃圾回收器。对于要求服务器响应速度的应用上，这种垃圾回收器非常适合。在启动JVM参数加上 -XX:+UseConcMarkSweepGC ，这个参数表示对于老年代的回收采用 CMS。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_cms_structure.png)

CMS 采用的基础算法是：标记—清除 算法。IBM 的专门研究表明，新生代中的对象98%是朝生夕死的，所以并不需要按照1∶1的比例来划分内存空间，而是将内存分为一块较大的 Eden 空间和两块较小的 Survivor 空间，每次使用 Eden 和其中的一块 Survivor。当回收时，将 Eden 和 Survivor 中还存活着的对象一次性地拷贝到另外一块 Survivor 空间上，最后清理掉 Eden 和刚才用过的 Survivor 的空间。

#### 2.3.4.1. CMS 的优缺点
优点： 并发收集，低停顿时间。
缺点：
1. 会产生空间碎片。CMS 垃圾回收器采用的基础算法是 Mark-Sweep，没有内存整理的过程，所以经过 CMS 收集的堆会产生空间碎片。
2. 对CPU资源非常敏感。为了让应用程序不停顿，CMS 线程需要和应用程序线程并发执行，这样就需要有更多的 CPU，同时会使得总吞吐量降低。
3. 需要更大的堆空间。因为 CMS 在标记阶段应用程序的线程还是在执行的，那么就会有堆空间继续分配的情况，为了保证在 CMS 回收完堆之前还有空间分配给正在运行的应用程序，必须预留一部分空间。

#### 2.3.4.2. CMS 执行过程
CMS 是老年代垃圾回收算法，老年代回收时，本身不会先进行 Minor GC。因为老年代很多对象都会引用到新生代的对象，我们可以通过配置先进行一次Minor GC，再执行老年代 GC，提高老年代 GC 的速度。比如老年代使用CMS时，设置 CMSScavengeBeforeRemark优化，让CMS remark之前先进行一次Minor GC。
CMS 的回收过程主要分为下面的几个步骤：
- 初始标记(STW initial mark)
- 并发标记(Concurrent marking)
- 并发预清理(Concurrent pre-cleaning)
- 重新标记(STW remark)
- 并发清理(Concurrent sweeping)
- 并发重置(Concurrent reset)
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_cms_process.gif)

##### 2.3.4.2.1. 初始标记
初始标记也就是标记一下 GC roots 关联到的对象（并不是所有活动对象），这个过程会出现 STW。
注意：CMS 虽然是老年代算法，但也是需要扫描新生代区域的。

##### 2.3.4.2.2. 并发标记
并发标记就需要标记出 GC roots 关联到的对象 的引用对象有哪些。比如说 A -> B (A 引用 B，假设 A 是 GC Roots 关联到的对象)，那么这个阶段就是标记出 B 对象， A 对象会在初始标记中标记出来。

##### 2.3.4.2.3. 并发预清理
预清理也属于并发处理阶段。这个阶段主要并发查找在做并发标记阶段时从新生代晋升到老年代的对象或老年代新分配的对象(大对象直接进入老年代)或被用户线程更新的对象，来减少重新标记阶段的工作量。
如何处理并发阶段被修改了的对象？
- 场景：初始标记阶段的引用关系为：A -> B -> C，并发标记时引用关系由用户现场改成了 A -> C，即 B 不再引用 C。由于 C 在并发阶段无法被标记，就会被回收，这样是不允许的。该问题可以通过三色标记算法解决。
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_three_color_mark.gif)

三色标记法把 GC 中的对象划分成三种情况：
- 白色：还没有搜索过的对象（白色对象会被当成垃圾对象）
- 灰色：正在搜索的对象
- 黑色：搜索完成的对象（不会当成垃圾对象，不会被 GC）

在初始标记阶段，A 会被标记成灰色（证明现在正在搜索 A 相关的），然后搜索 A 的引用，把 B 变成了灰色，然后 A 就搜索完成了，A 就变成了黑色。
在并发标记阶段，如果用户线程不在引用 B 对象，而是变成了 A->C，此时线程会将 C 这个对象会设置为已标记（把 C 变成灰色），这个过程就称之为写入屏障（ Write Barrier）。伪代码描述如下：
```
write_barrier(obj,field,newobj){
    if(newobj.mark == FALSE){
        newobj.mark = TRUE
        push(newobj,$mark_stack)
    }
    *field = newobj
}
```
 并发预清理阶段可能会出现 Young GC（是否出现 Young GC 由 CMSScheduleRemarkEdenSizeThreshold、CMSScheduleRemarkEdenPenetration、CMSMaxAbortablePrecleanTime 这个三个 GC 参数来保证）。
出现老年代引用新生代的对象，GC 怎么处理？
JVM采用了 Card Marking(卡片标记)的方法，避免了在做Minor GC时需要对整个老年代扫描。具体的方法：将老年代按照一定大小分片，每一片对应 Cards中的一项，如果老年代老年代的对象发生了修改，或者老年代对象指向了新生代对象，就把这个老年代对象所在的 Card 标记为脏 dirty。Young GC 时，dirty card 加入待扫描的 GC Roots 范围，避免扫描整个老年代。
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_card_table.png)


##### 2.3.4.2.4. 重新标记
重新标记是干什么的呢？就是由于在并发标记和并发预清理这个阶段，用户线程和GC 线程并发，假如这个阶段用户线程产生了新的对象，总不能被 GC 掉吧。这个阶段就是为了让这些对象重新标记。
这个过程会出现 STW，所有用户线程会暂停工作，GC 线程重新扫描堆中的对象，进行可达性分析，标记活着的对象。

##### 2.3.4.2.5. 并发清理
这个阶段的目的就是移除那些不用的对象，回收他们占用的空间并且为将来使用。注意这个阶段会产生新的垃圾，新的垃圾在此次GC无法清除，只能等到下次清理。这些垃圾有个专业名词：浮动垃圾。

##### 2.3.4.2.6. 并发重置
这个阶段并发执行，重新设置 CMS 算法内部的数据结构，准备下一个 CMS 生命周期的使用。

#### 2.3.4.3. CMS 触发条件
注意分为下面两类：
- 如果应用主动请求 GC，直接触发；
- 是否设置 UseCMSInitiatingOccupancyOnly；
	- 没有设置 UseCMSInitiatingOccupancyOnly
		- 统计开启（stats.valid），统计的cms完成时间小于cms剩余空间被填满的时间，则触发；
		- 统计不可用，（第一次没有统计信息，!stats.valid)，年老代大于 _bootstrap_occupancy，则触发；
	- 设置 UseCMSInitiatingOccupancyOnly
		- 根据指定年老代的判断逻辑 should_concurrent_collect，true 则触发；
		- 根据增量模式收集是否失败，incremental_collection_will_fail，true 则触发；
		- 根据元数据区的判断逻辑 should_concurrent_collect，true 则触发；
		- 最后根据触发间隔(CMSTriggerInterval，默认为-1，所以一般不走这个逻辑);
			
**should_concurrent_collect的逻辑实现**
- 判断occupancy是否大于init_occupancy，大于则触发；
- 如果设置了UseCMSInitiatingOccupancyOnly，直接返回，不再继续后面逻辑；

#### 2.3.4.4. CMS 降级
当 CMS 进行并发操作时，如果剩余的内存已经无法满足用户线程（ 比如 执行CMS 的阈值为 90%堆内存，假设这个时候用户线程需要 20% 的内存）了，此时老年代垃圾回收器自动降级为 Serial Old，这个时候你会在 GC 日志里看到 Concurrent Mode Failure。串行回收时，会出现 STW，也就不存在垃圾持续增长的问题了。

#### 2.3.4.5. CMS 调优参数
这里介绍几个重要的调优参数，更详细的参数请参考 CMS 描述文档。
*-XX:CMSInitiatingOccupancyFraction=70*
- 该值代表老年代堆空间的使用率，默认值是92。比如，value=70 意味着第一次 CMS 垃圾收集会在老年代被占用 70% 时被触发，该数字为经验值。

*-XX:+UseCMSCompactAtFullCollection*
*-XX:CMSFullGCsBeforeCompaction=4*
- 上面两个参数表示执行4次不压缩的 Full GC 后，会执行一次内存压缩的过程，用来消除 CMS 引入的空间碎片。

*-XX:+CMSScavengeBeforeRemark*
- 使用上述参数，会在重新标记阶段前强制进行一次 Young GC。

*-XX：ConcGCThreads
- 定义并发 CMS 过程运行时的线程数。CMS 默认的回收线程数是( CPU 个数+3)/4，意思是当 CPU 大于4个时,保证回收线程占用至少25%的 CPU 资源，这样用户线程占用75%的 CPU。更多的线程会加快并发 CMS 过程，但其也会带来额外的同步开销。

一些简单的调优策略：
新生代优化:
- Young GC 频率高 -> 增大新生代
- Young GC 时间长 -> 减小新生代
老年代优化：
- 尽量避免内存整理，整理会 STW，可以通过优化新生代到老生代的提升率事项；
- 设置合理的 -XX:CMSInitiatingOccupancyFraction=n 值，过大或让 STW 时间变长，过小会影响吞吐率。

一个原则：尽量让 Young GC 回收大部分的垃圾。

### 2.3.5. 全局垃圾回收器：G1

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jvm/jvm_g1_region.gif)

Garbage First（G1）的设计初衷是，以更高的计算成本为代价最小化 STW 中断时间。通常来说，限制 GC 中断时间比最大化吞吐量更重要。对大部分用户而言，与面向吞吐量的收集器相比（如并行垃圾收集器），切换到中断时间短的垃圾收集器（如 G1），可以获得更好的整体体验。在Java9里，G1 已经成为了默认的垃圾回收器。

CMS 算法中，GC 管理的内存被划分为新生代、老年代和永久代/元空间。这些空间必须是地址连续的。在G1算法中，采用了另外一种完全不同的方式组织堆内存，堆内存被划分为多个大小相等的内存块（Region），每个Region是逻辑连续的一段内存，Region的大小可以通过 -XX:G1HeapRegionSize 参数指定，如果没有设置，默认把堆内存按照2048份均分，最后得到一个合理的大小。

在G1中，还有一种特殊的区域，叫 Humongous 区域。 如果一个对象占用的空间超过了分区容量 50% 以上，G1 收集器就认为这是一个巨型对象。这些巨型对象，默认直接会被分配在年老代，但是如果它是一个短期存在的巨型对象，就会对垃圾收集器造成负面影响。为了解决这个问题，G1 划分了一个 Humongous 区，它用来专门存放巨型对象。

#### 2.3.5.1. G1 的优缺点
G1 和 CMS相比，有一些显而易见的优点。
- 简单可行的性能调优：使用 -XX:+UseG1GC -Xmx32g 这两个参数就可以用于生产环境的 Java 应用（使用-XX:MaxGCPauseMillis=n设置期望的暂停时间）。
- 取消了老年代的物理空间划分，无需对每个代进行空间大小的设置。

只不过不过，目前 G1 垃圾回收器存在使用场景的限制：

> 许多公开的基准测试都表明，在内存占用相对较小的应用程序中，CMS 的性能往往要胜过 G1，这与 Oracle 对G1的描述一致，即 G1 适用于堆大小为6GB及以上的服务器应用程序。

Elasticsearch 社区的建议：
> 像 Elasticsearch 这样低延迟需求的软件的最佳垃圾收集器。官方建议使用 CMS。Lucene 的测试套件表明 G1 GC 一直都无法完全胜任测试场景下的 GC 工作([Don’t Touch These Settings](https://www.elastic.co/guide/en/elasticsearch/guide/current/_don_8217_t_touch_these_settings.html))。

JVM 大佬 RednaxelaFX:
> 其实CMS在较小的堆、合适的workload的条件下暂停时间可以很轻松的短于G1。在2011年的时候Ramki告诉我堆大小的分水岭大概在10GB～15GB左右：以下的-Xmx更适合CMS，以上的才适合试用G1。现在到了2014年，G1的实现经过一定调优，大概在6GB～8GB也可以跟CMS有一比，我之前见过有在-Xmx4g的环境里G1比CMS的暂停时间更短的案例。

#### 2.3.5.2. G1 执行过程
G1 垃圾收集器会执行一个全局的并发标记阶段来决定堆中的对象的活跃度。之后标记阶段就完成了。G1 收集器知道哪个区域基本上是空的。它首先会收集那些产出大量空闲空间的区域。这就是为什么这个垃圾收集的方法叫做垃圾优先的原因。

有若干介绍 G1 执行垃圾回收过程博客，大多数作者是将其与 CMS 的垃圾回收过程做了类比，即分为了6个阶段（Phase），个人认为这样是不太合适的。比较 CMS 是一个老年代的垃圾回收期，而 G1 的回收，同时涉及到了新生代和老年代。在这里我采用 RednaxelaFX 的解释来叙述 G1 的垃圾回收过程。

从全局来看看，G1垃圾回收可以分为两大部分：
- 全局并发标记（Global Concurrent Marking） 
- 拷贝存活对象（Evacuation）

##### 2.3.5.2.1. Global Concurrent Marking
*Global Concurrent Marking* 是基于 SATB 形式的并发标记，SATB（snapshot-at-the-beginning）是一种比CMS收集器更快的算法。Global Concurrent Marking 具体分为下面几个阶段：

1. 初始标记（STW initial marking）：扫描根集合，标记所有从根集合可直接到达的对象并将它们的字段压入扫描栈。在分代式G1模式中，初始标记阶段借用 Young GC 的暂停，因而没有额外的、单独的暂停阶段。 
2. 并发标记（concrrent marking）：这个阶段可以并发执行，GC 线程 不断从扫描栈取出引用，进行递归标记，直到扫描栈清空。
3. 最终标记（STW final marking，在实现中也叫Remarking）：重新标记写入屏障（ Write Barrier）标记的对象，这个阶段也进行弱引用处理（reference processing）。 
5. 清理（STW cleanup）：统计每个 Region 被标记为活的对象有多少，如果发现完全没有活对象的 Region 就会将其整体回收到可分配 Region 列表中。

##### 2.3.5.2.2. Evacuation
Evacuation阶段是全暂停的。它负责把一部分 Region 里的活对象拷贝到空 Region 里去，然后回收原本的 Region 的空间。

#### 2.3.5.3. G1 分代回收
可以分为 Young GC 和 Mixed GC 两种类型：
- Young GC：选定所有 新生代 里的 Region 。通过控制 新生代 的 Region 个数来控制 Young GC 的开销。 
- Mixed GC：选定所有 新生代 里的 Region ，外加根据 Global Concurrent Marking 统计得出收集收益高的若干老年代 Region 。在用户指定的开销目标范围内尽可能选择收益高的老年代 Region 。
分代式 G1 的正常工作流程就是在 Young GC 与 Mixed GC之间视情况切换，背后定期做做全局并发标记。Initial marking 默认搭在 Young GC 上执行；当全局并发标记正在工作时，G1 不会选择做 Mixed GC，反之如果有 Mixed GC 正在进行中 G1 也不会启动 initial marking。
同 CMS 一样，当所有 Eden Region 被耗尽无法申请内存时，Young GC 就会被触发。
一个假想的混合的STW时间线：
```
-> young GC
-> young GC
-> young GC
-> young GC + initial marking
(... concurrent marking ...)
-> young GC (... concurrent marking ...)
(... concurrent marking ...)
-> young GC (... concurrent marking ...)
-> final marking
-> cleanup
-> mixed GC
-> mixed GC
-> mixed GC
...
-> mixed GC
-> young GC + initial marking
(... concurrent marking ...)
...
```
注意：G1 里不存在Full GC，在正常工作流程中没有 Full GC 的概念，老年代的收集全靠 Mixed GC 来完成，当 Region 无法继续分配对象后，G1 将会 退化成 Serial old 。 

#### 2.3.5.4. G1 调优参数
*-XX:MaxGCPauseMillis=n*
- 设置垃圾收集暂停时间最大值指标，注意这个目标不一定能满足，Java虚拟机将尽最大努力实现它，不建议设置得过小（ < 50ms ）;
*-XX:InitiatingHeapOccupancyPercent=n*
- 触发并发垃圾收集周期的整个堆空间的占用比例。

**最佳实践1**：不要设置新生代的大小
通过 -Xmn 显式地设置新生代大小会干扰 G1 的垃圾回收策略：
- 设置的最大暂停时间指标将不再有效，事实上，设置新生代大小后，将不会启用暂停时间目标。
- G1收集器将不能按需调整新生代的大小空间。

**最佳实践2**：避免晋升失败带来的副作用
晋升失败后，如果此时 JVM 堆内存也耗尽了，就会 出现 Evacuation Failure，在 GC 日志里将会看到 to-space overflow 的日志。 Evacuation Failure 的开销是巨大的，为了避免这种情况，可以执行下面的步骤：
- 增加 -XX:G1ReservePercent 选项的值（并相应增加总的堆大小），为“目标空间”增加预留内存量；
- 通过减少 -XX:InitiatingHeapOccupancyPercent 提前启动标记周期。
- 通过设置 -XX:ConcGCThreads=n 增加并行标记线程的数量；

### 2.3.6. 易混淆的概念
**CMS 和 G1 启动 GC 的时机？**
- 对于 CMS，配置 -XX:CMSInitiatingOccupancyFraction=n 即可，注意这这里的n表示垃圾对象在老年代的空间占比。
- 对于G1，配置 -XX:InitiatingHeapOccupancyPercent=n，表示垃圾对象在整个G1堆内存的空间占比（Mixed GC）。

**什么时间会出现 Full GC？**
对于CMS垃圾回收器：
- Concurrent-mode-failure：当 CMS GC 正进行时，此时有新的对象要进行老年代，但是老年代空间不足造成的；
- Promotion-failed：当进行 Young GC 时，有部分新生代代对象仍然可用，但是S0或S1放不下，因此需要放到老年代，但此时老年代空间无法容纳这些对象。

对于 G1 垃圾回收器：
- 如果 Mixed GC 实在无法跟上程序分配内存的速度，导致老年代填满无法继续进行 Mixed GC，就会切换到 G1之外的 Serial old GC 来收集整个GC Heap（注意，包括young、old、perm），所以，对于 正常工作的G1 垃圾回收期是不能存在Full GC，如果真出现了，估计就很悲剧了，毕竟单线程 + 大内存 + 整个堆，时间开销可想而知。

**Full GC、Magjor GC、Minor GC、Young GC、Old GC 之间的关系？**
- Minor GC == Young GC ，只回收新生代的空间；Old GC是回收老年代的GC；Major GC指的是对老年代/永久代的 STW 的 GC;
- Full GC 是针对整个新生代、老生代、元空间（metaspace，java8以上版本取代 perm gen）的全局范围的GC，是一种不正常的 GC 活动；
- Jstat 命令里的 FGCC、FGCT：
	- 统计的数据里：Full GC的次数 = 老年代GC时 STW 的次数，Full GC的时间 = 老年代GC时 stop the world 的总时间;
	- CMS 不等于Full GC，我们可以看到 CMS 分为多个阶段，只有 STW 的阶段被计算到了Full GC 的次数和时间，而和业务线程并发的 GC 的次数和时间则不被认为是 Full GC;

## 2.4. JVM调优步骤
完成一个JVM的性能测试，了解程序当前的状态，确定瓶颈点；
明确调优的目标，比如减少 FULL GC 的次数、降低暂停的时间、提高吞吐量等、减少GC的总时间等；
调整参数后再进行多次的测试、分析、对比，最终达到一个较为理想的状态，各种参数要根据场景选择，没有统一的解决方案。

# 3. 推荐阅读
[垃圾回收的算法与实现](https://book.douban.com/subject/26821357/)
[深入理解Java虚拟机](https://book.douban.com/subject/24722612/)
[请教G1算法的原理(R大的回答)](http://hllvm.group.iteye.com/group/topic/44381)
[Getting Started with the G1 Garbage Collector](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)