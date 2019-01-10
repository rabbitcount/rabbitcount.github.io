---
title: JVM的Heap Memory和Native Memory
date: 2018-03-12 21:37:47
tags:
- jvm
- heap memory
- native memory
---

# Jvm 内存划分

1.8前，permGen仍未移除前

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/jvm-memory/jvm%207%20memory.png)

jvm管理的内存可以总体划分为两部分
- `Heap Memory`
供Java应用程序使用；Heap Memory及其内部各组成的大小可以通过JVM的一系列命令行参数来控制
- `Native Memory`
也称为C-Heap，是供JVM自身进程使用的；
Native Memory没有相应的参数来控制大小，其大小依赖于操作系统进程的最大值（对于32位系统就是3~4G，各种系统的实现并不一样），以及生成的Java字节码大小、创建的线程数量、维持java对象的状态信息大小（用于GC）以及一些第三方的包，比如JDBC驱动使用的native内存。

# Native Memory 存储的内容

1.  管理java heap的状态数据（用于GC）;
2.  JNI调用，也就是Native Stack;
3.  JIT（即使编译器）编译时使用Native Memory，并且JIT的输入（Java字节码）和输出（可执行代码）也都是保存在Native Memory；
4.  NIO direct buffer。对于IBM JVM和Hotspot，都可以通过-XX:MaxDirectMemorySize来设置nio直接缓冲区的最大值。
默认是64M。超过这个时，会按照32M自动增大。
5.  对于IBM的JVM某些版本实现，类加载器和类信息都是保存在Native Memory中的。

## Native Memory 溢出

简单理解 
> java process memory = java heap + native memory

因此内存溢出时，首先要区分是堆内存溢出还是本地内存溢出。Native Memory本质上就是因为耗尽了进程地址空间。对于 `HotSpot jvm`来书，不断的分配直接内存，会导致如下错误信息：Allocated 1953546760 bytes of native memory before running out

# Direct Buffer
> DirectBuffer对象的数据实际是保存在native heap中，但是引用保存在HeapBuffer中。 

DirectBuffer访问更快，避免了从HeapBuffer还需要从java堆拷贝到本地堆，操作系统直接访问的是DirectBuffer。
另外，DirectBuffer的引用是直接分配在堆得Old区的，因此其回收时机是在FullGC时。因此，需要避免频繁的分配DirectBuffer，这样很容易导致Native Memory溢出。

---

# 线程

这里所说的线程指程序执行过程中的一个线程实体。JVM 允许一个应用并发执行多个线程。Hotspot JVM 中的 Java 线程与原生操作系统线程有直接的映射关系。当线程本地存储、缓冲区分配、同步对象、栈、程序计数器等准备好以后，就会创建一个操作系统原生线程。Java 线程结束，原生线程随之被回收。操作系统负责调度所有线程，并把它们分配到任何可用的 CPU 上。当原生线程初始化完毕，就会调用 Java 线程的 run() 方法。run() 返回时，被处理未捕获异常，原生线程将确认由于它的结束是否要终止 JVM 进程（比如这个线程是最后一个非守护线程）。当线程结束时，会释放原生线程和 Java 线程的所有资源。

## Jvm 系统线程

如果使用 jconsole 或者其它调试器，很多线程在后台运行。这些后台线程与触发 `public static void main(String[])`函数的主线程以及主线程创建的其他线程一起运行。`Hotspot JVM` 后台运行的系统线程主要有下面几个：

线程| 用途             
--- | ---
虚拟机线程（VM thread） | 这个线程等待 JVM 到达安全点操作出现。这些操作必须要在独立的线程里执行，因为当堆修改无法进行时，线程都需要 JVM 位于安全点。这些操作的类型有：stop-the-world 垃圾回收、线程栈 dump、线程暂停、线程偏向锁（biased locking）解除。
周期性任务线程          | 这线程负责定时器事件（也就是中断），用来调度周期性操作的执行。
GC 线程            | 这些线程支持 JVM 中不同的垃圾回收活动。
编译器线程            | 这些线程在运行时将字节码动态编译成本地平台相关的机器码。
信号分发线程  | 这个线程接收发送到 JVM 的信号并调用适当的Jvm方法处理。                                                                                                    

## 线程相关组件

**每个运行的线程** 都 **包含** 下面这些组件

