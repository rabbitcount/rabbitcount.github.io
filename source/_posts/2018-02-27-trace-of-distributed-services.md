---
title: trace of distributed services
date: 2018-02-27 22:13:18
tags:
---

[Google Dapper 翻译](http://bigbully.github.io/Dapper-translation/)

# 

现代互联网服务：复杂的大规模分布式系统

Requst Tracing 中的三个基本诉求
- 低损耗
- 应用透明的
- 大范围部署

ubiquitous deployment(无所不在的部署), and continuous monitoring(持续的监控).

# Key Word

- 采样率
- trace context
- span

# Point

## 安全性

由于安全和隐私问题是不可忽略的，dapper中的虽然存储RPC方法的名称，但在这个时候不记录任何有效载荷数据。相反，应用程序级别的Annotation提供了一个方便的可选机制：应用程序开发人员可以在span中选择关联那些为以后分析提供价值的数据。

Dapper还提供了一些安全上的便利，是它的设计者事先没有预料到的。通过跟踪公开的安全协议参数，Dapper可以通过相应级别的认证或加密，来监视应用程序是否满足安全策略。例如。Dapper还可以提供信息，以基于策略的的隔离系统按预期执行，例如支撑敏感数据的应用程序不与未经授权的系统组件进行了交互。这样的测算提供了比源码审核更强大的保障。

## Dapper代码

Dapper代码中中最关键的部分，就是对基础RPC、线程控制和流程控制的组件库的植入，其中包括span的创建，采样率的设置，以及把日志写入本地磁盘

## Dapper损耗

- 4.1 生成跟踪的损耗
生成跟踪的开销是Dapper性能影响中最关键的部分，因为收集和分析可以更容易在紧急情况下被关闭。Dapper运行库中最重要的跟踪生成消耗在于创建和销毁span和annotation，并记录到本地磁盘供后续的收集。根span的创建和销毁需要损耗平均204纳秒的时间，而同样的操作在其他span上需要消耗176纳秒。时间上的差别主要在于需要在跟span上给这次跟踪分配一个全局唯一的ID。
如果一个span没有被采样的话，那么这个额外的span下创建annotation的成本几乎可以忽略不计，他由在Dapper运行期对ThreadLocal查找操作构成，这平均只消耗9纳秒。如果这个span被计入采样的话，会用一个用字符串进行标注--在图4中有展现--平均需要消耗40纳秒。这些数据都是在2.2GHz的x86服务器上采集的。

在Dapper运行期写入到本地磁盘是最昂贵的操作，但是他们的可见损耗大大减少，因为写入日志文件和操作相对于被跟踪的应用系统来说都是异步的。不过，日志写入的操作如果在大流量的情况，尤其是每一个请求都被跟踪的情况下就会变得可以察觉到。我们记录了在4.3节展示了一次Web搜索的负载下的性能消耗。

- 4.2 跟踪收集的消耗

读出跟踪数据也会对正在被监控的负载产生干扰。表1展示的是最坏情况下，Dapper收集日志的守护进程在高于实际情况的负载基准下进行测试时的cpu使用率。在生产环境下，跟踪数据处理中，这个守护进程从来没有超过0.3%的单核cpu使用率，而且只有很少量的内存使用（以及堆碎片的噪音）。我们还限制了Dapper守护进程为内核scheduler最低的优先级，以防在一台高负载的服务器上发生cpu竞争。

Dapper也是一个带宽资源的轻量级的消费者，每一个span在我们的仓库中传输只占用了平均426的byte。作为网络行为中的极小部分，Dapper的数据收集在Google的生产环境中的只占用了0.01%的网络资源。

- 4.3 在生产环境下对负载的影响

每个请求都会利用到大量的服务器的高吞吐量的线上服务，这是对有效跟踪最主要的需求之一；这种情况需要生成大量的跟踪数据，并且他们对性能的影响是最敏感的。在表2中我们用集群下的网络搜索服务作为例子，我们通过调整采样率，来衡量Dapper在延迟和吞吐量方面对性能的影响。

我们看到，虽然对吞吐量的影响不是很明显，但为了避免明显的延迟，跟踪的采样还是必要的。然而，延迟和吞吐量的带来的损失在把采样率调整到小于1/16之后就全部在实验误差范围内。在实践中，我们发现即便采样率调整到1/1024仍然是有足够量的跟踪数据的用来跟踪大量的服务。保持Dapper的性能损耗基线在一个非常低的水平是很重要的，因为它为那些应用提供了一个宽松的环境使用完整的Annotation API而无惧性能损失。使用较低的采样率还有额外的好处，可以让持久化到硬盘中的跟踪数据在垃圾回收机制处理之前保留更长的时间，这样为Dapper的收集组件给了更多的灵活性。

- 4.4 可变采样(根据流量自动调节采样率)

任何给定进程的Dapper的消耗和每个进程单位时间的跟踪的采样率成正比。Dapper的第一个生产版本在Google内部的所有进程上使用统一的采样率，为1/1024。这个简单的方案是对我们的高吞吐量的线上服务来说是非常有用，因为那些感兴趣的事件(在大吞吐量的情况下)仍然很有可能经常出现，并且通常足以被捕捉到。

然而，在较低的采样率和较低的传输负载下可能会导致错过重要事件，而想用较高的采样率就需要能接受的性能损耗。对于这样的系统的解决方案就是覆盖默认的采样率，这需要手动干预的，这种情况是我们试图避免在dapper中出现的。

我们在部署可变采样的过程中，参数化配置采样率时，不是使用一个统一的采样方案，而是使用一个采样期望率来标识单位时间内采样的追踪。这样一来，低流量低负载自动提高采样率，而在高流量高负载的情况下会降低采样率，使损耗一直保持在控制之下。实际使用的采样率会随着跟踪本身记录下来，这有利于从Dapper的跟踪数据中准确的分析。

- 4.5 应对积极采样(Coping with aggressive sampling)

新的Dapper用户往往觉得低采样率--在高吞吐量的服务下经常低至0.01％--将会不利于他们的分析。我们在Google的经验使我们相信，对于高吞吐量服务，积极采样(aggressive sampling)并不妨碍最重要的分析。如果一个显着的操作在系统中出现一次，他就会出现上千次。低吞吐量的服务--也许是每秒请求几十次，而不是几十万--可以负担得起跟踪每一个请求，这是促使我们下决心使用自适应采样率的原因。

- 4.6 在收集过程中额外的采样

上述采样机制被设计为尽量减少与Dapper运行库协作的应用程序中明显的性能损耗。Dapper的团队还需要控制写入中央资料库的数据的总规模，因此为达到这个目的，我们结合了二级采样。

目前我们的生产集群每天产生超过1TB的采样跟踪数据。Dapper的用户希望生产环境下的进程的跟踪数据从被记录之后能保存至少两周的时间。逐渐增长的追踪数据的密度必须和Dapper中央仓库所消耗的服务器及硬盘存储进行权衡。对请求的高采样率还使得Dapper收集器接近写入吞吐量的上限。

为了维持物质资源的需求和渐增的Bigtable的吞吐之间的灵活性，我们在收集系统自身上增加了额外的采样率的支持。我们充分利用所有span都来自一个特定的跟踪并分享同一个跟踪ID这个事实，虽然这些span有可能横跨了数千个主机。对于在收集系统中的每一个span，我们用hash算法把跟踪ID转成一个标量Z，这里0<=Z<=1。如果Z比我们收集系统中的系数低的话，我们就保留这个span信息，并写入到Bigtable中。反之，我们就抛弃他。通过在采样决策中的跟踪ID，我们要么保存、要么抛弃整个跟踪，而不是单独处理跟踪内的span。我们发现，有了这个额外的配置参数使管理我们的收集管道变得简单多了，因为我们可以很容易地在配置文件中调整我们的全局写入率这个参数。

如果整个跟踪过程和收集系统只使用一个采样率参数确实会简单一些，但是这就不能应对快速调整在所有部署的节点上的运行期采样率配置的这个要求。我们选择了运行期采样率，这样就可以优雅的去掉我们无法写入到仓库中的多余数据，我们还可以通过调节收集系统中的二级采样率系数来调整这个运行期采样率。Dapper的管道维护变得更容易，因为我们就可以通过修改我们的二级采样率的配置，直接增加或减少我们的全局覆盖率和写入速度。



# 引文 

```
当然Dapper设计之初，参考了一些其他分布式系统的理念，尤其是Magpie和X-Trace，但是我们之所以能成功应用在生产环境上，还需要一些画龙点睛之笔，例如采样率的使用以及把代码植入限制在一小部分公共库的改造上。
```

```
The basic idea behind request tracing is relatively straightforward: specific inflexion points must be identified within a system, application, network, and middleware -- or indeed any point on a path of a (typically user-initiated) request -- and instrumented. 
These points are of particular interest as they typically represent forks in execution flow, such as the parallelization of processing using multiple threads, a computation being made asynchronously, or an out-of-process network call being made. 
All of the independently generated trace data must be collected, coordinated and collated to provide a meaningful view of a request’s flow through the system.
```
- google: dapper([Stubby](https://landing.google.com/sre/book/chapters/production-environment.html#our-software-infrastructure-XQs4iw))
- twitter: zipkin([Finagle](https://blog.twitter.com/engineering/en_us/a/2011/finagle-a-protocol-agnostic-rpc-system.html))

OpenTracing
