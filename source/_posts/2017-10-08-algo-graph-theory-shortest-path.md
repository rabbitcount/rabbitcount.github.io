---
title: algo-graph-theory-shortest-path
date: 2017-10-08 11:29:41
mathjax: true
tags:
 - algo
 - shortest path
---

# 最短路径算法
1. 深度、广度优先搜索算法
2. 弗洛伊德算法
3. 迪杰斯特拉算法
4. Bellman-Ford算法

## 名词
边：(u,v)
边上的代价或值：ci,j

## 深度、广度优先搜索算法（解决单元最短路径问题）
从起始结点（单源）开始访问所有的**深度或广度优先路径**，则到达终点结点的路径有多条，取其中路径权值最短的一条则为最短路径（取源点到终点间最短路径）

## 弗洛伊德算法（解决多源最短路径）
时间复杂度O($n^3$),空间复杂度O($n^2$)


测试文本

![测试文本](http://wikianocelot.oss-cn-beijing.aliyuncs.com/fe/flux/flux-stream.png)


$F_{\mu}$