- 线程计数器（Program Counter Register）
- 本地方法栈（Native Stack）
- 栈（Stack）
  - 栈帧（Frame）
  - 

### 程序计数器（Program Counter Register）
> 一块较小的内存地址。可以看做当前锁执行的字节码的行号指示器；

PC 指当前指令（或操作码）的地址，本地指令除外。如果当前方法是 native 方法，那么PC 的值为 undefined。所有的 CPU 都有一个 PC，典型状态下，每执行一条指令 PC 都会自增，因此 PC 存储了指向下一条要被执行的指令地址。JVM 用 PC 来跟踪指令执行的位置，PC 将实际上是指向方法区（Method Area）的一个内存地址。

> 由于Java虚拟机的多线程是通过线程轮流切换并分配处理器的执行时间的方式实现，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个核）都只会执行一条线程中的指令。

### 本地方法栈（Native Stack）

并非所有的 JVM 实现都支持本地（native）方法，那些提供支持的 JVM 一般都会为每个线程创建本地方法栈。如果 JVM 用 C-linkage 模型实现 JNI（Java Native Invocation），那么本地栈就是一个 C 的栈。在这种情况下，本地方法栈的参数顺序、返回值和典型的 C 程序相同。本地方法一般来说可以（依赖 JVM 的实现）反过来调用 JVM 中的 Java 方法。这种 native 方法调用 Java 会发生在栈（一般是 Java 栈）上；线程将离开本地方法栈，并在 Java 栈上开辟一个新的栈帧。

#### 虚拟机栈 vs 本地方法栈
- 虚拟机栈：为虚拟机执行Java方法服务（字节码）
- 本地方法栈：虚拟机使用到的Native方法服务

### 栈（Stack）
> 描述Java方法执行的内存模型：每个方法在执行时都会创建一个栈帧（Stack Frame）用于存储：
**局部变量表、操作数栈、动态链接、方法出口等**
每个方法从调用直至执行完成的过程，对应一个栈帧在虚拟机栈中入栈到出栈的过程；

每个线程拥有自己的栈，栈包含每个方法执行的`栈帧`。栈是一个后进先出（LIFO）的数据结构，因此当前执行的方法在栈的顶部。每次方法调用时，一个新的栈帧创建并压栈到栈顶。当方法正常返回或抛出未捕获的异常时，栈帧就会出栈。除了栈帧的压栈和出栈，栈不能被直接操作。所以可以在堆上分配栈帧，并且不需要连续内存。

#### 栈的限制
栈可以是动态分配也可以固定大小。
- 如果线程请求一个超过允许范围的空间，就会抛出一个`StackOverflowError`
- 如果线程需要一个新的栈帧，但是没有足够的内存可以分配，就会抛出一个 `OutOfMemoryError`。

#### 栈帧（Frame）

每次方法调用都会新建一个新的栈帧并把它压栈到栈顶。当方法正常返回或者调用过程中抛出未捕获的异常时，栈帧将出栈。更多关于异常处理的细节，可以参考下面的异常信息表章节。

每个栈帧包含：

* 局部变量数组
* 返回值
* 操作数栈
* 类当前方法的运行时常量池引用

##### 局部变量数组（Local Variable）

局部变量数组包含了方法执行过程中的所有变量，包括 this 引用、所有方法参数、其他局部变量。对于类方法（也就是静态方法），方法参数从下标 0 开始，对于对象方法，位置0保留为 this。有下面这些局部变量：

* boolean
* byte
* char
* long
* short
* int
* float
* double
* reference
* returnAddress

除了 long 和 double 类型以外，所有的变量类型都占用局部变量数组的一个位置。long 和 double 需要占用局部变量数组两个连续的位置，因为它们是 64 位双精度，其它类型都是 32 位单精度。

##### 操作数栈（Operand Variable）

操作数栈在执行字节码指令过程中被用到，这种方式类似于原生 CPU 寄存器。大部分 JVM 字节码把时间花费在操作数栈的操作上：入栈、出栈、复制、交换、产生消费变量的操作。因此，局部变量数组和操作数栈之间的交换变量指令操作通过字节码频繁执行。比如，一个简单的变量初始化语句将产生两条跟操作数栈交互的字节码。

```
int i;
```

被编译成下面的字节码：

