---
title: Java性能调优
date: 2018-02-15 11:24:45
tags:
- jvm
- 性能调优
---

[coggle.it java 性能调优脑图](https://coggle.it/diagram/WpQbJdYyEQAB9nXT/t/java%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98)

[coggle.it jvm trouble shooting](https://coggle.it/diagram/WpQez8nlFwABBWeO/t/fire-troubleshooting-tools-fire)

# 堆和栈

堆和栈是程序运行的关键，很有必要把他们的关系说清楚。

- 栈是运行时的单位，而堆是存储的单位。
- 栈解决程序的运行问题，即程序如何执行，或者说如何处理数据；堆解决的是数据存储的问题，即数据怎么放、放在哪儿。

# 导论

![导论](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/1%E5%AF%BC%E8%AE%BA.png)

# 性能测试方法
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/2%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%96%B9%E6%B3%95.png)

# java线程状态

**首先要清楚线程的状态**

线程的状态有：**new、runnable、running、waiting、timed_waiting、blocked、dead**

各状态说明：
- **New**: 当线程对象创建时存在的状态，此时线程不可能执行；
- **Runnable**：当调用thread.start()后，线程变成为Runnable状态。只要得到CPU，就可以执行；
- **Running**：线程正在执行；
- **Waiting**：执行thread.join()或在锁对象调用obj.wait()等情况就会进该状态，表明线程正处于等待某个资源或条件发生来唤醒自己；
- **Timed_Waiting**：执行Thread.sleep(long)、thread.join(long)或obj.wait(long)等就会进该状态，与Waiting的区别在于Timed_Waiting的等待有时间限制；
- **Blocked**：如果进入同步方法或同步代码块，没有获取到锁，则会进入该状态；
- **Dead**：线程执行完毕，或者抛出了未捕获的异常之后，会进入dead状态，表示该线程结束

其次，对于jstack日志，我们要着重关注如下关键信息
- **Deadlock**：表示有死锁
- **Waiting on condition** ：等待某个资源或条件发生来唤醒自己。具体需要结合jstacktrace来分析，比如线程正在sleep，网络读写繁忙而等待
- **Blocked**：阻塞
- **Waiting on monitor entry**：在等待获取锁
- **in Object.wait()**：获取锁后又执行obj.wait()放弃锁

对于**Waiting on monitor entry** 和 **in Object.wait()的详细描述**：Monitor是 Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 Class的锁。每一个对象都有，也仅有一个 monitor。从下图中可以看出，每个 Monitor在某个时刻，只能被一个线程拥有，该线程就是 "Active Thread"，而其它线程都是 "Waiting Thread"，分别在两个队列 " Entry
 Set"和 "Wait Set"里面等候。在 "Entry Set"中等待的线程状态是 "Waiting for monitor entry"，而在 "Wait Set"中等待的线程状态是 "in Object.wait()"

# java性能调优工具
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/3java%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98%E5%B7%A5%E5%85%B7.png)

## Linux Performance 
[Linux performance 系统性能大牛 -> brendangregg.com，本节图片来源](http://www.brendangregg.com/linuxperf.html)

### Linux perf tools full
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/linux_perf_tools_full.png)

### Linux observability tools \| Linux 性能观测工具
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/Linux%20observability%20tools.jpg)

### Linux performance benchmarking tools | Linux 性能测评工具
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/Linux%20benchmarking%20tools.jpg)

### Linux performance tuning tools | Linux 性能调优工具
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/Linux%20tuning%20tools.jpg)

### Linux static tools
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/linux_static_tools.png)

### Linux observability: *sar*
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/Linux%20observability%20sar.jpg)

### Linux performance observability tools: perf-tools
[perf-tools github](https://github.com/brendangregg/perf-tools#contents)
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/perf-tools.png)

### Linux bcc/BPF Tracing Tools
[bcc-tools github](https://github.com/iovisor/bcc#tools)
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/bcc_tracing_tools.png)

## linux性能工具

### cpu使用率

用户状态时间和系统状态时间，任何使用系统底层资源的操作，都会导致应用占用更多的系统状态时间；

**性能调优的目的**
- 在尽可能短的时间内让CPU使用率尽可能高

