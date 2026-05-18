---
title: "MIT 6.1200J: 第22讲 — 期望"
description: "第22讲：期望（Expectation）"
publishDate: 2026-05-15
tags: [离散数学, 概率, 期望, 线性期望, 指示变量, 平均故障时间, MIT 6.1200J, 笔记, CS110, 讲义]
category: "CS110-讲义"
draft: false
comment: true
---
# 第22讲：期望（Expectation）

:::tip
期望的线性性质 $\mathrm{Ex}[X+Y] = \mathrm{Ex}[X] + \mathrm{Ex}[Y]$（无论 $X$ 和 $Y$ 是否独立）是概率论中最强大的工具之一。很多看似复杂的期望计算，通过把随机变量拆成简单的指示变量之和，就能轻松搞定。这是算法分析中随机化算法复杂度分析的核心技巧。
:::

## 1 复习

- **随机变量**：从样本空间 $S$ 到另一个集合（通常是非负实数）的（全）函数
- 指示随机变量（indicator random variables）：映射到 $\{0, 1\}$ 的函数
- RV $R$ 与事件 $[R = x] := \{\omega : R(\omega) = x\}$
- 事件 $E$ 与指示 RV $I_E(\omega) = 1$ 若 $\omega \in E$，否则为 0
- 随机变量的独立性
- 概率质量函数（PMF）和累积分布函数（CDF）

## 2 期望（Expectation）

定义 1（随机变量的期望值/平均值/均值）：

$$
\mathrm{Ex}[R] := \sum_{\omega \in S} R(\omega) \cdot \Pr[\omega].
$$

**示例 1**：单次掷骰子，样本空间 $S = \{1, 2, 3, 4, 5, 6\}$，$R(\omega) = \omega$。

$$
\mathrm{Ex}[R] = 1 \cdot \frac{1}{6} + 2 \cdot \frac{1}{6} + 3 \cdot \frac{1}{6} + 4 \cdot \frac{1}{6} + 5 \cdot \frac{1}{6} + 6 \cdot \frac{1}{6} = \frac{7}{2}.
$$

期望值不一定是实际可能看到的值！

**示例 2**：指示随机变量的期望值是它取值为 1 的概率。若 $I_A$ 是事件 $A$ 的指示 RV，

$$
\mathrm{Ex}[I_A] = 0 \cdot \Pr[I_A = 0] + 1 \cdot \Pr[I_A = 1] = \Pr[I_A = 1] = \Pr[A].
$$

## 3 如何赢彩票

三人（学生、助教、教授）各押两枚糖果猜正反面。掷一枚硬币：

- 如果三人全部猜对或全部猜错，各拿回自己的糖果
- 如果两人猜对，6 枚糖果的奖池由此二人平分
- 如果一人猜对，此人独得全部奖池

令 $P$ 为学生收益的随机变量。$\mathrm{Ex}[P] = 0$。但如果助教和教授事先串通总猜相反？则 $\mathrm{Ex}[P] = -1/2$。

## 4 计算期望的替代方法

**定理 1**：

$$
\mathrm{Ex}[R] = \sum_{x \in \text{range}(R)} x \cdot \Pr[R = x].
$$

当有很多结果但 $R$ 只能取较少值时，这个公式更高效。

**定理 2**：若 $\text{range}(R) \subseteq \mathbb{N}$，

$$
\mathrm{Ex}[R] = \sum_{i=0}^{\infty} i \cdot \Pr[R = i] = \sum_{i=0}^{\infty} \Pr[R > i].
$$

后一个等式成立是因为 $\Pr[R = i]$ 在 $\sum \Pr[R > i]$ 中出现恰好 $i$ 次。

## 5 平均故障时间（Mean Time to Failure）

制造零件，每个零件独立地以概率 $p$ 为次品。在见到第一个次品时（包括该次品），期望制造了多少零件？

令 $R$ 为此随机变量。

$$
\mathrm{Ex}[R] = \sum_{i=0}^{\infty} \Pr[R > i] = \sum_{i=0}^{\infty} (1-p)^i = 1/(1-(1-p)) = 1/p.
$$

## 6 关于相似公式的警告

注意区分这四个公式：

1. $\mathrm{Ex}[R] = \sum_{\omega \in S} R(\omega) \cdot \Pr[\omega]$ —— 按**结果** $\omega$ 求和（定义）
2. $\mathrm{Ex}[R] = \sum_{x} x \cdot \Pr[R = x]$ —— 按 RV **值** $x$ 求和
3. $\mathrm{Ex}[R] = \sum_{i=0}^{\infty} \Pr[R > i]$ —— 仅当 $\text{range}(R) \subseteq \mathbb{N}$，按自然数 $i$ 求和
4. $\mathrm{Ex}[R_1 + R_2] = \mathrm{Ex}[R_1] + \mathrm{Ex}[R_2]$ —— 期望的线性性

## 7 无穷期望

假设要估计通信信道上的期望延迟。如果延迟为 $i$ 毫秒的概率是 $1/i$，则期望延迟为 $\sum_{i=1}^{\infty} 1/i$，发散！期望值无界而经验均值有限。

## 8 期望的线性（Linearity of Expectation）

**定理：对随机变量 $R_1$ 和 $R_2$，

$$
\mathrm{Ex}[R_1 + R_2] = \mathrm{Ex}[R_1] + \mathrm{Ex}[R_2].
$$

对比：对事件，$\Pr(A \cap B) = \Pr(A) \cdot \Pr(B)$ 仅当 $A$ 和 $B$ 独立时才成立。$\Pr[A \cup B] = \Pr[A] + \Pr[B]$ 仅当不相交时才成立。

相比之下，期望的线性性适用于任何随机变量，不需要独立性！这使其成为我们工具箱中的强大工具。

**示例：$R$ 为两枚公平骰子之和。$R = R_1 + R_2$，则 $\mathrm{Ex}[R] = 7/2 + 7/2 = 7$。骰子不需要独立！**即使两枚骰子完美相关（总掷出相同点数），同样的计算仍然适用！

## 9 手机检查问题（Cellphone Check Problem）

进入期末考试前，你把手机放进一个袋子（和其他所有人的手机一起）。离开时，你从袋子中随机取一部手机。期望有多少人拿回自己的手机？

令 $R$ 为拿回自己手机的人数。令 $I_i$ 为指示 RV：

$$
I_i = \begin{cases} 1 & \text{如果第 } i \text{ 人拿回自己的手机} \\ 0 & \text{否则} \end{cases}
$$

则 $R = I_1 + I_2 + \ldots + I_n$。

$$
\mathrm{Ex}[R] = \sum_{i=1}^{n} \mathrm{Ex}[I_i] = \sum_{i=1}^{n} \Pr[I_i = 1] = n \cdot \frac{1}{n} = 1.
$$

用蛮力计算 $\Pr[R = k]$ 会非常复杂，但期望的线性性使其异常简单！

**转盘版本**：$n$ 个人去餐厅，把手机放在餐桌中间的转盘上并用力一转。期望有多少人拿回面前的手机？要么所有人拿回（概率 $1/n$），要么没人拿回（概率 $1-1/n$）。同样 $\mathrm{Ex}[R] = 1$。虽然两种游戏中 $I_i$ 都不独立（转盘版本中"更不独立"），但线性期望计算完全相同！
