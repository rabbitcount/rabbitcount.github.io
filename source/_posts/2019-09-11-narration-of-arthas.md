title: Arthas 源码
date: 2019-09-11 19:12:50
tags: [arthas]
categories: []
---
# Arthas简介
arthas 是 Alibaba 开源的 Java 诊断工具，基于 `jvm Agent` 方式；
使用 `Instrumentation` 方式修改字节码方式以及使用 `java.lang.management` 包提供的管理接口的方式进行java应用诊断。

> [官网地址](https://alibaba.github.io/arthas/)
> [github](https://github.com/alibaba/arthas/)

# Arthas 组成模块

![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-09-11-narration-of-arthas/arthas.png)