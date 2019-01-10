---
title: 动态规划 Dynamic Programming(DP)
date: 2017-08-05 10:47:50
tags: 
- 算法
- 动态规划
---

# 什么是动态规划 Dynamic Programming (DP)
DP = Divide and Conquer + Memorization
（programming指规划，而非写计算机代码）

## 应用场景
通常应用于**最优化问题**
此类问题通常有多种可行解，希望找出多个解中的最优（最大或最小）解

## DP算法设计
1. 描述最优解的结构
2. 递归定义最优解的值
3. 按照**自底向上**的方式，计算最优解的值（DC是自顶向下）
4. 由计算出的结果构造一个最优解

## DC 与 DP 的差别
DC 将原问题划分为独立的子问题，递归的求解子问题的解
DP 子问题不是独立的，子问题又依赖子子问题（如果使用DC会有重复计算）

# 贪心算法
使所做的选择当前看起来都是最佳的，期望通过所做的局部最优解产生一个全局最优解

## DP 与 GA 的关系
在使用GA前，首先考虑DP，证明总能通过GA的选择得到最优解

# 动态规划的执行过程是什么

每次决策**依赖当前状态**，又随即**引起状态转移**

> DP = Divide and Conquer + Memoization

DP 可視做是 Divide and Conquer 的延伸版本。當運用 Divide and Conquer 所遞迴分割出來的子問題都非常相像的時候，並且當同樣的子問題一而再、再而三出現的時候──就運用 Memoization 儲存全部子問題的答案，節省重複計算相同問題的時間，以空間來換取時間。

>  a method for solving a complex problem by breaking it down into a collection of simpler subproblems, solving each of those subproblems just once, and storing their solutions. The next time the same subproblem occurs, instead of recomputing its solution, one simply looks up the previously computed solution, thereby saving computation time at the expense of a (hopefully) modest expenditure in storage space. (Each of the subproblem solutions is indexed in some way, typically based on the values of its input parameters, so as to facilitate its lookup.) The technique of storing solutions to subproblems instead of recomputing them is called "memoization". ([引用自 wikipedia](https://en.wikipedia.org/wiki/Dynamic_programming) )

# 动态规划 VS 分治

适合于用动态规划法求解的问题，经分解后得到的子问题往往不是互相独立的（即下一个子阶段的求解是建立在上一个子阶段的解的基础上，进行进一步的求解
（重用中间结果）

# 矩阵
动态规划的精髓所在：**一个问题的解依附于其子问题的解**

动态规划利用低阶数据的性质，使其具体表现为二位矩阵

# 对比
已证明的贪婪算法、未证明的贪婪算法（GA, Greedy Algorithm），动态规划（DP, Dynamic Programming），暴力搜索
Greedy ⊂ DP ⊂ Searching

## 动态规划 VS 贪婪算法
贪婪不是“**只顾眼前利益的贪婪**”，而是“**看穿一切，一往无前的气魄**”。
> 贪婪是优化了选择的策略，而非与动态规划背道而驰，Greedy算法仅仅是动态规划的一个平凡态罢了

## 多维解空间和不完全贪心
> [很多时候，问题的解空间并不适合用一维来描述，当解空间在一维以上，比如……还记得经典的0-1背包问题么？那就是一个典型的二维解空间问题。](http://blog.csdn.net/zccz14/article/details/51288079)

## 贪心算法与动态规划的效率区别
从动态规划优化到贪心算法真的提高了算法效率吗？未必。 
这取决于动态规划中计算所有可选的策略的代价，如果代价是常数的，那么将其贪心优化并不会带来时间复杂度的下降（本文的两个例子都是如此），该走的解空间还是要走，只是不保存其中的一些值而已，可能会带来空间复杂度的下降。
有趣的是，当计算策略的代价并非常数（如Floyd全局最短路算法）时，往往并不只有一个策略，因而不能贪心优化。
因此，贪心算法不会降低从其对应的动态规划解法的时间复杂度，如果你发现它降低了，那么一定存在更好的解空间建模，更好的动态规划算法。

> 贪心算法确实可能比其对应的动态规划快不少，因为它的常数可能小得多。 
但是，当你试图对同一解空间的不同点进行多次查询时，你会发现贪心可能会得不偿失，在均摊时间上输给不贪心的动态规划。

## 贪心优化是否失去了什么

**贪心在于其抛弃了部分子结构的解。 **
如顺带要求出方案而不仅仅是最大价值的0-1背包问题，本来使用不优化空间的解法完全能够保留倒推回去的线索，如PAT 1068 Find More Coins 解题报告 所说一般。如果抛弃了动态规划带来的一些解，很有可能在其衍生的问题上得不偿失。


---

# NPC问题

NP问题(Non-deterministic Polynomial )：多项式复杂程度的非确定性问题，这些问题无法根据公式直接地计算出来。比如，找大质数的问题（有没有一个公式，你一套公式，就可以一步步推算出来，下一个质数应该是多少呢？这样的公式是没有的）；再比如，大的合数分解质因数的问题（有没有一个公式，把合数代进去，就直接可以算出，它的因子各自是多少？也没有这样的公式）。

NPC问题(Non-deterministic Polynomial complete)：NP完全问题，可以这么认为，这种问题只有把解域里面的所有可能都穷举了之后才能得出答案，这样的问题是NP里面最难，但是这样算法的复杂程度，是指数关系。一般说来，如果要证明一个问题是NPC问题的话，可以拿已经是NPC问题的一个问题经过多项式时间的变化变成所需要证明的问题，那么所要证明的问题就是一个NPC问题了。NPC问题是一个问题族，如果里面任意一个问题有了多项式的解，即找到一个算法，那么所有的问题都可以有多项式的解。

著名的NPC问题：

背包问题（Knapsack problem）：01背包是在M件物品取出若干件放在空间为W的背包里，每件物品的体积为W1，W2……Wn，与之相对应的价值为V1,V2……Vn。求出获得最大价值的方案。

旅行商问题（Traveling Saleman Problem，TSP），该问题是在寻求单一旅行者由起点出发，通过所有给定的需求点之后，最后再回到原点的最小路径成本。

哈密顿路径问题（Hamiltonian path problem）与哈密顿环路问题（Hamiltonian cycle problem）为旅行推销员问题的特殊案例。哈密顿图：由指定的起点前往指定的终点，途中经过所有其他节点且只经过一次。

欧拉回路（从图的某一个顶点出发，图中每条边走且仅走一次，最后回到出发点；如果这样的回路存在，则称之为欧拉回路。）与欧拉路径（从图的某一个顶点出发，图中每条边走且仅走一次，最后到达某一个点；如果这样的路径存在，则称之为欧拉路径。）

无向图欧拉回路存在条件：所有顶点的度数均为偶数。
无向图欧拉路径存在条件：至多有两个顶点的度数为奇数，其他顶点的度数均为偶数。
有向图欧拉回路存在条件：所有顶点的入度和出度相等。
有向图欧拉路径存在条件：至多有两个顶点的入度和出度绝对值差1（若有两个这样的顶点，则必须其中一个出度大于入度，另一个入度大于出度）,其他顶点的入度与出度相等。