---
title: linux，Java 监控工具
date: 2018-02-14 21:03:16
tags:
- linux
- jvm
---

# 性能监控
性能分析过程中一切都要可视化，从而了解应用内部及应用所在的环境发生了什么；

http://java.net (已关闭，参见oracle)


# linux 监控工具

## uptime

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

## top

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174140233-1402600015.png)

通过TOP命令可以详细看出当前系统的CPU、内存、负载以及各进程状态（PID、进程占用CPU、内存、用户）等。从上面的结果看出该系统上安装了MySQL、java，可以看到他们各自的进程ID，假如这时Java进程占用较高的CPU和内存，那么你就要留心了，如果程序中没有计算量特别大、占用内存特别多的代码，可能你的java程序出现了未知的问题，可以根据进程ID做进一步的跟踪。除了通过TOP命令找到进程信息以外，还可以通过jdk自带的工具JPS直接找到java程序的进程号。

## jps

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174143827-887028040.png)

可以看到jps命令直接罗列出了当前系统中存在的java进程，这里第一个是jps命令自己的java进程，而另外一个是我启动的nosql监控工具进程。通过这种方法查询到java程序的进程ID后，可以进一步通过：

### top

```
top -p 3618 
// 这里的3618就是上面查询到的java程序的进程ID
```

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174145405-624701483.png)

通过此方法可以准确的查看指定java进程的CPU/内存使用情况。

### vmstat
除此之外，vmstat命令也可以查看系统CPU/内存、swap、io等情况：

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901171921858-708989302.png)

上面的命令每隔1秒采样一次，一共采样四次。CPU占用率很高，上下文切换频繁，说明系统有线程正在频繁切换，这可能是你的程序开启了大量的线程存在资源竞争的情况。另外swap也是值得关注的指标，如果swpd过高则可能系统能使用的物理内存不足，不得不使用交换区内存，还有一个例外就是某些程序优先使用swap，导致swap飙升，而物理内存还有很多空余，这些情况是需要注意的。

### vmstat vs top

相比top，可以看到整个机器的CPU,内存,IO的使用情况，而不是单单看到各个进程的CPU使用率和内存使用率(使用场景不一样)。



### pidstat
查看系统指标，还有一个第三方工具：pidstat，这个工具还是很好用的，需要先安装：

> yum install sysstat

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174146640-951850615.png)

该命令监控进程id为3618的CPU状态，每隔1秒采样一次，一共采样四次。“%CPU”表示CPU使用情况，“CPU”则表示正在使用哪个核的CPU，这里为0表示正在使用第一个核。如果还要显示线程ID，则可以使用：

> pidstat -p 3618 -u -t 1 4

如果要监控磁盘读写情况，这可以使用：

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/352511-20170901174148468-33690562.png)

pidstat还有其他的参数，可以通过pidstat --help获取，再次不再赘述。

# jdk 常用工具汇总

下面再介绍几个JDK自带有用的工具：**jps、jstat、jmap、jstack**

## JDK常用内置工具

工具 | 用途         
----------------------------------------------------------------------------------------------- | -----------
[jps](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)                      | 列出已装载的JVM  
[jstack](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html#BABGJDIF)       | 打印线程堆栈信息   
[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html#BEHHGFAE)         | JVM监控统计信息  
[jmap](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html#CEGCECJB)           | 打印JVM堆内对象情况
[jinfo](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html#BCGEBFDD)         | 输出JVM配置信息  
[jconsole](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jconsole.html#CACCABEH)   | GUI监控工具    
[jvisualvm](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jvisualvm.html#CBBEGDEJ) | GUI监控工具    
[jhat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html#CIHHJAGE)           | 堆离线分析工具    
[jdb](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jdb.html#CHDGJCIE)             | java进程调试工具 
[jstatd](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstatd.html#BABEHFHF)       | 远程JVM监控统计信息


## jps

上面我们已经使用过了，他可以罗列出目前再服务器上运行的java程序及进程ID；

## jstat
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

## jmap
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

## jstack
用户输出java程序线程栈的情况，常用于定位因为某些线程问题造成的故障或性能问题。

> jstack 3618 > jstack.out

## jhat
用于对JAVA heap进行离线分析的工具，可以对不同虚拟机中导出的heap信息文件进行分析。详细使用见[手册](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html#CIHHJAGE)。

# 参考文献

1.  [Java Platform, Standard Edition Tools Reference](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/)
2.  [JDK内置工具使用](http://blog.csdn.net/fenglibing/article/details/6411924)