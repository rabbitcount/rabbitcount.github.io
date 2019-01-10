---
title: 算法名词
date: 2017-08-07 23:00:45
tags:
- algo
- 算法
---

# 名词
## 常规算法
分治		Divide and Conqeuer （DC）
动态规划 Dynamic Programming （DP）
贪心		Greedy Algoritym	（GA）
数据缓存	Memorization

### 树
二叉查找树	Binary Search Tree	(BST)：是一棵二叉树，每个节点都大于其左子树的任意节点的键，且小于右子树的任意节点的键
（投影到同一条直线上，是有序的；中根遍历是有序的）


## 其他
快速傅里叶变化	Fast Fourier Transform（FFT）：将多项式相乘的计算时间有O(n^2)变为O(nlogn)

测试的内容

# VS

## AVL树，红黑树，B树，B+树，Trie树

AVL是一种高度平衡的二叉树，所以通常的结果是，维护这种高度平衡所付出的代价比从中获得的效率收益还大，故而实际的应用不多，更多的地方是用追求局部而不是非常严格整体平衡的红黑树。
当然，如果场景中对插入删除不频繁，只是对查找特别有要求，AVL还是优于红黑的。
红黑树的应用就很多了，除了上面同学提到的STL，还有
- 著名的linux进程调度Completely Fair Scheduler,用红黑树管理进程控制块
- epoll在内核中的实现，用红黑树管理事件块
- nginx中，用红黑树管理timer等
- Java的TreeMap实现

B和B+主要用在文件系统以及数据库中做索引等，比如Mysql：B-Tree Index in MySqltrie 树的一个典型应用是前缀匹配，比如下面这个很常见的场景，在我们输入时，搜索引擎会给予提示；还有比如IP选路，也是前缀匹配，一定程度会用到trie
