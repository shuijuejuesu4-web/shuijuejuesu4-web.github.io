---
title: "MIT 6.1200J: 第6讲 — 渐进分析"
date: 2026-05-15
categories: [MIT 6.1200J, 笔记]
tags: [离散数学, 渐进分析, Big-O, 调和数, Stirling公式]
mathjax: true
---

# 第6讲：渐进分析（Asymptotics）

上一讲我们学习了如何计算和近似求和。这一讲我们将用这些工具来解决一个著名的物理问题——Goomy 堆叠——然后引入计算机科学中最重要的工具之一：**渐进记号**。

## 1 Goomy 堆叠问题

上周我们看到了一些计算和近似求和的技巧。今天我们将用这些技巧来解决一个著名的物理问题。

### 1.1 问题设定

规则：

1. 我们有一堆 $n$ 个 Goomy，编号从 $0$ 到 $n-1$。为记号方便，我们将桌子视为 Goomy $n$。
2. 对每个 $i$，Goomy $i$ 必须放在 Goomy $i+1$ 上面（不能有两个 Goomy 在同一高度）。
3. 某些 Goomy 必须延伸出桌边 1 英尺（Goomy 的宽度）。

这可能吗？还是说存在某种基本物理定律，规定只要有一个 Goomy 延伸出去那么远，整个堆叠就必定倒塌？

### 1.2 建立递推关系

我们将看到，实际上这是可能的。定义 $d_i$ 为 Goomy $i$ 的右边缘与 Goomy $0$ 的右边缘之间的水平距离。如果我们可以设置所有的 $d_i$ 使堆叠稳定，并且 $d_n > 1$，那么最顶上的 Goomy 就延伸出了桌边 1 英尺。

$d_i$ 受到重力约束。注意如果最顶上的 Goomy 超过一半悬空在下一个 Goomy 之外，其质心就不在支撑物之上，因此会从堆叠上掉落。类似地，如果对于任何 $k$，最上面的 $k$ 个 Goomy 的集体质心不在下一个 Goomy 之上，它们就会全部掉落。换句话说，我们的约束是：对于每个 $k$，最上面 $k$ 个 Goomy 的集体质心必须在下一个 Goomy 之上。

注意 Goomy $i$ 的右边缘位于 $d_i$，其中心在其右边缘之外另外 $1/2$ 英尺。因此 Goomy $i$ 的质心位于 $d_i + 1/2$。

最上面的 $k$ 个 Goomy 是 Goomy $0, 1, \ldots, k-1$，因此它们的集体质心为：

$$\frac{1}{k}(d_0 + 1/2 + d_1 + 1/2 + \cdots + d_{k-1} + 1/2) = \frac{1}{2} + \frac{1}{k}\sum_{i=0}^{k-1} d_i$$

下一个 Goomy 是 Goomy $k$，其范围从 $d_k$ 到 $d_k+1$，所以我们有：

$$d_k \leq \frac{1}{2} + \frac{1}{k}\sum_{i=0}^{k-1} d_i \leq d_k + 1$$

::: tip

这里的物理直觉：最上面 $k$ 个 Goomy 的集体质心必须落在第 $k+1$ 个 Goomy 的范围内（从 $d_k$ 到 $d_k+1$），否则会翻倒。这就像把书堆叠出桌边——每本书露出的部分不能超过它下面那本书的支撑范围。贪心法（取最小值 $d_k$）给出最极限的悬出距离。

:::

### 1.3 求解递推

特别地，如果我们设（即贪心法，从上到下构建）：

$$d_k = \frac{1}{2} + \frac{1}{k}\sum_{i=0}^{k-1} d_i \quad \text{即} \quad k \cdot d_k = \frac{k}{2} + \sum_{i=0}^{k-1} d_i$$

用**扰动法**简化递推。写出 $k$ 和 $k-1$ 两个版本：

$$\begin{aligned} k \cdot d_k &= \frac{k}{2} + \sum_{i=0}^{k-1} d_i \\ (k-1) \cdot d_{k-1} &= \frac{k-1}{2} + \sum_{i=0}^{k-2} d_i \end{aligned}$$

将两式相减。注意 $\sum_{i=0}^{k-1} d_i = \sum_{i=0}^{k-2} d_i + d_{k-1}$，所以：

