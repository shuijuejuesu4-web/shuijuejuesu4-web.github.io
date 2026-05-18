---
title: "MIT 6.1200J: 第19讲 — 条件概率"
date: 2026-05-15
categories: [MIT 6.1200J, 笔记]
tags: [离散数学, 概率, 条件概率, 贝叶斯规则, 辛普森悖论, 蒙提霍尔]
mathjax: true
---

# 第19讲：条件概率（Conditional Probability）

::: tip
条件概率是概率论中最重要也最容易被忽视的概念。贝叶斯规则 $\Pr[A|B] = \frac{\Pr[B|A]\Pr[A]}{\Pr[B]}$ 看起来简单，但它支撑了机器学习中的贝叶斯推断、垃圾邮件过滤器、医学诊断、推荐系统等。理解条件概率，你就理解了「证据如何更新信念」。
:::

## 1 概率规则

**定义 1**：定义事件 $A$ 的概率为

$$\Pr[A] := \sum_{\omega \in A} \Pr[\omega].$$

命题 1（加法规则，Sum Rule）：如果 $A$ 和 $B$ 是不相交的事件，则

$$\Pr[A \cup B] = \Pr[A] + \Pr[B].$$

推论 2（补集规则，Complement Rule）：$\Pr[\overline{A}] = 1 - \Pr[A].$

推论 3（差集规则，Difference Rule）：$\Pr[A \setminus B] = \Pr[A] - \Pr[A \cap B].$

推论 4（容斥原理，Inclusion-Exclusion）：

$$\Pr[A \cup B] = \Pr[A] + \Pr[B] - \Pr[A \cap B].$$

推论 5（联合界，Union Bound）：$\Pr[A \cup B] \leq \Pr[A] + \Pr[B].$

推论 6（单调性规则，Monotonicity Rule）：若 $A \subseteq B$，则 $\Pr[A] \leq \Pr[B].$

## 2 条件概率（Conditional Probability）

如何用数学方式表达"如果汽车在 1 号门后面，参赛者以 $1/3$ 的概率选择 1 号门"这样的陈述？

**定义 2：对于两个事件 $A, B$，给定 $B$ 下 $A$ 的条件概率（conditional probability）**为

$$\Pr[A \mid B] = \frac{\Pr[A \cap B]}{\Pr[B]}.$$

推论 10（乘法规则，Product Rule）：

$$\Pr[A \cap B] = \Pr[A \mid B] \Pr[B].$$

可扩展到多个事件：

$$\Pr[A \cap B \cap C] = \Pr[A \mid B \cap C] \Pr[B \mid C] \Pr[C].$$

乘法规则正是上一讲中树方法计算概率的依据：树边上的数字就是乘积中的项。这意味着树边上的数字（最高层除外）都是**条件概率**！

## 3 示例 1：锦标赛

假设 Ash 和 Gary 进行一系列对战，先赢两场者赢得系列赛。胜负概率行为如下：
1. 第一场势均力敌：每人 $1/2$ 概率获胜。
2. 如果一名训练师赢了上一场，他有 $2/3$ 的概率赢下一场。
3. 没有平局。

令 $A$ 为 Ash 赢得系列赛的事件，$B$ 为他赢得第一场的事件。则

$$\Pr[A \mid B] = \frac{\Pr[A \cap B]}{\Pr[B]} = \frac{1/3 + 1/18}{1/2} = \frac{7}{9}.$$

## 4 贝叶斯规则（Bayes' Rule）

$\Pr[B \mid A]$ 呢？可以同样计算，得到 $7/9$。这样的条件概率表达了一种推断（inference）：给定我们后来观察到 Ash 赢得了整个系列赛，他赢得第一场的概率是多少？

通常我们有"模型"使得计算"前向"条件概率 $\Pr[A \mid B]$ 很容易，但我们真正想知道"后向"概率 $\Pr[B \mid A]$。贝叶斯规则（Bayes' rule）将它们联系起来：

$$\Pr[B \mid A] = \frac{\Pr[A \mid B] \Pr[B]}{\Pr[A]}.$$

