---
title: "MIT 6.1200J: 第24讲 — 大偏差：切比雪夫与切尔诺夫界"
date: 2026-05-15
categories: [MIT 6.1200J, 笔记]
tags: [离散数学, 概率, 方差, 马尔可夫不等式, 切比雪夫不等式, 切尔诺夫界, 大偏差]
mathjax: true
---

# 第24讲：大偏差：切比雪夫与切尔诺夫界（Large Deviations: Chebyshev and Chernoff Bound）

::: tip
概率能告诉你期望，但「边界」能告诉你离期望有多远。切尔诺夫界回答了「抛 1000 次硬币，正面数偏离 500 超过 50 的概率有多大？」这类问题。这些工具在算法分析中极其重要——当你分析随机化算法的错误概率或负载均衡的最坏情况时，依赖的就是这些不等式。
:::

## 1 复习：方差（Variance）

**定义 1**：$R$ 的方差（variance）为

$$\mathrm{Var}[R] = \mathrm{Ex}\left[(R - \mathrm{Ex}[R])^2\right].$$

$R$ 的标准差（standard deviation），记作 $\sigma(R)$，是方差的（正）平方根。

**定理 1**：$\mathrm{Var}[R] = \mathrm{Ex}[R^2] - \mathrm{Ex}[R]^2.$

**定理 2**：若 $R_1, \ldots, R_n$ 是两两独立的随机变量，则

$$\mathrm{Var}[R_1 + \ldots + R_n] = \mathrm{Var}[R_1] + \ldots + \mathrm{Var}[R_n].$$

**警告**：即使 $R_1$ 和 $R_2$ 独立，$\sigma(R_1 + R_2) \neq \sigma(R_1) + \sigma(R_2)$。但定理 2 告诉我们 $\sigma(R_1 + R_2)^2 = \sigma(R_1)^2 + \sigma(R_2)^2$。

## 2 大偏差界（Large Deviation Bounds）

### 3.1 马尔可夫不等式（Markov's Inequality）

定理 3（马尔可夫不等式）：令 $R$ 为非负随机变量。则

$$\Pr[R \geq x] \leq \frac{\mathrm{Ex}[R]}{x}.$$

**示例**：令 $R$ 为随机人的体重。假设 $\mathrm{Ex}[R] = 100$。则 $\Pr[R \geq 200] \leq 100/200 = 1/2$。这有一个确定的、非概率的解释：至多一半人口体重至少 200 磅。

**替代形式**：$\Pr[R \geq c \cdot \mathrm{Ex}[R]] \leq 1/c.$

### 3.2 有用策略：调整界

如果知道 $S \geq \ell$，尝试对 $S - \ell$ 应用马尔可夫。如果知道 $S \leq u$，尝试对 $u - S$ 应用马尔可夫以界定 $S$ 至多为某值的概率。这包括了随机变量可能为负的情况：如果 $S \geq -4$，不能对 $S$ 应用马尔可夫（因为 $S$ 非负），但 $S + 4$ 是非负的。

### 3.3 马尔可夫通常不紧

在手机检查问题中，$\mathrm{Ex}[R] = 1$。所有人拿回手机的概率？马尔可夫给出 $\leq 1/n$。真实答案？在排列版本中是 $1/(n!)$。$n! \gg n$，所以马尔可夫的估计相去甚远。上界正确但松散（loose）。

**示例：掷 $n$ 枚硬币**。$R = R_1 + \ldots + R_n$，其中 $R_i$ 为指示 RV（第 $i$ 枚为正面）。$\mathrm{Ex}[R_i] = 1/2$，$\mathrm{Var}[R_i] = 1/4$。$\mathrm{Ex}[R] = n/2$，$\mathrm{Var}[R] = n/4$，$\sigma(R) = \sqrt{n}/2$。

马尔可夫告诉我们：$\Pr[R \geq 3n/4] \leq 2/3$。我们之后会做得更好。

## 4 切比雪夫不等式（Chebyshev's Inequality）

定理 5（切比雪夫不等式）：对每个 $x > 0$ 和每个 RV $R$（不必非负），

$$\Pr[|R - \mathrm{Ex}[R]| \geq x] \leq \frac{\mathrm{Var}[R]}{x^2} = \left(\frac{\sigma(R)}{x}\right)^2.$$

$R$ 可以是任意随机变量！不再需要非负！

**证明**：对（非负）随机变量 $(R - \mathrm{Ex}[R])^2$ 使用马尔可夫：

$$\Pr[|R - \mathrm{Ex}[R]| \geq x] = \Pr[(R - \mathrm{Ex}[R])^2 \geq x^2] \leq \frac{\mathrm{Ex}[(R - \mathrm{Ex}[R])^2]}{x^2} = \frac{\mathrm{Var}[R]}{x^2}.$$

**推论**：$\Pr[|R - \mathrm{Ex}[R]| \geq c \cdot \sigma(R)] \leq 1/c^2.$

**示例 1**：回到考试分数（方差 25，标准差 5），$\Pr[\text{分数} \leq 65] \leq \Pr[|\text{分数} - 75| \geq 10] \leq 25/100 = 0.25$。等价于问距离均值至少 $c = 2$ 个标准差的概率，切比雪夫给出至多 $1/c^2 = 1/4$。比单独用马尔可夫好得多！

**示例 2**：掷 $n$ 枚硬币。切比雪夫告诉我们：

$$\Pr[R \geq 3n/4] \leq \Pr[|R - n/2| \geq n/4] \leq \frac{n/4}{(n/4)^2} = \frac{4}{n}.$$

远好于马尔可夫的 $2/3$。

## 5 切尔诺夫界（Chernoff Bound）

切比雪夫只利用了硬币投掷的**两两独立**性。利用所有硬币投掷的**相互独立**性，可以通过切尔诺夫界得到更好的界。

定理 7（切尔诺夫界）：令 $T_1, \ldots, T_n$ 为相互独立的随机变量，满足 $0 \leq T_i \leq 1$ 对所有 $i$。令 $T = T_1 + T_2 + \ldots + T_n$。则对所有 $c \geq 1$，

$$\Pr[T \geq c \cdot \mathrm{Ex}[T]] \leq e^{-(c \ln c - c + 1) \cdot \mathrm{Ex}[T]}.$$

证明与切比雪夫类似，对随机变量 $c^T$ 使用马尔可夫。

**应用于掷硬币**（令 $c = 3/2$）：

$$\Pr[R \geq 3n/4] = \Pr[R \geq 3/2 \cdot n/2] \leq e^{-0.1 \cdot n/2} = e^{-n/20}.$$

这是比切比雪夫的 $4/n$ 呈指数级更好的界！

令 $c = 1 + (4/\sqrt{n})$，可以证明对大的 $n$：

$$\Pr\left[R \geq \frac{n}{2} + 2\sqrt{n}\right] \leq 0.02.$$

注意 $\sqrt{n}$ 比 $n$ 小得多，所以随着 $n$ 增大，这个分布在均值周围（按比例）越来越集中。这是硬币投掷非常集中于 $n/2$ 附近的一种意义。

## 6 结束！

这学期很有意思，谢谢大家！祝期末考试顺利，享受暑假！
