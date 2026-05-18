---
title: "MIT 6.1200J: 第15讲 — 关系与计数"
date: 2026-05-15
categories: [MIT 6.1200J, 笔记]
tags: [离散数学, 关系, 等价关系, 偏序, 计数, 乘法规则, 双射规则]
mathjax: true
---

# 第15讲：关系与计数（Relations and Counting）

在之前的几讲中我们学习了图论。现在我们将进入两个新主题：**关系**和**计数**。关系是数学中描述对象之间关联的基本工具，而计数（组合数学）则是概率论和算法分析的基础。这两个主题虽然看起来不同，但通过双射规则紧密相连。

## 1 关系（Relations）

**定义 1**：一个关系（relation） $R \subseteq A \times B$ 包含：
- 一个定义域（domain） $A$（可以是任意集合），
- 一个陪域（codomain） $B$（可以是任意集合），
- 以及一个有序对的子集 $R \subseteq A \times B$。

关系推广了从 $A$ 到 $B$ 的函数概念。

::: tip

如果你写过 SQL，你就已经用过关系了。数据库中的表本质上就是关系（relation），`JOIN` 操作就是关系的组合。等价关系则和哈希表、并查集（Union-Find）等数据结构密切相关。这一讲的数学概念直接映射到你每天用的数据库系统。
:::

记号约定：我们通常写 $a R b$ 表示 $(a, b) \in R$。也可以将 $R$ 视为谓词，写 $R(a, b)$ 表示 $(a, b) \in R$。当 $(a, b) \in R$ 时，我们说 $a$ 在 $R$ 中关联到 $b$，但注意方向是重要的——我们将 $(a, b) \in R$ 画成从 $a \in A$ 到 $b \in B$ 的有向箭头。

值得一提的是，二元关系正是关系数据库中"关系"一词的由来——它只是一组有序对，某些对在其中，某些不在。

函数是一个重要的例子：

**定义 2**：关系 $R \subseteq A \times B$ 是一个函数（function），如果每个 $a \in A$ 关联到至多一个 $b \in B$。此时记为 $R : A \to B$。每个 $a \in A$ "至多 1 条出边"。

当 $R$ 是函数时，我们可以用 $R(a)$ 表示它所关联到的那个唯一元素 $b$（如果存在）。

**定义 3**：关系 $R \subseteq A \times B$ 是**全（total）的**，如果每个 $a \in A$ 关联到至少一个 $b \in B$："至少 1 条出边"。

这两者经常一起出现：一个全函数（total function） $f$ 在每个 $a \in A$ 恰有一条出边。因此 $f(a)$ 对所有输入都存在，且无歧义。

关于入边的类似术语：

**定义 4**：关系 $R \subseteq A \times B$ 是**单射（injective）的**，如果每个 $b \in B$ 至多有 1 个 $a$ 满足 $a R b$。"至多 1 条入边"。

$R$ 是**满射（surjective）的**，如果每个 $b \in B$ 至少有 1 个 $a$ 满足 $a R b$。"至少 1 条入边"。

这些性质对比较集合的大小很有用：

**定理 1**：若 $A$ 和 $B$ 是有限集，且 $R \subseteq A \times B$ 是全单射，则 $|A| \leq |B|$。

**定理 2**：若 $A$ 和 $B$ 是有限集，且 $R \subseteq A \times B$ 是满射函数，则 $|A| \geq |B|$。

**定义 5**：既是单射又是满射的全函数称为双射（bijection）。

**定理 3**：若 $A$ 和 $B$ 是有限集，且 $R \subseteq A \times B$ 是双射，则 $|A| = |B|$。

## 2 单个集合上的关系

我们从未说过 $A$ 和 $B$ 必须不相交甚至不同！很多有用的例子来自 $A = B$ 的情况。关系 $R \subseteq A \times A$ 称为 $A$ 上的二元关系（binary relation）。

这个定义与有向图的定义完全相同。$a R b$ 意味着图中有向边 $(a, b)$。熟悉的例子：$a = b$、$a \equiv b \pmod{10}$、$a \leq b$、$A \subseteq B$、$a \mid b$。

如果 $G$ 是有向图，我们考察其漫步关系（walk relation），即可达关系（reachability relation） $G^*$，其中 $a G^* b$ 当且仅当存在从 $a$ 到 $b$ 的漫步。还有强连通关系（strong connectivity relation） $S$，其中 $a S b$ 当且仅当 $a G^* b$ 且 $b G^* a$。

### 2.1 等价关系（Equivalence Relations）

想要刻画像"$=$"一样的行为，表示"相同性"或"等价性"。

**定义 6**：设 $R \subseteq A \times A$ 是 $A$ 上的关系。
- $R$ 是自反的（reflexive）：对所有 $a \in A$，$a R a$。
- $R$ 是对称的（symmetric）：对所有 $a, b \in A$，$a R b$ 当且仅当 $b R a$。
- $R$ 是传递的（transitive）：对所有 $a, b, c \in A$，$a R b$ 且 $b R c$ 蕴含 $a R c$。
- $R$ 是等价关系（equivalence relation），如果 $R$ 是自反的、对称的且传递的。