**CPU为什么会空闲**
- 应用被同步原语阻塞，知道锁释放才能继续执行；
- 应用在等待某些东西，例如数据库调用锁返回的响应；
- 应用的确无所事事；

**监控可运行的线程数**
运行队列（run queue）

#### vmstat
[不错的整理，output info图片来源](https://www.cnblogs.com/kerrycode/p/6208257.html)
vmstat命令可以查看系统CPU/内存、swap、io等情况：

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901171921858-708989302.png)

上面的命令每隔1秒采样一次，一共采样四次。CPU占用率很高，上下文切换频繁，说明系统有线程正在频繁切换，这可能是你的程序开启了大量的线程存在资源竞争的情况。另外swap也是值得关注的指标，如果swpd过高则可能系统能使用的物理内存不足，不得不使用交换区内存，还有一个例外就是某些程序优先使用swap，导致swap飙升，而物理内存还有很多空余，这些情况是需要注意的。

vmstat output info
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/linux/monitor/vmstat_output.png)

```
收下未整理的资料

三.实际分析
1. r:运行队列的等待进程数

r（run:运行队列正在执行进程数）和 b(block等待CPU资源的进程个数)。当r超过了CPU数目，就会出现CPU瓶颈了。

查看CPU的核的数量：cat /proc/cpuinfo|grep processor|wc -l


在评估cpu的性能优劣时完全照搬网上说的几倍几倍是不准确的，不能只看top里的参数，还得你自己动手看看vmstat显示的run值和blocked值，当出现明 显较多的blocked的时候，就说明cpu产生了瓶颈。而top命令和uptime命令显示的负载均值，只能作为判断系统过去某个时间段的状态的参照，
 与cpu的性能关系不大。

当r值超过了CPU个数，就会出现CPU瓶颈，解决办法大体几种：
1. 最简单的就是增加CPU个数和核数
2. 通过调整任务执行时间，如大任务放到系统不繁忙的情况下进行执行，进尔平衡系统任务
3. 调整已有任务的优先级

(tips:
vmstat中CPU的度量是百分比的。当us＋sy的值接近100的时候，表示CPU正在接近满负荷工作。
但要注意的是，CPU 满负荷工作并不能说明什么，Linux总是试图要CPU尽可能的繁忙，使得任务的吞吐量最大化。
唯一能够确定CPU瓶颈的还是r（运行队列）的值。)

2.cpu使用率
如果CPU的id(空闲率)长期低于10%，那么表示CPU的资源已经非常紧张，应该考虑进程优化或添加更多地CPU。
wa(等待IO)表示CPU因等待IO资源而被迫处于空闲状态，这时候的CPU并没有处于运算状态，而是被白白浪费了，所以“等待IO应该越小越好。”


【top命令和uptime命令显示的负载均值，只能作为判断系统过去某个时间段的状态的参照， 与cpu的性能关系不大。】

文章推荐：
关于CPU的运行队列与系统负载 ：
http://www.cnblogs.com/hecy/p/4128605.html
vmstat详解 (包含实例分析)：

http://blog.chinaunix.net/uid-20775448-id-3668337.html

```

#### top

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174140233-1402600015.png)

通过TOP命令可以详细看出当前系统的CPU、内存、负载以及各进程状态（PID、进程占用CPU、内存、用户）等。从上面的结果看出该系统上安装了MySQL、java，可以看到他们各自的进程ID，假如这时Java进程占用较高的CPU和内存，那么你就要留心了，如果程序中没有计算量特别大、占用内存特别多的代码，可能你的java程序出现了未知的问题，可以根据进程ID做进一步的跟踪。

#### vmstat vs top

相比top，可以看到整个机器的CPU,内存,IO的使用情况，而不是单单看到各个进程的CPU使用率和内存使用率(使用场景不一样)。



#### pidstat
查看系统指标，还有一个第三方工具：pidstat，这个工具还是很好用的，需要先安装：

> yum install sysstat

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174146640-951850615.png)

该命令监控进程id为3618的CPU状态，每隔1秒采样一次，一共采样四次。“%CPU”表示CPU使用情况，“CPU”则表示正在使用哪个核的CPU，这里为0表示正在使用第一个核。如果还要显示线程ID，则可以使用：

> pidstat -p 3618 -u -t 1 4

