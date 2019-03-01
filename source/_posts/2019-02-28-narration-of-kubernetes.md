---
title: Kubernetes 的那些事儿
date: 2019-02-28 10:35:24
tags:
- kubernetes
- container
- orchestrate
- 容器
- 编排框架
---
K8s，编排容器的框架；
Master components、Control plane、Pod、Container

<!-- more -->

# What is Kubernetes
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-02-28-narration-of-kubernetes/kubernetes-cluster.png)
## What is
编排框架（竞品：Mesos、Docker Swarm），用于调度和惯例容器
> Kubernetes is a portable, extensible open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation.
> Kubernetes是一个可移植的、可扩展的开放源码平台，用于管理容器化的工作负载和服务，这有助于实现声明性配置和自动化

## What is not
> 不是PaaS

相同：operate container，and support deployment, scaling, load balancing, logging, and monitoring
差异：
1. kubernetes is not monolithic, and these default solutions are optional and pluggable; 
2. Kubernetes provides the building blocks for building developer platforms, but preserves user choice and flexibility where it is important.

## Kubernetes含义
源自希腊语中的舵手、飞行员；K8s中的8表示中间8个字母"ubernete"的缩写；
> The name Kubernetes originates from Greek, meaning helmsman or pilot, and is the root of governor and cybernetic. K8s is an abbreviation derived by replacing the 8 letters “ubernete” with “8”.

# Kubernetes Components
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-02-28-narration-of-kubernetes/20190228120800511.png)
## Master Components
> provide the cluster’s control plane

{% exturl 官方高可用解决方案 https://kubernetes.io/docs/setup/independent/high-availability %}，实现将 `Master Components` 部署在集群中的多台机器上

### kube-apiserver
Component on the master that exposes the **Kubernetes API**. It is the **front-end** for the Kubernetes **control plane**.
It is designed to **scale horizontally** – that is, it scales by deploying more instances.
### etcd
Consistent and highly-available key value store used as Kubernetes’ backing store for all cluster data.
Always have a backup plan for etcd’s data for your Kubernetes cluster
{% exturl etcd documentation https://github.com/coreos/etcd/blob/master/Documentation/docs.md %}

### kube-scheduler
Component on the master that watches newly created pods that have no node assigned, and selects a node for them to run on.
### kube-controller-manager
Component on the master that runs controllers .
Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.
These controllers include:
- Node Controller: Responsible for noticing and responding when nodes go down.
- Replication Controller: Responsible for maintaining the correct number of pods for every replication controller object in the system.
- Endpoints Controller: Populates the Endpoints object (that is, joins Services & Pods).
- Service Account & Token Controllers: Create default accounts and API access tokens for new namespaces.

### cloud-controller-manager

## Node Components
Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment

### kubelet
An agent that runs on each node in the cluster. It makes sure that containers are running in a pod.

### kube-proxy
kube-proxy enables the Kubernetes service abstraction by maintaining network rules on the host and performing connection forwarding.

### Container Runtime
The container runtime is the software that is responsible for running containers. Kubernetes supports several runtimes: Docker, containerd, cri-o, rktlet and any implementation of the Kubernetes CRI (Container Runtime Interface).

## Addons

# 名词解析
## ad hoc

# 参考
1. [kubernetes官网](https://kubernetes.io)
2. [三小时攻克kubernetes](http://baijiahao.baidu.com/s?id=1602795888204860650)（[配套git仓库](https://github.com/rabbitcount/k8s-mastery)）