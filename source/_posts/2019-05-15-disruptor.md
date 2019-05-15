---
title: Disruptor 高并发处理框架？？
date: 2019-05-15 09:21:08
tags: 
- disruptor
- ring
---
# 何用
官方定位：High Performance Inter-Thread Messaging Library
2011年Disruptor论文中的定位：High performance alternative to bounded queues for exchanging data between concurrent threads
用于替代应用内的有界队列
> 疑问：诞生于2011年，在分布式和微服务年代，Disruptor真正的价值体现在哪里？

# 何来
Disrutpor 的母公司 LMAX，目标作出世界上最快的交易平台（the fastest trading platform in the world），因此需要一个低延迟、高性能的jvm平台（needed to do something special to achieve very low-latency and high-throughput with our Java platform）。

> [LMAX aims to be the fastest trading platform in the world. Clearly, in order to achieve this we needed to do something special to achieve very low-latency and high-throughput with our Java platform. Performance testing showed that using queues to pass data between stages of the system was introducing latency, so we focused on optimising this area.](https://lmax-exchange.github.io/disruptor/) 

使用 Disruptor 并非简单的将原有队列替换为 Disruptor 的环形buffer（[magic ring buffer](http://mechanitis.blogspot.com/2011/06/dissecting-disruptor-whats-so-special.html)），而是需要在了解Disruptor的是实现原理后，进行一些改造。

## 有什么大不了 What's the big deal?

## 并发执行的两个关注点
### mutual exclusion 相互排斥
如何实现或管理资源的竞态更新（contended updates）

### visibility of change 变化的可见性
确保变更对其他线程可见

# 成本
## The Cost of Locks
[Locks provide mutual exclusion and ensure that the visibility of change occurs in an ordered manner. Locks are incredibly expensive because they require arbitration when contended. This arbitration is achieved by a context switch to the operating system kernel which will suspend threads waiting on a lock until it is released. During such a context switch, as well as releasing control to the operating system which may decide to do other house-keeping tasks while it has control, execution context can lose previously cached data and instructions. This can have a serious performance impact on modern processors](http://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf)

## The Costs of “CAS”
A more efficient alternative to the use of locks can be employed for updating memory when the target of the update is a single word. These alternatives are based upon the atomic, or interlocked, instructions implemented in modern processors. These are commonly known as CAS(Compare And Swap) operations, e.g. “lock cmpxchg” on x86. A CAS operation is a special machine-code instruction that allows a word in memory to be conditionally set as an atomic operation. For the “increment a counter experiment” each thread can spin in a loop reading the counter then try to atomically set it to its new incremented value. The old and new values are provided as parameters to this instruction. If, when the operation is executed, the value of the counter matches the supplied expected value, the counter is updated with the new value. If, on the other hand, the value is not as expected, the CAS operation will fail. It is then up to the thread attempting to perform the change to retry, re-reading the counter incrementing from that value and so on until the change succeeds. This CAS approach is significantly more efficient than locks because it does not require a context switch to the kernel for arbitration. However CAS operations are not free of cost. The processor must lock its instruction pipeline to ensure atomicity and employ a memory barrier to make the changes visible to other threads. CAS operations are available in Java by using the java.util.concurrent.Atomic* classes. 
