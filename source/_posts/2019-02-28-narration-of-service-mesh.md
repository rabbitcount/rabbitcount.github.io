---
title: 2019-02-28-narration-of-service-mesh
date: 2019-02-28 10:42:58
tags:
- service-mesh
- kubernetes
- docker
---


### 服务治理
#### Service Mesh
解决系统架构微服务化后的服务间通信和治理问题；
适用场景：大规模部署微服务（微服务数>1000）、内部服务异构程度高(交互协议/开发语言类型>5)的场景；
第一代：Linkerd 和 Envoy；
第二代：Istio 和 Conduit；主要改进集中在更加强大的控制面功能（与之对应的 sidecar proxy 被称之为数据面）
> 缺点：
- 网络中多了一跳，增加了性能和延迟的开销；
- 每个服务都需要sidecar, 这会给本来就复杂的分布式系统更加复杂，尤其是在实施初期，运维对service mesh本身把控能力不足的情况下，往往会使整个系统更加难以管理。

参考：[Service Mesh 及其主流开源实现解析](http://www.importnew.com/28798.html)
#### Gateway
相比 Service Mesh，只负责进入的请求，不关注对外的请求；
粒度可粗可细，粗可到整个api总入口，细可到每个服务实例。
### 编排框架
kubernetes、Mesos、Docker Swarm
### 容器
docker