一个特别有用的推论是给定 $A$ 下两个事件 $B, C$ 的条件概率之比：

$$\frac{\Pr[B \mid A]}{\Pr[C \mid A]} = \frac{\Pr[A \mid B] \Pr[B]}{\Pr[A \mid C] \Pr[C]}.$$

## 5 示例 2：有偏与公平硬币

假设我有一枚有偏硬币（biased coin）（总是正面）和一枚公平硬币（fair coin）（正面概率一半）。我以均匀概率选一枚硬币并掷出正面。选中的硬币是公平的概率是多少？

令 $H$ 为看到正面的事件，$F$ 为选中公平硬币的事件，$B$ 为选中有偏硬币的事件。

$$\frac{\Pr[F \mid H]}{\Pr[B \mid H]} = \frac{\Pr[H \mid F] \Pr[F]}{\Pr[H \mid B] \Pr[B]} = \frac{1/2 \cdot 1/2}{1 \cdot 1/2} = 1/2.$$

因此公平硬币的概率是 $1/3$，有偏硬币的概率是 $2/3$。

术语：$\Pr[A \mid B]$ 称为似然度（likelihood），$\Pr[B]$ 称为先验概率（prior probability），$\Pr[B \mid A]$ 称为后验概率（posterior probability）。

## 6 示例 3：新冠检测

假设 MIT 社区 10% 的人有新冠（COVID）。检测的假阳性率（false positive rate）为 0.3，假阴性率（false negative rate）为 0.1。如果我检测呈阳性，我患新冠的概率是多少？

- 事件：$H$ 我健康，$S$ 我患病，$+$ 我检测阳性。
- 概率：$\Pr[H] = 0.9$，$\Pr[+ \mid H] = 0.3$，$\Pr[- \mid S] = 0.1$。
- 由此推导：$\Pr[S] = 0.1$，$\Pr[- \mid H] = 0.7$，$\Pr[+ \mid S] = 0.9$。

使用贝叶斯规则（比值形式）：

$$\frac{\Pr[S \mid +]}{\Pr[H \mid +]} = \frac{\Pr[+ \mid S] \Pr[S]}{\Pr[+ \mid H] \Pr[H]} = \frac{0.9 \cdot 0.1}{0.3 \cdot 0.9} = \frac{1}{3}.$$

所以我有 $1/4$ 的概率真的生病，$3/4$ 的概率是健康的！尽管检测看起来不错，但基础患病率（base rate）（先验概率）是主导因素。

## 7 示例 4：辛普森悖论（Simpson's Paradox）

1973 年 UC Berkeley 研究生录取数据的分析揭示了以下矛盾事实：大学整体上男性的录取率高于女性，但对**每个系**而言，男性的录取率都**低于**女性。这怎么可能？

设事件 $A$ 为被录取，$M/F$ 为性别，$EE/CS$ 为申请的系。

直观解释：假设 CS 和 EE 都轻微偏向女性，但 CS 更受女性欢迎，同时 CS（对所有人）都更难进。则 $\Pr[A \mid F]$ 会比 $\Pr[A \mid M]$ 小很多，纯粹因为申请不同系的学生"基础率"不同，而非录取条件概率中的性别差异。

## 8 示例 5：O. J. Simpson 案

O. J. Simpson 被指控谋杀妻子 Nicole。

- **控方**：施虐者比普通人成为谋杀犯的可能性高 10 倍。因此虐待史应被采纳为证据。
- **辩方**：虐待妻子的丈夫谋杀妻子的概率约为 $1/2500$。因此虐待史几乎没有证明价值。

谁是对的？双方都忽略了 Nicole **已被谋杀**这一事实！相关概率是 $\Pr[G \mid A \cap M]$（在虐待且妻子被谋杀的条件下，丈夫是凶手的概率），结果约为 **80%**！

概率和条件概率一直在被使用和误用，甚至专家也会犯（非常公开的）错误。如有疑问，将所有东西精确化并回归基本原理！