$$k \cdot d_k - (k-1) \cdot d_{k-1} = \left(\frac{k}{2} - \frac{k-1}{2}\right) + \left(\sum_{i=0}^{k-1} d_i - \sum_{i=0}^{k-2} d_i\right)$$

化简右边：$\frac{k}{2} - \frac{k-1}{2} = \frac{1}{2}$，求和差为 $d_{k-1}$，得：

$$k \cdot d_k - (k-1) \cdot d_{k-1} = \frac{1}{2} + d_{k-1}$$

整理：

$$k \cdot d_k = (k-1) \cdot d_{k-1} + d_{k-1} + \frac{1}{2} = k \cdot d_{k-1} + \frac{1}{2}$$

两边除以 $k$：

$$d_k = d_{k-1} + \frac{1}{2k}$$

这是一个简单的线性递推！展开 $d_n$：

$$d_n = d_0 + \frac{1}{2}\left(1 + \frac{1}{2} + \frac{1}{3} + \cdots + \frac{1}{n}\right) = \frac{1}{2}\sum_{i=1}^{n} \frac{1}{i}$$

（其中 $d_0 = 0$，因为 Goomy $0$ 的右边缘是参考点。）

## 2 调和数（Harmonic Numbers）

### 2.1 定义与基本性质

**定义 1**：第 $n$ 个调和数（Harmonic Number）为 $H_n = \sum_{i=1}^{n} \frac{1}{i}$。

利用这个定义，$d_n = \frac{1}{2}H_n$。计算前几个调和数：

$$H_1 = 1,\quad H_2 = \frac{3}{2},\quad H_3 = \frac{11}{6},\quad H_4 = \frac{25}{12} > 2$$

由于 $H_4 > 2$，$d_4 > 1$，因此**只用四个 Goomy**，我们就可以让其中一个伸出桌边 1 英尺！

我们知道调和数发散到无穷，因此原则上，只要把 Goomy 堆叠得足够高，就可以让它延伸到桌边以外任意远。例如 $H_{227} \approx 6.004$，所以用 227 个 Goomy 就可以达到 1 码（3 英尺）。

但一个自然的问题是：需要多少个 Goomy？更正式地说，给定 $x$，使 $H_n \geq x$ 的最小 $n$ 是多少？我们能否用 $x$ 的函数以闭式算出这个 $n$ ？

不幸的是，尽管许多杰出的数学家多年来试图寻找答案，没有人知道如何用闭式计算 $H_n$。但是，我们可以**近似** $H_n$！而且有了足够好的 $H_n$ 近似，我们也能近似使 $H_n \geq x$ 的最小 $n$。

::: tip

调和数在算法分析中随处可见：快速排序的平均比较次数是 $2n H_n$，随机化算法中的优惠券收集问题期望为 $n H_n$。你会在算法课中反复遇见它。

:::

### 2.2 积分近似

在第 5 讲中，我们看到了用积分近似求和的方法。这里我们的求和项 $f(x) = 1/x$ 是递减的，所以使用递减序列的积分界限：

$$f(n) + \int_1^n f(x) \, dx \leq \sum_{i=1}^{n} f(i) \leq f(1) + \int_1^n f(x) \, dx$$

取 $f(x) = \frac{1}{x}$，计算积分：

$$\int_1^n \frac{dx}{x} = \ln x \Big|_1^n = \ln n - \ln 1 = \ln n$$

代入界限：

$$\frac{1}{n} + \ln n \leq H_n \leq 1 + \ln n$$

现在上下界的差小于 1！事实上，如果我们简单地用 $\ln n$ 作为 $H_n$ 的近似，误差最多为 1。当 $n$ 趋于无穷时，$\ln n$ 也趋于无穷，因此误差项 1 相比之下变得可以忽略。我们忽略这个误差项，记为 $H_n \sim \ln n$。

定义 2（波浪号记号，Tilde 记号）：$f \sim g$（读作「$f$ tilde $g$」）如果 $\lim_{x \to \infty} \frac{f(x)}{g(x)} = 1$。

Tilde 记号给出了 $f$ 和 $g$「大致相等」的含义；如果我们只关心近似的极限行为，可以安全地忽略 $f$ 和 $g$ 之间的精确差异并将它们视为相同。

