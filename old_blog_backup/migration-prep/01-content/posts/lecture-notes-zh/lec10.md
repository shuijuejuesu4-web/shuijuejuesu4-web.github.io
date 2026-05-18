---
title: "MIT 6.1200J: 第10讲 — 密码学"
date: 2026-05-15
categories: [MIT 6.1200J, 笔记]
tags: [离散数学, 密码学, RSA, Diffie-Hellman, 中国剩余定理]
mathjax: true
---

# 第10讲：密码学（Cryptography）

## 1 密码学

为什么数论在 6.1200 课程中？今天我们将看到 GCD 和模运算对计算机安全有多么重要！

::: tip

前面学的 GCD、模运算、费马小定理，全是这一讲的铺垫。RSA 加密算法把这些数学工具组合了起来，保护了互联网上几乎所有的安全通信。当你学完这一讲，你就真正理解了 HTTPS 锁图标背后的数学原理。
:::

密码学（Cryptography）是保护信息的技术与科学。基本思想是加密（Encrypt）消息，使只有特定方能够阅读；预期的接收者则能够解密（Decrypt）以恢复原始消息。

惯例：**Alicekazam** 向 **Bobasaur** 发送加密消息，Eevee 是窃听者，能听到一切但希望无法理解。

## 2 历史（不考）

### Caesar 密码
将每个字母向后移动 3 位（模 26）。例如 CRYPTO → FUBSWR。隐匿式安全（Security by Obscurity）：只要 Eevee 不知道你用了它就行。

### Caesar 移位
Alice 和 Bob 事先约定秘密移位值 $k$。Eevee 不知道 $k$，但我们安全吗？容易受到暴力攻击（Brute Force）：Eevee 只需尝试全部 26 种选项。

### 替换密码（Substitution Cipher）
将每个字母映射到不同字母。共有 $26!$ 种可能的密钥。容易受到频率分析（Frequency Analysis）攻击。

### 德国 Enigma
二战德国 Enigma 机根据消息本身和秘密初始配置改变移位值。大约 $3 \cdot 10^{114}$ 种设置。傲慢式安全（Security by Hubris）：开发 Enigma 的工程师不知道如何破解它，就假设没人能破解。但盟军破解了密码。

### 一次性密码本（One-Time Pad）
Caesar 移位但用更大的数。消息编码为 0 到 $n-1$ 之间的数。Alice 发送 $\text{enc}(m, k) = \text{rem}(m+k, n)$，Bob 计算 $m = \text{rem}(\text{enc} - k, n)$。如果 $k$ 均匀随机选择，则 $m+k$ 也是均匀随机的，不泄露任何信息。**但不能重用 $k$**（已知明文攻击和消息关联性分析），因此称为「一次性」密码本。

### Diffie-Hellman 密钥交换
Alice 和 Bob 通过公开信道协商秘密值 $k$，Eevee 无法发现。

1. 选择一个大素数 $n$（数百位）
2. 选择基数 $c$（$1 < c < n-1$）并公开
3. Alice 选择随机数 $a$，发送 $c^a \bmod n$ 给 Bob
4. Bob 选择随机数 $b$，发送 $c^b \bmod n$ 给 Alice
5. Alice 计算 $x := (c^b)^a \bmod n = c^{ab} \bmod n$
6. Bob 计算 $y := (c^a)^b \bmod n = c^{ab} \bmod n = x$

这是共享密钥。Eevee 知道 $c, c^a, c^b$，但从 $c$ 和 $c^a \bmod n$ 恢复 $a$ 是离散对数问题（Discrete Log Problem），目前没有计算上可行的方法。

## 3 RSA

RSA = **Rivest, Shamir, Adleman（MIT 发明者，2002 年 Turing 奖）。公钥密码系统（Public-Key Cryptosystem）**：可以公开告诉所有人加密密钥，但它只允许加密，不能解密。

**密钥生成**：
1. 选择两个大素数 $p, q$。保密 $p, q$，但公开 $n := pq$
2. 选择一个大数 $e$，与 $(p-1)(q-1)$ 互素。公钥：$k_p := (n, e)$
3. 计算 $d$，它是 $e$ 模 $(p-1)(q-1)$ 的乘法逆元（用 Pulverizer）。私钥：$k_s := (n, d)$

**加密**：$E(m, k_p) := \text{rem}(m^e, n)$

**解密**：$D(c, k_s) := \text{rem}(c^d, n)$

解密正确性：$m^{ed} \equiv_n m$。（证明：$ed = 1 + t(p-1)(q-1)$，由费马小定理，模 $p$ 下 $m^{ed} \equiv_p m \cdot (m^{p-1})^{t(q-1)} \equiv_p m$，模 $q$ 下同理，由中国剩余定理得模 $pq$ 下也成立。）

### 3.1 安全性
依赖于**大整数分解困难性**。公开信息 $n = pq$ 和 $e$。私密信息 $p, q, d$。目前没有已知的高效方法将 $n$ 分解为两个素数因子。

### 3.2 寻找大素数
1. 用 Miller-Rabin 算法可以高效测试素数
2. 根据**素数定理**：$\pi(k) \sim k / \ln k$。$k = 10^{300}$ 附近，素数密度约为 $1/700$，期望只需尝试约 700 次！

## 4 中国剩余定理（Chinese Remainder Theorem, CRT）

定理 1（CRT）：假设 $p$ 和 $q$ 互素，$a, b \in \mathbb{Z}$。则在模 $pq$ 下，方程组存在唯一解 $x$：

$$\begin{cases} x \equiv_p a \\ x \equiv_q b \end{cases}$$

**存在性证明**：定义 $p^{-1}$ 为 $p$ 模 $q$ 的逆元，$e_q := p^{-1}p$。则 $e_q \equiv_p 0$ 且 $e_q \equiv_q 1$。类似地定义 $q^{-1}$ 为 $q$ 模 $p$ 的逆元，$e_p := q^{-1}q$。则 $x := ae_p + be_q$ 满足两个同余式。

**唯一性证明**：假设 $x$ 和 $x'$ 都满足，则 $p \mid (x-x')$ 且 $q \mid (x-x')$。由于 $\gcd(p, q) = 1$，有 $pq \mid (x-x')$，即 $x \equiv_{pq} x'$。