如果要监控磁盘读写情况，这可以使用：

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174148468-33690562.png)

pidstat还有其他的参数，可以通过pidstat --help获取，再次不再赘述。

### 磁盘使用率

磁盘速度赶不上IO请求时，会导致问题
完成I/O：w_await

#### iostat

### 网络使用率

#### nicstat，netstat

## uptime(系统基本状态)

> [root@centos7_template ~]# uptime 
10:31:42 up 4 days, 1:01, 1 user,load average: 0.02, 0.02, 0.05

```
10:31:42    	
// 当前系统时间
up 4 days, 1:01 
// 持续运行时间,时间越大，说明你的机器越稳定。 
1 user  		
// 用户连接数，是总连接数而不是用户数 
load average: 0.02, 0.02, 0.05  
// 统平均负载，统计最近1，5，15分钟的系统平均负载该命令将显示目前服务器持续运行的时间，以及负载情况。
```

通过这个命令，可以最简便的看出系统当前基本状态信息，这里面最有用是负载指标，如果你还想查看当前系统的CPU/内存以及相关的进程状态，可以使用TOP命令。


## Java监控工具

[Oracle Monitoring Tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr025.html)，[Java Troubleshooting Guide(官方目录)](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html)

Tool or Option       | Description and Usage                                                         
-------------------- | ------
Java Mission Control | Java Mission Control (JMC) is a new JDK profiling and diagnostic tools platform for HotSpot JVM. It s a tool suite basic monitoring, managing, and production time profiling and diagnostics with high performance. Java Mission Control minimizes the performance overhead that's usually an issue with profiling tools. See [Java Mission Control](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr002.html#BABIBBDE).
jcmd utility         | The jcmd utility is used to send diagnostic command requests to the JVM, where these requests are useful for controlling Java Flight Recordings. The JFRs are used to troubleshoot and diagnose JVM and Java Applications with flight recording events. See [The jcmd Utility](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG).                                                                      
Java VisualVM        | This utility provides a visual interface for viewing detailed information about Java applications while they are running on a Java Virtual Machine. This information can be used in troubleshooting local and remote applications, as well as for profiling local applications. See [Java VisualVM](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr010.html#BABEEIFH).                                                 
JConsole utility     | This utility is a monitoring tool that is based on Java Management Extensions (JMX). The tool uses the built-in JMX instrumentation in the Java Virtual Machine to provide information about performance and resource consumption of running applications. See [JConsole](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr009.html#BABDCICF).                                                                           
`jmap` utility       | This utility can obtain memory map information, including a heap histogram, from a Java process, a core file, or a remote debug server. See [The jmap Utility](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr014.html#BABGAFEG).                                                                                                                                                                                      
`jps` utility        | This utility lists the instrumented Java HotSpot VMs on the target system. The utility is very useful in environments where the VM is embedded, that is, it is started using the JNI Invocation API rather than the `java` launcher. See [The jps Utility](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr015.html#BABHCEDC).                                                                                          
`jstack` utility     | This utility can obtain Java and native stack information from a Java process. On Oracle Solaris and Linux operating systems the utility can alos get the information from a core file or a remote debug server. See [The jstack Utility](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr016.html#BABFCHDE).                                                                                                           
`jstat` utility      | This utility uses the built-in instrumentation in Java to provide information about performance and resource consumption of running applications. The tool can be used when diagnosing performance issues, especially those related to heap sizing and garbage collection. See [The jstat Utility](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr017.html#BABCDBEA).                                                  
`jstatd` daemon      | This tool is a Remote Method Invocation (RMI) server application that monitors the creation and termination of instrumented Java Virtual Machines and provides an interface to allow remote monitoring tools to attach to VMs running on the local host. See [The jstatd Daemon](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr033.html#CJGHGBFD).                                                                    
`visualgc` utility   | This utility provides a graphical view of the garbage collection system. As with `jstat`, it uses the built-in instrumentation of Java HotSpot VM. See [The visualgc Tool](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr018.html#BABCDEBE).                                                                                                                                                                          
Native tools         | Each operating system has native tools and utilities that can be useful for monitoring purposes. For example, the dynamic tracing (DTrace) capability introduced in Oracle Solaris 10 operating system performs advanced monitoring. See [Native Operating System Tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr020.html#BABBHHIE).                                                                            

### Java Web Start

工具 | 用途         
:---: | --- | 
javaws	|    

### Monitor Java Applications

工具 | 用途         
:---: | --- | 
[jconsole](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jconsole.html#CACCABEH)   | GUI监控工具    
[jvisualvm](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jvisualvm.html#CBBEGDEJ) | GUI监控工具 

### Monitor the JVM

工具 | 用途         
:---: | --- | 
[jps](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)                      | 列出已装载的JVM，获取进程ID   
[jstat(性能分析)](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html#BEHHGFAE)         | JVM监控统计信息  
[jstatd](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstatd.html#BABEHFHF)       | 远程JVM监控统计信息
[<font color=#0099ff>**jmc**(新)</font>](http://www.oracle.com/technetwork/java/javase/2col/jmc-relnotes-2004763.html) |  <font color=#0099ff>**jmc**</div>：Java Mission Control；<font color=#0099ff>**jfr**</div>：Java Flight Recorder



### Troubleshooting
 工具 | 用途         
:---: | --- | 
[<font color=#0099ff>**jcmd**(新)</font>(建议替代jmap,jstack)](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jcmd.html#CIHEEDIB) | 打印Java进程设计的基本类、线程和VM信息(向正在运行的JVM发送诊断信息请求,是从JDK1.7开始提供可以说是jstack和jps的结合体)
[jinfo](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html#BCGEBFDD)         | 输出JVM配置信息(无需重启，动态调整jvm参数)
[jhat(对jmap导出的堆信息进行分析)](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html#CIHHJAGE)           | 堆离线分析工具(Java Head Analyse Tool)，类比MAT、VisualVM  
[jmap(查看内存，建议使用jcmd)](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html#CEGCECJB)           | 打印JVM堆内对象情况
jsadebugd|
[jstack(查看线程，建议使用jcmd)](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html#BABGJDIF)       | 打印线程堆栈信息  

### Detail

#### jps

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174143827-887028040.png)

可以看到jps命令直接罗列出了当前系统中存在的java进程，获取进程ID

#### jinfo(无需重启，动态调整jvm参数)



#### jstat
用于输出java程序内存使用情况，包括新生代、老年代、元数据区容量、垃圾回收情况。

* -class
* -compiler
* -gc
* -gccapacity
* -gccause
* -gcnew
* -gcnewcapacity
* -gcold
* -gcoldcapacity
* -gcpermcapacity
* -gcutil
* -printcompilation

具体日志输出含义，看[手册](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html#BEHHGFAE)。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174149874-878717667.png)

上述命令输出进程ID为3618的内存使用情况（每2000毫秒输出一次，一共输出20次）

```
S0：幸存1区当前使用比例
S1：幸存2区当前使用比例
E：伊甸园区使用比例
O：老年代使用比例
M：元数据区使用比例
CCS：压缩使用比例
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

#### jcmd (1.7后 建议替代jmap, jstack)

- from jdk1.7
- jstack + jps
- 向正在运行的JVM发送诊断信息请求

##### jcmd常用指令

- 实例查看当前java进程
```
$ jcmd
```

- 查看目标jvm中能获取到的信息
```
$ jcmd 4876 help
4876:
The following commands are available:
VM.native_memory
VM.commercial_features
ManagementAgent.stop
ManagementAgent.start_local
ManagementAgent.start
Thread.print
GC.class_histogram
GC.heap_dump
GC.run_finalization
GC.run
VM.uptime
VM.flags
VM.system_properties
VM.command_line
VM.version
help

····

```

- 查看目标jvm进程的版本信息
```
$ jcmd 4876  VM.version
```

- 查看目标JVM进程的properties
```
$ jcmd 4876 VM.system_properties
```

- 查看虚拟机启动时间
```
$ jcmd 4876 VM.uptime
```

- 查看目标进程的参数
```
$ jcmd 4876 VM.flags
```

- 查看类柱形图（ 同 jmap -histo pid）
```
$ jcmd 4876 GC.class_histogram
```

- 查看JVM性能相关的参数
```
$ jcmd 4876  PerfCounter.print
```

- 显示所有线程栈
```
$ jcmd 4876  Thread.print | more
```

- dump出hprof文件
```
$ jcmd 4876 GC.heap_dump dump.bin
```

- 执行一次finalization操作，相当于执行java.lang.System.runFinalization()
```
$ jcmd 4876 GC.run_finalization
```

##### jcmd help

`jcmd [ options ]`
`jcmd [ pid | main-class ] PerfCounter.print`
`jcmd [ pid | main-class ] command [ arguments ]`
`jcmd [ pid | main-class ] -f file`

- main-class
接收诊断命令请求的进程的main类

- PerfCounter.print
打印目标进程的性能计数器

- file
从文件file中读取命令，然后在目标Java进程上调用这些命令。在file中，每个命令必须写在单独的一行。
以"#"开头的行会被忽略。当所有行的命令被调用完毕后，或者读取到含有stop关键字的命令，
将会终止对file的处理。

#### jmap(1.7后 建议使用jcmd)

使用jmap生成heapdump
```
Usage:
    jmap -clstats <pid>
        to connect to running process and print class loader statistics
    jmap -finalizerinfo <pid>
        to connect to running process and print information on objects awaiting finalization
    jmap -histo[:live] <pid>
        to connect to running process and print histogram of java object heap
        if the "live" suboption is specified, only count live objects
    jmap -dump:<dump-options> <pid>
        to connect to running process and dump java heap

    dump-options:
      live         dump only live objects; if not specified,
                   all objects in the heap are dumped.
      format=b     binary format
      file=<file>  dump heap to <file>

    Example: jmap -dump:live,format=b,file=heap.bin <pid>
```

用于输出java程序中内存对象的情况，包括有哪些对象，对象的数量。

打印JVM堆内对象情况，常用参数有：（具体描述见[手册](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html#CEGCECJB)）

* -dump:[live,]format=b,file=< filename> 

    使用hprof二进制形式,输出jvm的heap内容到文件=. live子选项是可选的，假如指定live选项,那么只输出活的对象到文件。

* -finalizerinfo

    打印正等候回收的对象的信息。

* -heap

    打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况。

* -histo[:live]

    打印每个class的实例数目、内存占用、类全名信息。VM的内部类名字开头会加上前缀“*”。如果live子参数加上后,只统计活的对象数量。

* -clstats

    打印classload的信息。包含每个classloader的名字、活泼性、地址、父classloader和加载的class数量。 

* -F

    在pid没有响应的时候强制使用-dump或者-histo参数。在这个模式下，live子参数无效。

> jmap -histo 3618

上述命令打印出进程ID为3618的内存情况。但我们常用的方式是将指定进程的内存heap输出到外部文件，再由专门的heap分析工具进行分析,例如mat（Memory Analysis Tool），所以我们常用的命令是：

> jmap -dump:live,format=b,file=heap.hprof 3618

将heap.hprof传输出来到window电脑上使用mat工具分析：

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901171922874-980374978.png)

#### jstack(1.7后 建议使用jcmd)

不仅能够导出线程堆栈，还能自动进行死锁检测，输出线程死锁原因；

用户输出java程序线程栈的情况，常用于定位因为某些线程问题造成的故障或性能问题。

> jstack 3618 > jstack.out

#### jhat

主要是用来分析java堆的命令，可以将堆中的对象以html的形式显示出来，包括对象的数量，大小等等，并支持对象查询语言。

用于对JAVA heap进行离线分析的工具，可以对不同虚拟机中导出的heap信息文件进行分析。详细使用见[手册](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html#CIHHJAGE)。

# 垃圾回收
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/4%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6.png)

# 堆内存
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/5%E5%A0%86%E5%86%85%E5%AD%98.png)

# 原生内存
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/6%E5%8E%9F%E7%94%9F%E5%86%85%E5%AD%98.png)

# 线程和同步的性能
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/7%E7%BA%BF%E7%A8%8B%E5%92%8C%E5%90%8C%E6%AD%A5%E7%9A%84%E6%80%A7%E8%83%BD.png)

# JavaEE性能调优
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/8JavaEE%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98.png)

# JavaSE优化
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/9JavaSE%E4%BC%98%E5%8C%96.png)

# JIT即时编译
![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/java_performance/JIT%E5%8D%B3%E6%97%B6%E7%BC%96%E8%AF%91.png)