### 2.3 回到 Goomy 问题

使用 tilde 记号，$H_n \sim \ln n$。利用这个近似，若想让 Goomy 伸出桌边 1 链（22 码），$H_n = 2d_n \approx 132$，所以 $n \approx e^{132}$，约一百亿亿亿（$10^{57}$）。

::: tip

虽然数学上可能，但物理学处处受限：离开地球引力井需要约 $10^9$ 个 Goomy，恒星会阻挡约 $10^{18}$ 个，Heisenberg 不确定性原理使堆叠不可观测约 $10^{36}$ 个，堆叠质量超过整个宇宙约 $10^{54}$ 个。所以放心——没人真能用 Goomy 堆到月球。

:::

## 3 近似乘积

**定义 3**：$\prod_{i=1}^{n} x_i$ 表示乘积 $x_1 \times x_2 \times \cdots \times x_n$。

**定义 4**：$n!$（读作「$n$ 的阶乘」）是最小 $n$ 个正整数的乘积：$n! = \prod_{i=1}^{n} i$。

类似于 $H_n$，$n!$ 在计算机科学中非常重要，特别是在计数和概率中。我们可以按定义迭代计算，但我们希望更高效地计算它。

### 3.1 乘积转求和：取对数的技巧

正如我们可以通过重复平方法用大约 $\log_2 n$ 次乘法（而不是 $n$ 次）计算 $a^n$，我们也希望找到类似的算法来计算 $n!$。关键的技巧是 **取对数**：对数将乘积转为求和（$\log(a \cdot b) = \log a + \log b$），这样我们就可以用上一讲的积分近似方法。

$$\ln(n!) = \ln\left(\prod_{i=1}^{n} i\right) = \sum_{i=1}^{n} \ln i$$

::: tip

取对数将乘积变成求和，这是处理阶乘、排列数等巨大乘积的标准技巧。在机器学习中，你经常看到 $\log$ 似然函数（log-likelihood）——原因完全相同：把概率乘积变成对数概率之和，数值更稳定，求导更方便。

:::

### 3.2 用积分近似 $\ln(n!)$

这次 $f(x) = \ln x$，是递增的。使用递增函数的积分界限（定理 1）：

$$f(1) + \int_1^n f(x) \, dx \leq \sum_{i=1}^{n} f(i) \leq f(n) + \int_1^n f(x) \, dx$$

其中 $f(1) = \ln 1 = 0$。用**分部积分法**计算积分：

$$\int_1^n \ln x \, dx$$

令 $u = \ln x$，$dv = dx$，则 $du = \frac{1}{x} dx$，$v = x$。分部积分公式 $\int u\,dv = uv - \int v\,du$ 给出：

$$\int_1^n \ln x \, dx = x \ln x \Big|_1^n - \int_1^n x \cdot \frac{1}{x} \, dx = n \ln n - 0 - \int_1^n 1 \, dx = n \ln n - (n - 1) = n \ln n - n + 1$$

代入积分界限：

$$0 + (n \ln n - n + 1) \leq \sum_{i=1}^{n} \ln i \leq \ln n + (n \ln n - n + 1)$$

即：

$$n \ln n - n + 1 \leq \ln(n!) \leq n \ln n - n + 1 + \ln n$$

取指数（$e$ 的幂）：

$$\frac{n^n}{e^{n-1}} \leq n! \leq \frac{n^{n+1}}{e^{n-1}}$$

这次我们近似到了 $n$ 的因子范围内（而不是加法常数范围内）。然而考虑到 $n!$ 的大小（超过指数级别），即使是乘法因子 $n$ 也相当好了！

### 3.3 Stirling 公式

一个更精确的近似，称为 **Stirling 公式**：

$$n! = \left(\frac{n}{e}\right)^n \sqrt{2\pi n} \; e^{\epsilon(n)}, \quad \text{其中 } \frac{1}{12n+1} \leq \epsilon(n) \leq \frac{1}{12n}$$

乘法误差小于 $1 + \frac{1}{144n^2}$。由于 $\epsilon(n) \to 0$（$n \to \infty$），我们可以去掉 $e^{\epsilon(n)}$ 简写为：

