---
title: Jvm 异常
date: 2018-03-15 22:21:58
tags:
- jvm
---

# OutOfMemoryError 堆溢出

```
/**
 * VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

    static class OOMObject {
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}
```

运行结果

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid36288.hprof ...
Heap dump file created [29130720 bytes in 0.332 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.base/java.util.Arrays.copyOf(Arrays.java:3719)
	at java.base/java.util.Arrays.copyOf(Arrays.java:3688)
	at java.base/java.util.ArrayList.grow(ArrayList.java:237)
	at java.base/java.util.ArrayList.grow(ArrayList.java:242)
	at java.base/java.util.ArrayList.add(ArrayList.java:467)
	at java.base/java.util.ArrayList.add(ArrayList.java:480)
```

# 虚拟机栈与本地方法栈溢出

- 线程请求的栈深度 > 虚拟机允许的最大深度 --> StackOverFlowError
- 虚拟机在扩展栈是无法申请到足够的内存空间 --> OutOfMemoryError

> `-Xss`设置栈内存容量；Hotspot中`-Xoss`参数无效

```
/**
 * VM Args: -Xss144k
 */
public class JavaVMStackSOF {

    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.printf("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

运行结果

```
stack length:495Exception in thread "main" java.lang.StackOverflowError
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:9)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:10)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:10)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:10)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:10)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:10)
	at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:10)
	·······
	at JavaVMStackSOF.main(JavaVMStackSOF.java:16)
```

# 常量池内存溢出

```
/**
 * VM Args: -Xmx20m
 */
public class RuntimeConstantPoolOOm {

    public static void main(String[] args) {
        // 保持对字符串的引用，防止被GC
        List<String> list = new ArrayList<>();
        int i = 0;
        // 1.8后 字符串常量池位于堆中
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```

常量数量过多，导致堆溢出（1.8后不再有permGen）
```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.base/java.util.Arrays.copyOf(Arrays.java:3719)
	at java.base/java.util.Arrays.copyOf(Arrays.java:3688)
	at java.base/java.util.ArrayList.grow(ArrayList.java:237)
	at java.base/java.util.ArrayList.grow(ArrayList.java:242)
	at java.base/java.util.ArrayList.add(ArrayList.java:467)
	at java.base/java.util.ArrayList.add(ArrayList.java:480)
	at RuntimeConstantPoolOOm.main(RuntimeConstantPoolOOm.java:13)

```

# String intern()

> StringBuilder创建的字符串在Java Heap上；
intern()方法的字符串
- 1.6：首次遇到的字符串实例复制到永久代，返回永久代中这个字符串的引用
- 1.7：不会复制实例，只在常量池（Constant Pool）中记录首次出现的实例的引用

1.7后，intern()的实现不会再复制实例，只是在常量池中记录首次出现的实例引用；
因此intern()返回的引用和由StringBuilder创建的字符串上是同一个

```
public class RuntimeConstantPoolOOm {

    public static void main(String[] args) {
        // after 1.7: false (java在本次StringBuilder前，已经存在于虚拟机中，intern返回的jvm中第一次定义predef这个的地址)
        String ori = "gaova";
        String target = new StringBuilder("gao").append("va").toString();
        System.out.println(target.intern() == target);	// false gaova首次出现的堆地址 != 通过StringBuilder新定义字符串的堆地址
        System.out.println(ori == target.intern());		// true  gaova首次出现的堆地址 == gaova首次出现的堆地址(intern获得，记录在常量池中的地址信息) 
    }
}
```

# 方法区溢出

> 1.8后，方法区（Method Area）从永久代（permGen）移入元空间（Metaspace）；
因此，1.8后的方法区溢出实际为Metaspce区溢出：`java.lang.OutOfMemoryError: Metaspace`

```
/**
 * 设置Metaspace区的初始和最大内存均为10m
 * VM Args: -XX:MetaspaceSize=10m -XX:MaxMetaspaceSize=10m
 */
public class JavaMethodAreaOOM {

    static class OOMObject {
    }

    public static void main(String[] args) {

        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(
                    (MethodInterceptor) (obj, method, args1, proxy) -> proxy.invokeSuper(obj, args1));
            enhancer.create();
        }
    }
}
```

溢出
```
[2.541s][info][gc,metaspace,freelist] Metaspace (data) allocation failed for size 42
[2.541s][info][gc,metaspace,freelist] All Metaspace:
[2.541s][info][gc,metaspace,freelist] data space:   Chunk accounting: used in chunks 9266K + unused in chunks 72K  +  capacity in free chunks 5K = 9344K  capacity in allocated chunks 9339K
[2.541s][info][gc,metaspace,freelist] class space:   Chunk accounting: used in chunks 825K + unused in chunks 33K  +  capacity in free chunks 3K = 862K  capacity in allocated chunks 859K
[2.541s][info][gc,metaspace,freelist] Total fragmentation waste (words) doesn't count free space
[2.541s][info][gc,metaspace,freelist]   data: 11 specialized(s) 24, 28 small(s) 47, 80 medium(s) 164, large count 1
[2.541s][info][gc,metaspace,freelist]  class: 11 specialized(s) 0, 8 small(s) 0, 14 medium(s) 6, large count 1
java.lang.OutOfMemoryError: Metaspace
```

# 直接内存溢出

> 直接内存溢出的特点
Heap Dump文件中不会看到明显异常；此时可以检查是否有直接使用NIO的地方

属于C Heap，可以通过参数`-XX:MaxDirectMemorySize`指定。 
如果不指定，该参数的默认值为`Xmx`的值减去1个Survior区的值。如设置启动参数-Xmx20M -Xmn10M -XX：SurvivorRatio=8,那么申请20M-1M=19M的DirectMemory是没有问题的。

```
/**
 * VM Args: -Xmx20m -XX:MaxDirectMemorySize=10M -Xlog:gc*
 */
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        ByteBuffer.allocateDirect(11 * _1MB);
    }
}
```

```
[0.016s][info][gc,heap] Heap region size: 1M
[0.020s][info][gc     ] Using G1
[0.020s][info][gc,heap,coops] Heap address: 0x00000007bec00000, size: 20 MB, Compressed Oops mode: Zero based, Oop shift amount: 3
[0.317s][info][gc,start     ] GC(0) Pause Full (System.gc())
[0.317s][info][gc,phases,start] GC(0) Phase 1: Mark live objects
[0.320s][info][gc,stringtable ] GC(0) Cleaned string and symbol table, strings: 3056 processed, 15 removed, symbols: 22759 processed, 0 removed
[0.320s][info][gc,phases      ] GC(0) Phase 1: Mark live objects 3.421ms
[0.320s][info][gc,phases,start] GC(0) Phase 2: Compute new object addresses
[0.321s][info][gc,phases      ] GC(0) Phase 2: Compute new object addresses 0.881ms
[0.321s][info][gc,phases,start] GC(0) Phase 3: Adjust pointers
[0.323s][info][gc,phases      ] GC(0) Phase 3: Adjust pointers 1.495ms
[0.323s][info][gc,phases,start] GC(0) Phase 4: Move objects
[0.324s][info][gc,phases      ] GC(0) Phase 4: Move objects 1.116ms
[0.324s][info][gc,task        ] GC(0) Using 4 workers of 4 to rebuild remembered set
[0.326s][info][gc,heap        ] GC(0) Eden regions: 3->0(2)
[0.326s][info][gc,heap        ] GC(0) Survivor regions: 0->0(0)
[0.326s][info][gc,heap        ] GC(0) Old regions: 0->2
[0.326s][info][gc,heap        ] GC(0) Humongous regions: 0->0
[0.326s][info][gc,metaspace   ] GC(0) Metaspace: 4904K->4904K(1056768K)
[0.326s][info][gc             ] GC(0) Pause Full (System.gc()) 2M->1M(8M) 9.001ms
[0.326s][info][gc,cpu         ] GC(0) User=0.01s Sys=0.00s Real=0.01s
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
	at java.base/java.nio.Bits.reserveMemory(Bits.java:187)
	at java.base/java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
	at java.base/java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:310)
	at DirectMemoryOOM.main(DirectMemoryOOM.java:11)
[0.858s][info][gc,heap,exit   ] Heap
[0.858s][info][gc,heap,exit   ]  garbage-first heap   total 8192K, used 1036K [0x00000007bec00000, 0x00000007bed00040, 0x00000007c0000000)
[0.858s][info][gc,heap,exit   ]   region size 1024K, 1 young (1024K), 0 survivors (0K)
[0.858s][info][gc,heap,exit   ]  Metaspace       used 5212K, capacity 5244K, committed 5376K, reserved 1056768K
[0.858s][info][gc,heap,exit   ]   class space    used 472K, capacity 492K, committed 512K, reserved 1048576K
```