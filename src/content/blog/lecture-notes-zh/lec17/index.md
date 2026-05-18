---
title: "MIT 6.1200J: 第17讲 — 更多计数"
description: "第17讲：更多计数（More Counting）"
publishDate: 2026-05-15
tags: [离散数学, 计数, 容斥原理, 鸽巢原理, 组合证明, 二项式定理, 多项式定理, 帕斯卡三角, MIT 6.1200J, 笔记, CS110, 讲义]
category: "CS110-讲义"
draft: false
comment: true
---

# 第17讲：更多计数（More Counting）

:::tip
这一讲介绍鸽巢原理——听起来简单（如果 $n+1$ 只鸽子进 $n$ 个巢，必有一巢有至少两只），但它的应用出奇地广泛和精妙。容斥原理则是处理「有重叠」情况下的计数工具。这些方法在算法分析、密码学和数据结构中经常出现。
:::

## 1 容斥原理（Inclusion/Exclusion）

回忆加法规则：$|A \cup B| = |A| + |B|$ 若 $A, B$ 不相交。如果它们相交呢？

一副标准牌中有多少张 Q 和/或红心？4Q + 13♡ = 17 张？但 Q♡ 被数了两次！改为：$4 + 13 - 1 = 16$。

一般地，$|A \cup B| = |A| + |B| - |A \cap B|$。

三个集合的类似公式：

$$|A \cup B \cup C| = |A| + |B| + |C| - |A \cap B| - |A \cap C| - |B \cap C| + |A \cap B \cap C|.$$

**应用**：令 $n = pqr$ 为三个不同素数的乘积。$\{1, 2, \ldots, n\}$ 中有多少个数与 $n$ 互素？

令 $A_p$ 为 $\{1, 2, \ldots, n\}$ 中能被 $p$ 整除的数的集合，类似定义 $A_q$ 和 $A_r$。答案是 $n - |A_p \cup A_q \cup A_r|$。

容斥原理：$|A_p| = n/p$，$|A_p \cap A_q| = n/(pq)$，$|A_p \cap A_q \cap A_r| = n/(pqr) = 1$。

$$\begin{aligned}
|A_p \cup A_q \cup A_r| &= n/p + n/q + n/r - n/(pq) - n/(pr) - n/(qr) + n/(pqr) \\
&= n - (p-1)(q-1)(r-1).
\end{aligned}$$

所以答案是 $(p-1)(q-1)(r-1)$。

更一般地：

$$\left|\bigcup_{i=1}^{n} A_i\right| = \sum_i |A_i| - \sum_{i<j} |A_i \cap A_j| + \sum_{i<j<k} |A_i \cap A_j \cap A_k| - \cdots \pm \left|\bigcap_{i=1}^{n} A_i\right|.$$

定理 1（PIE）：令 $U = \bigcup_{i\in[n]} A_i$ 为有限论域。则

$$\sum_{I\subseteq[n]} (-1)^{|I|} \left|\bigcap_{i\in I} A_i\right| = 0,$$

其中按约定 $\bigcap \varnothing = U$。

**证明概要**：对 $x \in U$，令 $I_x \subseteq [n]$ 为 $x$ 属于的那些 $A_i$ 的指标集。$x$ 对每个偶数大小的 $I \subseteq I_x$ 贡献 $+1$，对每个奇数大小的 $I \subseteq I_x$ 贡献 $-1$。$I_x$ 非空，与 $\{i\}$ 的对称差给出了偶数大小和奇数大小子集 $I \subseteq I_x$ 之间的自逆双射，因此 $x$ 总共贡献 $0$。

## 2 鸽巢原理（Pigeonhole Principle）

定理 2（鸽巢原理）：如果 $|A| > |B|$，且 $f: A \to B$ 是全函数，则 $f$ 不是单射。换言之，存在 $a_1, a_2 \in A$ 使得 $a_1 \neq a_2$ 且 $f(a_1) = f(a_2)$。

更一般地，任何全关系 $R \subseteq A \times B$ 不是单射：至少存在两个不同的 $a_1, a_2 \in A$ 关联到同一个 $b \in B$。

**示例**：房间里有超过 26 个人，那么至少有两人名字的首字母相同。

**示例**：$n$ 双不同颜色的袜子；需要至少取多少只袜子才能保证有一双配对的？鸽巢原理告诉我们 $n+1$ 只就够了。且不能更少，因为可能恰好从每双中取了一只。所以 $n+1$ 是确切答案。

**示例：波士顿约有 650,000 个不秃的人，每人头上最多有 200,000 根头发，因此必存在两个不秃的人有相同数量的头发。