$$n! \sim \left(\frac{n}{e}\right)^n \sqrt{2\pi n}$$

::: tip

Stirling 公式中 $\sqrt{2\pi n}$ 从哪来？它来自对 $\ln n!$ 的更精细近似（Euler-Maclaurin 求和公式），$\pi$ 的出现和 Wallis 乘积有关。记不住完整公式的话，至少记住主导项 $n! \approx (n/e)^n$——这对大多数算法分析已经足够。

:::

## 4 渐进记号（Asymptotic Notation）

在计算机科学中，我们通常可以容忍更不精确的近似。例如在讨论算法运行时间时，我们甚至不太关心乘法常数因子误差。机器可能不同，所以我们想要归一化掉诸如单条指令在特定机器上执行时间之类的因素。我们真正关心的是当输入规模增长时，算法的运行时间如何变化。为了讨论这类近似，我们使用**渐进记号**。

::: tip

面试和考试中经常直接问算法复杂度。理解这五个记号（$O, o, \Omega, \omega, \Theta$）是算法分析的基础。一个简单的记忆法：大写字母代表"宽松"不等式（$\leq$ 和 $\geq$），小写字母代表"严格"不等式（$<$ 和 $>$），$\Theta$ 代表"相等"。

:::

在下文中，假设 $g: \mathbb{Z}^+ \to \mathbb{R}^+$，$f: \mathbb{Z}^+ \to \mathbb{R}$。

### 4.1 Big-O：渐进上界

定义 6（Big-O）：$f \in O(g)$ 当且仅当

$$\exists c \in \mathbb{R}. \; \exists M \in \mathbb{Z}^+. \; \forall x \in \mathbb{Z}^+. \; \big[x > M \Rightarrow |f(x)| \leq c \cdot g(x)\big]$$

$O(g)$ 是渐进上界不超过 $g$ 的函数集合。$c$ 意味着我们不关心常数因子；$M$ 意味着我们不关心在接近 0 的有界有限集上的行为。我们将 Big-O 视为捕获了 $\leq$ 的概念（模常数因子和有界例外）。

**极限测试**：如果 $\lim_{x \to \infty} \frac{|f(x)|}{g(x)} \in \mathbb{R}$，则 $f \in O(g)$。但反之不成立！

**例子**：

- $x \in O(x^2)$ — 极限为 0，满足条件
- $3\sin x \in O(1)$ — 取 $M=0, c=3$，虽然极限不存在，但定义满足
- $x^2 \notin O(x)$ — 对任意 $c, M$，取 $x > \max(c, M)$，则 $x^2 > c \cdot x$
- 任意二次函数 $\in O(x^2)$ — 取 $c$ 为首项系数的绝对值加一点余量
- 任意多项式 $\in O(2^x)$ — 指数增长最终碾压多项式增长
- $4^x \notin O(2^x)$ — 两者的比值 $(4/2)^x = 2^x \to \infty$

### 4.2 Little-o：严格的渐进上界

定义 7（Little-o）：$f \in o(g)$ 如果 $\lim_{x \to \infty} \frac{f(x)}{g(x)} = 0$。

$o(g)$ 是渐进远小于 $g$ 的函数集合。如果 Big-O 代表非严格不等式 $\leq$，那么 little-o 就是其严格对应，捕获 $<$ 的概念。

**例子**：
- $x \in o(x^2)$ — $\lim x/x^2 = 0$ ✓
- $3\sin x \notin o(1)$ — 极限不存在 ✗
- $x^2 \notin o(x)$ — $\lim x^2/x = \infty$ ✗
- 二次函数 $\notin o(x^2)$ — 极限为非零常数（首项系数）✗
- 任意多项式 $\in o(2^x)$ — 指数碾压多项式 ✓

**定理**：如果 $f \in o(g)$，则 $f \in O(g)$。直观理解：比 $g$ 小得多的函数当然也不超过 $g$。

### 4.3 Big-$\Omega$：渐进下界

定义 8（Big-$\Omega$）：$f \in \Omega(g)$ 如果 $g \in O(f)$。

$\Omega(g)$ 是渐进下界不低于 $g$ 的函数集合，代表非严格不等式 $\geq$。注意定义的巧妙：$f$ 渐进不低于 $g$，等价于 $g$ 渐进不超过 $f$。