**定理 4**：如果 $R$ 是 $A$ 上的等价关系，则 $R$ 将 $A$ 划分为称为等价类（equivalence classes）的子集，其中每个 $a \in A$ 恰好属于一个等价类，且 $a R b$ 为真当且仅当 $a$ 和 $b$ 属于同一个等价类。

### 2.2 弱偏序（Weak Partial Orders）

想要刻画像"$\leq$"一样的行为，表示"排序"。

**定义 7**：设 $R \subseteq A \times A$ 是 $A$ 上的关系。
- $R$ 是反对称的（antisymmetric）：对所有 $a, b \in A$，若 $a R b$ 且 $b R a$ 同时为真，则 $a = b$。
- $R$ 是弱偏序（weak partial order, WPO），如果 $R$ 是自反的、反对称的且传递的。

**定理 5**：如果 $G$ 是有向图，则 $G$ 上的漫步关系是 WPO 当且仅当 $G$ 是 DAG。

**定义 8**：对于 WPO $R \subset A \times A$，两个元素 $a, b$ 称为可比较的（comparable），当 $a R b$ 或 $b R a$。

一个 WPO 称为线性序（linear ordering），亦称全序（total ordering），如果每对元素都是可比较的。

$a \leq b$ 是全序，但 $a \mid b$ 和 $A \subseteq B$ 不是。

## 3 计数（Counting）

计数（Counting）不是数 1, 2, 3，而是求集合的大小。有多少种洗牌方式？答案：$52!$。有多少棵节点为 $\{1, 2, \ldots, n\}$ 的树？令人惊讶的答案：$n^{n-2}$。

在分析算法时很有用；可以用计数技巧证明运行时间界。对概率也很有用。

### 3.1 乘法规则（Product Rule）

对有限集 $A, B$，有 $|A \times B| = |A| \cdot |B|$。更一般地：

$$|A_1 \times \cdots \times A_n| = |A_1| \cdot \cdots \cdot |A_n|.$$

示例：长度为 $n$ 的二进制序列数是 $2^n$。这个集合恰为 $B^n := \{0, 1\} \times \cdots \times \{0, 1\} = \{0, 1\}^n$，故其大小为 $2 \cdot \cdots \cdot 2 = 2^n$。

### 3.2 双射规则（Bijection Rule）

如果 $A$ 和 $B$ 之间存在双射，则 $|A| = |B|$。

示例：$\{1, 2, \ldots, n\}$ 的子集数为 $2^n$。令 $P_n := \{A \mid A \subseteq \{1, 2, \ldots, n\}\}$。我们可以给出从 $P_n$ 到 $B^n$ 的双射：$A \in P_n$ 映射到 $n$ 位串，其中第 $i$ 位为 1 如果 $i \in A$，否则为 0。

### 3.3 加法规则（Sum Rule）

加法规则：如果 $A_1, \ldots, A_n$ 是两两不相交的有限集，则

$$|A_1 \cup \cdots \cup A_n| = |A_1| + \cdots + |A_n|.$$

示例：有多少密码长度在 6 到 8 之间，以大写字母开头，其余字符为大写字母、小写字母或数字？$W$ 是 $W_6, W_7, W_8$ 的不交并，其中 $W_k$ 统计长度为 $k$ 且满足这些约束的密码数。由乘法规则，$W_k = 26 \cdot 62^{k-1}$。故 $|W| = 26 \cdot 62^5 + 26 \cdot 62^6 + 26 \cdot 62^7$。

对于加法规则和乘法规则，我们通常在做多种选择。如果是这些选择的 OR，用加法规则。如果是 AND，用乘法规则。

### 3.4 广义乘法规则（Generalized Product Rule）

一副洗好的牌有多少种可能的排列？52 种不同的牌。

第一张牌有 52 种选择。选定后，第二张牌有 51 种选择。然后 50，然后 49，依此类推。答案是 $52 \cdot 51 \cdot 50 \cdots 1 = 52! \approx 8 \cdot 10^{67}$。

这并非乘法规则：第二张牌的选择集合根据第一张牌的选择而改变！重要的是，选择的数量是一致的，**无论之前做了何种选择**。

一般地，如果 $A$ 是长度为 $k$ 的序列 $(a_1, \ldots, a_k)$ 的集合，其中 $a_1$ 有 $n_1$ 种选择，无论 $a_1$ 取何值 $a_2$ 都有 $n_2$ 种选择，无论 $a_1$ 和 $a_2$ 取何值 $a_3$ 都有 $n_3$ 种选择，依此类推直到 $a_k$，则 $|A| = n_1 \cdot n_2 \cdots n_k$。