```
0:    iconst_0    // Push 0 to top of the operand stack
1:    istore_1    // Pop value from top of operand stack and store as local variable 1
```

##### 动态链接

> 每个栈帧都有一个运行时常量池的引用

这个引用指向栈帧当前运行方法所在类的常量池。通过这个引用支持动态链接（dynamic linking）。

C/C++ 代码一般被编译成对象文件，然后多个对象文件被链接到一起产生可执行文件或者 dll。在链接阶段，每个对象文件的符号引用被替换成了最终执行文件的相对偏移内存地址。在 Java中，链接阶段是运行时动态完成的。

当 Java 类文件编译时，所有变量和方法的引用都被当做符号引用存储在这个类的常量池中。符号引用是一个逻辑引用，实际上并不指向物理内存地址。JVM 可以选择符号引用解析的时机，一种是当类文件加载并校验通过后，这种解析方式被称为饥饿方式。另外一种是符号引用在第一次使用的时候被解析，这种解析方式称为惰性方式。无论如何 ，JVM 必须要在第一次使用符号引用时完成解析并抛出可能发生的解析错误。绑定是将对象域、方法、类的符号引用替换为直接引用的过程。绑定只会发生一次。一旦绑定，符号引用会被完全替换。如果一个类的符号引用还没有被解析，那么就会载入这个类。每个直接引用都被存储为相对于存储结构（与运行时变量或方法的位置相关联的）偏移量。

## 线程共享

类的元数据, 字符串池, 类的静态变量将会从永久代移除, 放入Java heap或者native memory. 其中建议JVM的实现中将类的元数据放入 native memory, 将字符串池和类的静态变量放入java堆中. 这样可以加载多少类的元数据就不在由MaxPermSize控制, 而由系统的实际可用空间来控制.

### 内存管理（Memory Management）

对象和数组永远不会显式回收，而是由垃圾回收器自动回收。通常，过程是这样的：
- 新的对象和数组被创建并放入老年代。
- Minor垃圾回收将发生在新生代。依旧存活的对象将从 eden 区移到 survivor 区。
- Major垃圾回收一般会导致应用进程暂停，它将在三个区内移动对象。仍然存活的对象将被从新生代移动到老年代。
- 每次进行老年代回收时也会进行永久代回收。它们之中任何一个变满时，都会进行回收。

### 堆（Heap）
堆被用来在运行时分配类实例、数组。不能在栈上存储数组和对象。因为栈帧被设计为创建以后无法调整大小。栈帧只存储指向堆中对象或数组的引用。与局部变量数组（每个栈帧中的）中的原始类型和引用类型不同，对象总是存储在堆上以便在方法结束时不会被移除。对象只能由垃圾回收器移除。

### 非堆内存（No-Heap Memory，Native Memory）
1.8永久代废弃前
- **永久代**，包括：
  * 方法区（Method Area）
  * 驻留字符串（interned strings）
- **代码缓存（Code Cache）**：用于编译和存储那些被 JIT 编译器编译成原生代码的方法。

1.8后：
- 方法区（Method Area）、符号引用（Symbols）移入Metaspace；
- Interned Strings、class statics（静态变量），移入Java Heap

#### 方法区（Method Area）
1.  **Classloader 引用**
2.  **运行时常量池（Runtime Constant Pool）**
    * 数值型常量
    * 字段引用
    * 方法引用
    * 属性
3.  **字段数据**
    * 针对每个字段的信息
        * 字段名
        * 类型
        * 修饰符
        * 属性（Attribute）
4.  **方法数据**
    * 每个方法
        * 方法名
        * 返回值类型
        * 参数类型（按顺序）
        * 修饰符
        * 属性
5.  **方法代码（Method Code）**
    * 每个方法
        * 字节码
        * 操作数栈大小
        * 局部变量大小
        * 局部变量表
        * 异常表
        * 每个异常处理器
        * 开始点
        * 结束点
        * 异常处理代码的程序计数器（PC）偏移量
        * 被捕获的异常类对应的常量池下标

# 类文件结构