**极限测试**：如果 $\lim_{x \to \infty} \frac{|f(x)|}{g(x)} \in (0, \infty]$，则 $f \in \Omega(g)$。

**例子**：

- $x^2 \in \Omega(x)$ — $\lim x^2/x = \infty$，满足
- $2^x \in \Omega(x^2)$ — 指数增长碾压二次方
- $\frac{x}{100} \in \Omega(100x + \sqrt{x})$ — $\lim \frac{x/100}{100x} = \frac{1}{10000} \in (0, \infty)$

### 4.4 Little-$\omega$：严格的渐进下界

定义 9（Little-$\omega$）：$f \in \omega(g)$ 如果 $g \in o(f)$。

代表严格不等式 $>$。等价刻画为 $\lim_{x \to \infty} \frac{f(x)}{g(x)} = \infty$——直观上，$f$ 不仅不低于 $g$，而且远远超过它。

### 4.5 $\Theta$：渐进相等

**定义 10**：$f \in \Theta(g)$ 如果 $f \in O(g)$ 且 $f \in \Omega(g)$。

代表渐进相等（$=$）——$f$ 和 $g$ 在常数因子范围内增长相同。这是算法分析中最常用的记号：当你说"归并排序是 $\Theta(n \log n)$ 的"，你同时给出了上界和下界。

**极限测试**：如果 $\lim_{x \to \infty} \frac{f(x)}{g(x)} \in \mathbb{R}^+$（极限存在且为正实数），则 $f \in \Theta(g)$。

**例子**：

- $10x^3 + 20x^2 + 5 \in \Theta(x^3)$ — 主导项是 $x^3$，其余可忽略
- $2 + \sin x \in \Theta(1)$ — 在常数因子范围内有界
- $\frac{x}{\ln x} \notin \Theta(x)$ — 比值 $\frac{x/\ln x}{x} = \frac{1}{\ln x} \to 0$，不满足 $\Theta$

### 4.6 注意事项

- **绝不要写** $f = O(g)$。这是对记号的滥用，会引发荒谬的推导。写 $f \leq O(g)$ 更可接受，但 Big-O 并不遵循不等式的所有规则。
- 另一个滥用是写 $f(n) \in O(g(n))$（如 $n \in O(n^2)$）。技术上不完全正确（吹毛求疵地说应该是 $(n \mapsto n) \in O(n \mapsto n^2)$），后者读起来很糟糕。$n \in O(n^2)$ 是清晰且无歧义的，请使用此记号。
- 类似 $f \geq O(g)$ 的语句毫无意义，因为常数零函数属于 $O(g)$。
- $O$ 和 $o$ **仅**是上界，$\Omega$ 和 $\omega$ **仅**是下界。
- 解析数论中使用不同的 $\Omega$ 和 $\omega$ 定义，但 CS 中的定义更强。

::: tip

日常工作中大家普遍写 $f = O(g)$（等号而非 $\in$），论文和教科书也这样写。但你要知道这只是一种约定——$O(g)$ 本质是函数的**集合**，$\in$ 才是正确的。考试中建议用 $\in$，面试中用 $=$ 也没问题。

:::

### 4.7 总结

| 记号 | 定义 | 极限测试 | 含义 |
|------|------|----------|------|
| $f \sim g$ | $\lim f/g = 1$ | $\lim f/g = 1$ | "$=$" |
| $f \in O(g)$ | $\exists c. \exists M. \forall x > M. \|f(x)\| \leq c \cdot g(x)$ | $\lim \|f\|/g \in \mathbb{R}$ | "$\leq$" |
| $f \in o(g)$ | $\lim f/g = 0$ | $\lim f/g = 0$ | "$<$" |
| $f \in \Omega(g)$ | $g \in O(f)$ | $\lim f/g \in (0, \infty]$ | "$\geq$" |
| $f \in \omega(g)$ | $g \in o(f)$ | $\lim f/g = \infty$ | "$>$" |
| $f \in \Theta(g)$ | $f \in O(g) \land f \in \Omega(g)$ | $\lim f/g \in (0, \infty)$ | "$=$" |
