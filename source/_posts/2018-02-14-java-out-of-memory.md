---
title: Java 内存溢出
date: 2018-02-14 22:45:41
tags:
- jvm
- out of memory
---

# 堆溢出

报错信息：_java.lang.OutOfMemoryError: Java heap space_

## 原因

1.  堆中（新生代和老年代）无法继续分配对象了；
2.  某些对象的引用长期被持有没有被释放，垃圾回收器无法回收；
3.  使用了大量的 Finalizer 对象，这些对象并不在 GC 的回收周期内。

## 解决方法

1.  将堆内存 dump 下来，使用 MAT 分析一下，解决内存泄漏；
2.  如果没有内存泄漏，使用 -Xmx 增大堆内存；
3.  如果有自定义的 Finalizable 对象，考虑其存在的必要性。

---

# GC超载溢出

报错信息：_java.lang.OutOfMemoryError：GC overhead limit exceeded_

## 原因

垃圾回收器超过98%的时间用来做垃圾回收，但回收了不到2%的堆内存。

## 解决方法

1. 添加 _-XX:-UseGCOverheadLimit_ 这个启动参数去掉报警，但这只是一种掩耳盗铃的方式，一般出现 _GC overhead limit exceeded_ 说明离真正的 OOM 也不远了；  
2. 将堆内存 _dump_ 下来，使用 MAT 分析一下，解决内存泄漏；  
3. 如果没有内存泄漏，使用 _-Xmx_ 增大堆内存；

---

# 永久代/元空间溢出

报错信息：_java.lang.OutOfMemoryError: PermGen space_ 或者  
_java.lang.OutOfMemoryError: Metaspace_（Java8及以上）

## 原因

永久代是 HotSot 虚拟机对 方法区的具体实现，存放了已被虚拟机加载的类信息、常量、静态变量、JIT编译后的代码等。

需要注意的是，在Java8后，永久代有了一个新名字：元空间，元空间使用的是本地内存。永久代里存在的信息也有了若干变化：

- 字符串常量由永久代转移到堆中；  
- 和永久代相关的JVM参数已移除。

出现永久代或元空间的溢出的原因可能有如下几种：

1. 有频繁的常量池操作（eg. String.intern），这种情况只适用于Java7之前应用；  
2. 加载了大量的类信息，且没有及时卸载；  
3. 应用部署完后没有重启。

没有重启 JVM 进程一般发生在调试时，如下面 tomcat 官网的一个 FAQ：

> Why does the memory usage increase when I redeploy a web application?  
> That is because your web application has a memory leak.  
> A common issue are “PermGen” memory leaks. They happen because the Classloader (and the Class objects it loaded) cannot be recycled unless some requirements are met (). They are stored in the permanent heap generation by the JVM, and when you redeploy a new class loader is created, which loads another copy of all these classes. This can cause OufOfMemoryErrors eventually.  
> (\*) The requirement is that all classes loaded by this classloader should be able to be gc’ed at the same time.


## 解决方法

永久代/元空间 溢出的原因比较简单，解决方法有如下几种：

1.  Java8前的应用：使用 _-XX:MaxPermSize_ 增加永久代的大小（）；
2.  Java8及以后的应用：如果设置了 _-XX:MaxMetaSpaceSize_，调整其大小或者移除掉该参数。
3.  尝试重启JVM。

---

# 方法栈溢出

报错信息：_java.lang.OutOfMemoryError : unable to create new native Thread_

## 原因

虚拟机在拓展栈空间时，无法申请到足够的内存空间。一般出现在内存空间过小，但是又创建了大量的线程的场景。

## 解决方法

1. 通过 _-Xss_ 降低的每个线程栈大小的容量，注意_-Xms，-Xmx_的影响；
2. 线程总数也受到系统空闲内存和操作系统的限制，检查是否该系统下有此限制：
    - /proc/sys/kernel/pid_max，
    - /proc/sys/kernel/thread-max，
    - max_user_process（ulimit -u）
    - /proc/sys/vm/max_map_count

---

# 非常规溢出

## 数组分配溢出

报错信息 ：_java.lang.OutOfMemoryError: Requested array size exceeds VM limit_

这种情况一般是由于不合理的数组分配请求导致的，消除代码逻辑错误或者调整堆大小。

## Swap分区溢出

报错信息 ：_java.lang.OutOfMemoryError: Out of swap space_

这种情况一般是操作系统导致的，可能的原因有：

1. swap 分区大小分配不足；  
2. 机器上其他进程消耗了所有的内存。

根据机器上的日志文件排查。

## 本地方法溢出

报错信息 ：_java.lang.OutOfMemoryError: stack_trace_with_native_method_

这种情况表明，本地方法在运行时出现了内存分配失败。和 _java.lang.OutOfMemoryError : unable to create new native Thread_ 保存不同，方法栈溢出出现在 JVM 的代码层面，而本地方法溢出发生在JNI代码或本地方法处。