编译后的类文件包含下面的结构：
```
ClassFile {
    u4            magic;
    u2            minor_version;
    u2            major_version;
    u2            constant_pool_count;
    cp_info        contant_pool[constant_pool_count – 1];
    u2            access_flags;
    u2            this_class;
    u2            super_class;
    u2            interfaces_count;
    u2            interfaces[interfaces_count];
    u2            fields_count;
    field_info        fields[fields_count];
    u2            methods_count;
    method_info        methods[methods_count];
    u2            attributes_count;
    attribute_info    attributes[attributes_count];
}
```

变量                               | 描述                                                                  
----------------------------------- | ------------------------------------------------------------------------------
magic, minor_version, major_version | 类文件的版本信息和用于编译这个类的 JDK 版本。
constant_pool                       | 类似于符号表，尽管它包含更多数据。下面有更多的详细描述。                                                  
access_flags                        |提供这个类的描述符列表。
this_class                          | 提供这个类全名的常量池(constant_pool)索引，比如org/jamesdbloom/foo/Bar。 
super_class                         | 提供这个类的父类符号引用的常量池索引。 
interfaces                          | 指向常量池的索引数组，提供那些被实现的接口的符号引用。
fields                              | 提供每个字段完整描述的常量池索引数组。 
methods                             | 指向constant_pool的索引数组，用于表示每个方法签名的完整描述。如果这个方法不是抽象方法也不是 native 方法，那么就会显示这个函数的字节码。
attributes                          | 不同值的数组，表示这个类的附加信息，包括 RetentionPolicy.CLASS 和 RetentionPolicy.RUNTIME 注解。 

---
# MetaSpace替换PermGen

## 永久代 permanent generation
- 1.8后移除；
- Hotspot Jvm
- 是一片连续的堆空间，在JVM启动之前通过在命令行设置参数-XX:MaxPermSize来设定永久代最大可分配的内存空间，默认大小是64M（64位JVM由于指针膨胀，默认是85M）。
- 垃圾收集是和老年代(old generation)捆绑在一起的，gc会同时触发永久代和老年代的垃圾收集
- 永久代的参数`-XX:PermSize`和`-XX：MaxPermSize`
- 永久代存了什么：
  - interned-strings：会导致大量的性能问题和OOM错误
  - Method Area

类的元数据信息（metadata）还在，只不过不再是存储在连续的堆空间上，而是移动到叫做“`Metaspace`”的本地内存（Native memory）中

## 永久代移除的意义

由于类的元数据可以在本地内存(native memory)之外分配，所以其最大可利用空间是整个系统内存的可用空间。这样，你将不再会遇到OOM错误，溢出的内存会涌入到交换空间。最终用户可以为类元数据指定最大可利用的本地内存空间，JVM也可以增加本地内存空间来满足类元数据信息的存储。

> 永久代的移除并不意味者类加载器泄露的问题就没有了。因此，你仍然需要监控你的消费和计划，因为内存泄露会耗尽整个本地内存，导致内存交换(swapping)，这样只会变得更糟。

`Metaspace` 和它的内存分配 `Metaspace VM`利用内存管理技术来管理Metaspace。这使得由不同的垃圾收集器来处理类元数据的工作，现在仅仅由`Metaspace VM`在`Metaspace`中通过C++来进行管理。Metaspace背后的一个思想是，类和它的元数据的生命周期是和它的类加载器的生命周期一致的.
> 只要类的类加载器是存活的，在Metaspace中的类元数据也是存活的，不能被释放。

对于术语“Metaspace”。更正式的，每个类加载器存储区叫做“`a Metaspace`”。这些 Metaspaces 一起总体称为”`the Metaspace`”。仅仅当类加载器不在存活，被垃圾收集器声明死亡后，该类加载器对应的 Metaspace 空间才可以回收。Metaspace 空间没有迁移和压缩。但是元数据会被扫描是否存在Java引用。

`Metaspace VM`使用一个块分配器(`chunking allocator`)来管理Metaspace空间的内存分配。块的大小依赖于类加载器的类型。其中有一个全局的可使用的块列表（a global free list of chunks）。当类加载器需要一个块的时候，类加载器从全局块列表中取出一个块，添加到它自己维护的块列表中。当类加载器死亡，它的块将会被释放，归还给全局的块列表。块（chunk）会进一步被划分成blocks,每个block存储一个元数据单元(a unit of metadata)。Chunk中Blocks的分配线性的（pointer bump）。这些chunks被分配在内存映射空间(memory mapped(mmapped) spaces)之外。在一个全局的虚拟内存映射空间（global virtual mmapped spaces）的链表，当任何虚拟空间变为空时，就将该虚拟空间归还回操作系统。