注意：非构造性的（nonconstructive）**！我们知道它们存在，但不知道是谁！

**示例**：不存在能严格缩短所有 $n$ 位串的无损压缩方案。长度为 $n$ 的位串有 $2^n$ 个，但更短的串只有 $2^n - 1$ 个（包括空串）。从较大集合到较小集合的任何全函数必有碰撞，因此不是无损的。

定理 3（广义鸽巢原理）：如果 $|A| > k \cdot |B|$，则从 $A$ 到 $B$ 的每个全关系/函数必须至少有 $k+1$ 个 $A$ 中的元素映射到 $B$ 中同一个元素。

## 3 组合证明/双重计数（Combinatorial Proofs / Double Counting）

$$\sum_{k=0}^{n} \binom{n}{k} = 2^n.$$

**证明**：$\{1, 2, \ldots, n\}$ 有多少个子集？

思路：对每个 $0 \leq k \leq n$，有 $\binom{n}{k}$ 个大小为 $k$ 的子集，因此它们之和应等于子集总数 $2^n$。

更精确地：令 $S$ 为 $\{1, 2, \ldots, n\}$ 的所有子集的集合。我们以两种不同方式计数 $|S|$。

首先，$|S| = 2^n$，因为每个元素要么在子集中要么不在。

按子集大小分情况：令 $S_k$ 为大小为 $k$ 的子集的集合，$S_0, S_1, \ldots, S_n$ 构成 $S$ 的一个划分。由加法规则，$|S| = \sum_{k=0}^{n} |S_k|$。但 $|S_k| = \binom{n}{k}$，故得证。

定理 4（二项式定理，Binomial Theorem）：对任意 $x, y$ 及 $n \in \mathbb{N}$：

$$(x+y)^n = \sum_{k=0}^{n} \binom{n}{k} x^k y^{n-k},$$

其中按约定 $0^0 = 1$。

**证明**：展开 $(x+y)^n$ 得到 $2^n$ 个形如 $a_1 a_2 \cdots a_n$ 的项之和，其中每个 $a_i$ 为 $x$ 或 $y$。由交换律，我们可以将形如 $x^k y^{n-k}$ 的同类项合并。有 $\binom{n}{k}$ 个这样的项——这是从 $n$ 个指标中选择 $k$ 个使 $a_i = x$ 的方式数。

定理 5（多项式定理，Multinomial Theorem）：对任意 $x_1, x_2, \ldots, x_m$ 及 $n \in \mathbb{N}$：

$$\left(\sum_{i=1}^{m} x_i\right)^n = \sum_{k_1+k_2+\cdots+k_m=n} \binom{n}{k_1, k_2, \ldots, k_m} \prod_{i=1}^{m} x_i^{k_i},$$

其中多项式系数（multinomial coefficient）定义为：

$$\binom{n}{k_1, k_2, \ldots, k_m} = \frac{n!}{k_1! k_2! \cdots k_m!}.$$

另一个有用的恒等式：

$$\binom{n}{k} = \binom{n-1}{k-1} + \binom{n-1}{k}.$$

**组合证明**：$\{1, 2, \ldots, n\}$ 有多少个大小为 $k$ 的子集？令 $S$ 为此集合，则 $|S| = \binom{n}{k}$。每个大小为 $k$ 的子集要么包含 $n$ 要么不包含。令 $A$ 为包含 $n$ 的子集，$B$ 为不包含的，$S$ 是 $A$ 和 $B$ 的不交并。$|B| = \binom{n-1}{k}$（不能使用 $n$）。$|A| = \binom{n-1}{k-1}$（必须选 $n$，再从 $\{1, \ldots, n-1\}$ 中选 $k-1$ 个）。故 $|S| = |A| + |B|$ 即上述恒等式。

注意：这个事实表明，如果把数 $\binom{n}{k}$ 排成一个大三角形，每个数是它上方两个数之和。这就是帕斯卡三角（Pascal's Triangle）。

## 附录：记号

$$\bigcup_{i=1}^{n} S_i := S_1 \cup S_2 \cup \ldots \cup S_n$$

$$\bigcap_{i=1}^{n} S_i := S_1 \cap S_2 \cap \ldots \cap S_n$$

$$[n] := \mathbb{N} \cap (0, n] = \{1, 2, \ldots, n\}$$

按约定：$\sum \varnothing = 0$，$\prod \varnothing = 1$，$\bigcup \varnothing = \varnothing$，$\bigcap \varnothing = U$（论域）。