## Metaspace
在JDK8中，classe metadata(the virtual machines internal presentation of Java class)，被存储在叫做`Metaspace`的native memory。一些新的flags被加入：
- `-XX:MetaspaceSize`：class metadata的初始空间配额，以bytes为单位，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当的降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize（如果设置了的话），适当的提高该值。
- `-XX:MaxMetaspaceSize`：可以为class metadata分配的最大空间。默认是没有限制的。
- `-XX:MinMetaspaceFreeRatio`：在GC之后，最小的Metaspace剩余空间容量的百分比，减少为class metadata分配空间导致的垃圾收集
- `-XX:MaxMetaspaceFreeRatio`：在GC之后，最大的Metaspace剩余空间容量的百分比，减少为class metadata释放空间导致的垃圾收集

默认情况下，class metadata的分配仅受限于可用的native memory总量。可以使用MaxMetaspaceSize来限制可为class metadata分配的最大内存。当class metadata的使用的内存达到MetaspaceSize(32位clientVM默认12Mbytes,32位ServerVM默认是16Mbytes)时就会对死亡的类加载器和类进行垃圾收集。设置MetaspaceSize为一个较高的值可以推迟垃圾收集的发生。

Native Heap，就是`C-Heap`
- 对于32位的JVM，`C-Heap的容量 = 4G - Java Heap-PermGen`；
- 对于64位的JVM，`C-Heap的容量 = 物理服务器的总RAM + 虚拟内存 - Java Heap-PermGen`

在JDK8，Native Memory，包括Metaspace和C-Heap。

IBM的J9和Oracle的JRockit(收购BEA公司的JVM)都没有永久代的概念，而Oracle移除HotSpot中的永久代的原因之一是为了与JRockit合并，以充分利用各自的特点。

# Direct Memory

属于C Heap，可以通过参数`-XX:MaxDirectMemorySize`指定。 
如果不指定，该参数的默认值为`Xmx`的值减去1个Survior区的值。如设置启动参数-Xmx20M -Xmn10M -XX：SurvivorRatio=8,那么申请20M-1M=19M的DirectMemory是没有问题的。

```
/*VM Args: -Xmx20M -Xmn10M -XX:MaxDirectMemorySize=10M*/
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args){
        ByteBuffer.allocateDirect(11*_1MB);
    }
}

Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
at java.nio.Bits.reserveMemory(Bits.java:658)
at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
at DirectMemoryOOM.main(DirectMemoryOOM.java:16)
```

修改上面的代码：
```
/*VM Args: -Xmx20M -Xmn10M -XX:MaxDirectMemorySize=10M -XX:+PrintGCDetails*/
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args){
        ByteBuffer.allocateDirect(10*_1MB);
        ByteBuffer.allocateDirect(_1MB);
    }
}

[GC (System.gc()) [PSYoungGen: 983K->632K(9216K)] 983K->640K(19456K), 0.0039667 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 632K->0K(9216K)] [ParOldGen: 8K->505K(10240K)] 640K->505K(19456K), [Metaspace: 2520K->2520K(1056768K)], 0.0073120 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 9216K, used 82K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 1% used [0x00000000ff600000,0x00000000ff614968,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 505K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 4% used [0x00000000fec00000,0x00000000fec7e6c8,0x00000000ff600000)
 Metaspace       used 2527K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 271K, capacity 386K, committed 512K, reserved 1048576K
```
先申请10M，再申请1M，此时会发现JVM不会出现OOM的现象。

可以发现发生了一个FullGC，FullGC后面的system关键字代表这一次FullGC是由System.gc()引起的。

> DirectBuffer的GC规则与堆对象的回收规则是一样的，只有垃圾对象才会被回收，而判定是否为垃圾对象依然是根据引用树中的存活节点来判定。
在垃圾收集时，虽然虚拟机会对DirectMemory进行回收，但是DirectMemory却不像新生代和老年代那样，发现空间不足了就通知收集器进行垃圾回收，它只能等待老年代满了后FullGC，然后“顺便地”帮它清理掉内存中废弃的对象。否则，只能等到抛出内存溢出异常时，在catch块里调用System.gc()。