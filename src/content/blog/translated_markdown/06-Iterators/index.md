---
title: "第6章：迭代器与指针 (Iterators and Pointers)"
description: "第6章：迭代器与指针 (Iterators and Pointers)"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第6章：迭代器与指针 (Iterators and Pointers)

> Stanford CS106L, Fall 2025 -- Rachel Fernandez, Thomas Poimenidis

---

## 目录 (Table of Contents)

1. [迭代器基础 (Iterator Basics)](#1-迭代器基础)
2. [迭代器类型 (Iterator Types)](#2-迭代器类型)
3. [指针与内存 (Pointers and Memory)](#3-指针与内存)
4. [本章回顾 (Recap)](#4-本章回顾)
5. [补充知识点](#5-补充知识点)
6. [常用API参考](#6-常用api参考)

---

## 1. 迭代器基础

### 1.1 为什么需要迭代器？

在前面的课程中，我们学习了各种STL容器。无论容器如何存储数据，我们都可以用相同的方式遍历它们：

```cpp
for (const auto& elem : container)
```

但这是如何实现的呢？考虑以下问题：我们如何以统一的方式遍历不同类型的数据结构？

对于 `std::vector`，我们可以使用索引：

```cpp
std::vector<int> v {1, 2, 3, 4};
for (size_t i = 0; i < v.size(); i++) {
    const auto& elem = v[i];
    std::cout << elem;
}
```

但对于 `std::set`，索引访问不可用：

```cpp
std::set<int> s {1, 2, 3, 4};
// for (size_t i = 0; i < s.size(); i++) -- 不能这样!
```

我们需要一种机制来**跟踪我们在容器中的位置** -- 类似于索引，但适用于所有容器。这就是迭代器的诞生背景。

### 1.2 迭代器的类比：抓娃娃机

C++ 迭代器的工作原理类似于抓娃娃机：

- **爪子 (claw)** 可以：抓取玩具、向前移动、检查是否完成
- **机器 (machine)** 可以：告诉我们从哪里开始、何时停止

在C++中：
- **迭代器** 扮演爪子的角色，负责移动和访问元素
- **容器** 扮演机器的角色，提供开始和结束位置

### 1.3 容器接口

每个STL容器提供两个关键方法：

```cpp
// 获取指向第一个元素的迭代器（假设容器非空）
container.begin()

// 获取"越界"迭代器 -- 指向最后一个元素之后的位置
container.end()
```

**重要提示：`end()` 永远不指向实际元素！** 它指向容器末尾之后的一个位置。如果容器为空，则 `begin() == end()`。

### 1.4 迭代器接口

迭代器提供四个基本操作：

```cpp
// 1. 拷贝构造 -- 获取迭代器
auto it = c.begin();

// 2. 向前移动迭代器
++it;

// 3. 解引用 -- 获取元素（如果 it == end() 则为未定义行为）
auto& elem = *it;

// 4. 相等比较 -- 我们在同一个位置吗？
if (it == c.end()) ...
```

有了这些操作，我们可以手动遍历任何容器：

```cpp
std::set<int> s {1, 2, 3, 4};
for (auto it = s.begin(); it != s.end(); ++it) {
    const auto& elem = *it;
    std::cout << elem;
}
```

### 1.5 范围for循环的底层实现

当你写范围for循环时，编译器实际上将其转换为迭代器形式：

```cpp
// 你写的代码：
for (auto elem : s) {
    std::cout << elem;
}

// 等价于编译器生成的代码：
auto b = s.begin();
auto e = s.end();
for (auto it = b; it != e; ++it) {
    auto elem = *it;
    std::cout << elem;
}
```

### 1.6 迭代器类型与 `auto`

迭代器类型通常很冗长，因此我们惯用 `auto`：

```cpp
std::map<int, int> m { {1, 2}, {3, 4}, {5, 6} };
auto it = m.begin();  // 避免写出 std::map<int, int>::iterator
```

每个容器内部使用 `using` 定义了其迭代器类型的别名：

```cpp
template <typename K, typename V>
class std::map {
    using iterator = /* 某种迭代器类型 */;
};
```

### 1.7 为什么用 `++it` 而非 `it++`？

```cpp
// 前缀形式 - ++it
// 递增 it 并返回同一对象的引用
Iterator& operator++();

// 后缀形式 - it++
// 递增 it 并返回旧值的副本
Iterator operator++(int);
```

迭代器是一个完整的对象，拷贝它（后缀形式）通常比拷贝 `int` 更昂贵。Bjarne Stroustrup 的建议：

> "++i 有时比 i++ 快，且从不比 i++ 慢。如果你将 i++ 作为语句而非表达式的一部分来写，为什么不直接写成 ++i 呢？你不会有任何损失，有时还能有所收获。"

---

## 2. 迭代器类型

### 2.1 迭代器类型层次结构

并非所有迭代器都是平等的。C++定义了五种迭代器类别，按功能递增排列：

| 迭代器类型 | 功能 | 示例容器 |
|-----------|------|---------|
| **Input（输入）** | 读取元素，单遍扫描 | `istream_iterator` |
| **Output（输出）** | 写入元素 | `ostream_iterator` |
| **Forward（前向）** | 读取元素，多遍扫描 | `std::forward_list` |
| **Bidirectional（双向）** | 前进 + 后退 | `std::map`, `std::set` |
| **Random Access（随机访问）** | 快速跳转任意步数 | `std::vector`, `std::deque` |

### 2.2 输入迭代器 (Input Iterator)

最基本的迭代器类型，只允许读取元素：

```cpp
auto elem = *it;  // 读取元素
```

如果元素是结构体，可以用 `->` 访问成员：

```cpp
struct Bibble {
    int zarf;
};

std::vector<Bibble> v {...};
auto it = v.begin();
int m = (*it).zarf;   // 先解引用，再访问成员
int m = it->zarf;     // 完全等价！更简洁
```

### 2.3 输出迭代器 (Output Iterator)

允许写入元素：

```cpp
*it = elem;  // 写入元素
```

### 2.4 前向迭代器 (Forward Iterator)

输入迭代器 + 多遍扫描保证。所有STL容器迭代器都属于此类别。

**多遍扫描保证 (Multi-pass Guarantee)**：如果 `it1 == it2`，那么 `++it1 == ++it2`。

什么数据结构不支持多遍扫描？**流 (Streams)** -- 你不能两次读取同一段输入流数据。

### 2.5 双向迭代器 (Bidirectional Iterator)

允许向前和向后移动：

```cpp
auto it = m.end();
--it;                 // 获取最后一个元素
auto& elem = *it;
```

`std::map` 和 `std::set` 提供双向迭代器。

### 2.6 随机访问迭代器 (Random Access Iterator)

允许快速跳过任意步数：

```cpp
auto it2 = it + 5;    // 前进5步
auto it3 = it2 - 2;   // 后退2步
auto& second = *(it + 2);   // 获取第3个元素
auto& second = it[2];       // 等价写法
```

`std::vector` 和 `std::deque` 提供随机访问迭代器。

**注意：不要越界！**

```cpp
std::vector<int> v { 1, 2, 3 };
auto it = v.begin();
it += 3;              // 现在 it 等于 v.end()
int& elem = *it;      // 未定义行为！
```

### 2.7 为什么要区分迭代器类型？

- **目标**：为所有容器提供统一的抽象
- **限制**：容器的实现方式影响迭代方式
  - 在序列容器（vector、deque）中进行随机访问（跳5步）比在关联容器（map、set）中容易/快速得多
  - C++默认避免提供慢速方法，因此你不能对 `map::iterator` 进行随机访问

不同算法的需求：

```cpp
std::vector<int> vec{1, 5, 3, 4};
std::sort(vec.begin(), vec.end());
// ✅ begin/end 是随机访问迭代器

std::unordered_set<int> set {1, 5, 3, 4};
std::sort(set.begin(), set.end());
// ❌ begin/end 是双向迭代器，sort 需要随机访问
```

### 2.8 各容器迭代器类型速查表

```cpp
// 随机访问迭代器
std::vector<T>::iterator          // 概念上类似于 T*（但不保证）
std::deque<T>::iterator

// 双向迭代器
std::map<K,V>::iterator
std::set<T>::iterator

// 前向迭代器
std::unordered_map<K,V>::iterator
std::unordered_set<T>::iterator
std::forward_list<T>::iterator

// 输入/输出迭代器
std::istream_iterator<T>          // 输入
std::ostream_iterator<T>          // 输出
```

---

## 3. 指针与内存

### 3.1 迭代器 vs 指针

- **迭代器** 指向容器元素
- **指针** 可以指向任意对象

### 3.2 内存基础

- 每个变量都存在于内存中的某个位置
- 所有可能的位置构成地址空间 (address space)
- 内存通常是按字节寻址的，每个字节从0开始编号
- 1字节 = 8位

在64位系统上，地址范围从 `0x0` 到 `2^64 - 1`。

**内存布局**：

```
      高地址 (2^64 - 1)
  +------------------+
  |   OS Shared      |
  +------------------+
  |  Stack (栈变量)   |
  +------------------+
  |                   |
  |  Heap (堆变量)    |
  +------------------+
  |  Global Variables |
  +------------------+
  |  Text (指令)      |
  +------------------+
      低地址 (0x0)
```

- 对象的地址是其最低字节的位置
- 例如，在大多数现代平台上，`int` 占用 32 位 = 4 字节：

```
int x = 106;  // 32位

  x's memory
0x10:  00000000
0x11:  00000000
0x12:  00000000
0x13:  01101010  (= 106)
```

### 3.3 指针基础

指针就是变量的地址：

```cpp
int x = 106;
int* px = &x;  // int* 表示 px 是指向 int 的指针
               // & 是取地址运算符

std::cout << x << std::endl;    // 106
std::cout << *px << std::endl;  // 106 (解引用)
std::cout << px << std::endl;   // 0x50527c (地址值)
```

指针本质上就是一个数字（地址值）。

### 3.4 指向各种类型的指针

```cpp
int x = 106;
int* px = &x;

StanfordID id { "jtrb" };
StanfordID* p = &id;
auto name = p->name;           // 通过指针访问成员

std::vector<int> v;
std::vector<int>* p = &v;      // 指向vector的指针

std::vector<int> v { 1, 2, 3, 4, 5 };
int* arr = &v[0];              // 指向vector内部数组的指针
```

### 3.5 数组指针与迭代器的相似性

用指针操作vector的内部数组：

```cpp
std::vector<int> v {1, 2, 3, 4, 5};

int* arr = &v[0];
std::cout << *arr << " ";       // 1
arr += 1;
std::cout << *arr << " ";       // 2
++arr;
std::cout << *arr << " ";       // 3
arr += 2;
std::cout << *arr << " ";       // 5
if (arr == &v[4])
    std::cout << "At last index";

// 输出: 1 2 3 5 At last index
```

注意指针操作与迭代器操作的对应关系：

| 指针操作 | 迭代器操作 | 含义 |
|---------|-----------|------|
| `int* arr = &v[0]` | `auto it = v.begin()` | 拷贝构造 |
| `arr += 1` | `it += 1` | 随机访问 |
| `++arr` | `++it` | 向前移动 |
| `arr += 2` | `it += 2` | 随机访问 |
| `arr == &v[4]` | `it == --v.end()` | 比较 |

用迭代器做同样的事：

```cpp
auto it = v.begin();
std::cout << *it << " ";              // 1
it += 1;
std::cout << *it << " ";              // 2
++it;
std::cout << *it << " ";              // 3
it += 2;
std::cout << *it << " ";              // 5
if (it == --v.end())
    std::cout << "At last element";
```

### 3.6 vector迭代器的底层实现

对于 `std::vector`，其迭代器本质上可以理解为 `T*`：

```cpp
template <typename T>
class vector {
    using iterator = T*;  // 概念上如此（实际STL实现更复杂）
};
```

迭代器和指针拥有**相同的接口** -- 这是STL设计的精妙之处！

---

## 4. 本章回顾

| 概念 | 要点 |
|------|------|
| **迭代器基础** | 迭代器允许我们向前遍历容器；容器提供 `begin()` 和 `end()`；迭代器支持 `++it`、`*it`、`it == c.end()` |
| **迭代器类型** | 五种类型：Input、Output、Forward、Bidirectional、Random Access |
| **指针与内存** | 指针指向内存中的任意C++对象；指针和迭代器拥有相同的接口 |

---

## 5. 补充知识点

### 5.1 范围for循环的高级用法

C++20引入了初始化语句在范围for循环中：

```cpp
// C++20: 在for循环中初始化额外变量
for (auto vec = getVector(); const auto& elem : vec) {
    // 使用 elem
}
```

### 5.2 反向迭代器 (Reverse Iterators)

大多数容器也提供反向迭代：

```cpp
std::vector<int> v {1, 2, 3, 4, 5};

// rbegin() 指向最后一个元素
// rend() 指向第一个元素之前
for (auto it = v.rbegin(); it != v.rend(); ++it) {
    std::cout << *it << " ";  // 输出: 5 4 3 2 1
}
```

反向迭代器的类型：`std::vector<T>::reverse_iterator`

### 5.3 迭代器失效 (Iterator Invalidation)

修改容器可能导致迭代器失效。这是C++编程中常见的陷阱：

```cpp
std::vector<int> v {1, 2, 3, 4, 5};
auto it = v.begin();  // 指向第一个元素

v.push_back(6);       // 可能导致重新分配！
// it 现在可能失效 -- 使用 *it 是未定义行为！
```

**常见失效规则**：

| 操作 | vector | deque | map/set | list |
|------|--------|-------|---------|------|
| `push_back` | 如果重新分配则失效 | 所有迭代器失效 | 不失效 | 不失效 |
| `insert` | 插入点及之后失效 | 所有迭代器失效 | 不失效 | 不失效 |
| `erase` | 擦除点及之后失效 | 可能所有失效 | 仅被擦除元素失效 | 仅被擦除元素失效 |

### 5.4 迭代器适配器 (Iterator Adapters)

C++提供了一些特殊的迭代器适配器：

```cpp
// 1. 插入迭代器 -- 将赋值转换为插入
std::vector<int> src {1, 2, 3};
std::vector<int> dst;
std::copy(src.begin(), src.end(), std::back_inserter(dst));
// dst = {1, 2, 3}

// 2. 流迭代器 -- 从输入流读取/写入输出流
std::vector<int> v;
std::copy(
    std::istream_iterator<int>(std::cin),
    std::istream_iterator<int>(),
    std::back_inserter(v)
);

// 3. 移动迭代器 -- 将拷贝转换为移动
std::vector<std::string> src {"hello", "world"};
std::vector<std::string> dst(
    std::make_move_iterator(src.begin()),
    std::make_move_iterator(src.end())
);
```

### 5.5 `std::advance` 与 `std::distance`

这两个工具函数可以统一处理不同类型的迭代器：

```cpp
// std::advance: 向前移动任意步数（自动选择最优方式）
std::list<int> l {1, 2, 3, 4, 5};
auto it = l.begin();
std::advance(it, 3);  // 对于list，等价于三次 ++it
                      // 对于vector，等价于 it += 3

// std::distance: 计算两个迭代器之间的距离
auto dist = std::distance(l.begin(), l.end());  // 5
```

### 5.6 自定义迭代器

你可以为自己的容器实现迭代器。最小的前向迭代器只需要：

```cpp
class MyIterator {
public:
    // 必须提供的类型别名
    using value_type = T;
    using reference = T&;
    using pointer = T*;
    using difference_type = std::ptrdiff_t;
    using iterator_category = std::forward_iterator_tag;

    reference operator*() const;       // 解引用
    MyIterator& operator++();          // 前缀递增
    MyIterator operator++(int);        // 后缀递增
    bool operator==(const MyIterator&) const;  // 相等比较
    bool operator!=(const MyIterator&) const;  // 不等比较
};
```

---

## 6. 常用API参考

### 6.1 迭代器获取

| API | 描述 | 示例 |
|-----|------|------|
| `c.begin()` | 获取指向第一个元素的迭代器 | `auto it = v.begin();` |
| `c.end()` | 获取越界迭代器 | `if (it != v.end())` |
| `c.rbegin()` | 获取反向迭代器（指向最后一个元素） | `auto rit = v.rbegin();` |
| `c.rend()` | 获取反向越界迭代器 | `for (; rit != v.rend(); ++rit)` |
| `c.cbegin()` | 获取const迭代器（C++11） | `auto it = v.cbegin();` |
| `c.cend()` | 获取const越界迭代器（C++11） | `auto e = v.cend();` |

### 6.2 迭代器操作

| 操作 | 描述 | 适用类型 | 示例 |
|------|------|---------|------|
| `*it` | 解引用，获取元素引用 | 所有（除Output） | `auto& x = *it;` |
| `it->member` | 访问成员 | Input及以上 | `auto n = it->name;` |
| `++it` | 向前移动（前缀） | Forward及以上 | `++it;` |
| `--it` | 向后移动 | Bidirectional及以上 | `--it;` |
| `it + n` | 向前跳n步 | Random Access | `auto it2 = it + 3;` |
| `it - n` | 向后跳n步 | Random Access | `auto it2 = it - 2;` |
| `it[n]` | 等价于 `*(it + n)` | Random Access | `auto x = it[2];` |
| `it1 - it2` | 返回两个迭代器间的距离 | Random Access | `auto d = e - b;` |
| `it1 < it2` | 比较位置先后 | Random Access | `if (it1 < it2)` |

### 6.3 工具函数 (`<iterator>` 头文件)

| 函数 | 描述 | 示例 |
|------|------|------|
| `std::advance(it, n)` | 将迭代器前进n步（自动选择最优方式） | `std::advance(it, 5);` |
| `std::distance(it1, it2)` | 返回两迭代器间的距离 | `auto d = std::distance(b, e);` |
| `std::next(it, n=1)` | 返回前进n步后的迭代器（不修改原迭代器） | `auto it2 = std::next(it, 3);` |
| `std::prev(it, n=1)` | 返回后退n步后的迭代器（Bidirectional） | `auto last = std::prev(e);` |
| `std::begin(c)` | 自由函数版begin（支持数组） | `auto it = std::begin(arr);` |
| `std::end(c)` | 自由函数版end（支持数组） | `auto e = std::end(arr);` |
| `std::iter_swap(it1, it2)` | 交换两个迭代器指向的元素 | `std::iter_swap(it1, it2);` |

### 6.4 迭代器适配器

| 适配器 | 描述 | 示例 |
|--------|------|------|
| `std::back_inserter(c)` | 将赋值转换为 `push_back` | `std::copy(b,e,std::back_inserter(v));` |
| `std::front_inserter(c)` | 将赋值转换为 `push_front` | `std::copy(b,e,std::front_inserter(l));` |
| `std::inserter(c, pos)` | 将赋值转换为 `insert` | `std::copy(b,e,std::inserter(s,s.begin()));` |
| `std::istream_iterator<T>(stream)` | 从输入流读取 | 见上文5.4节 |
| `std::ostream_iterator<T>(stream, delim)` | 写入输出流，可选分隔符 | `std::copy(b,e,std::ostream_iterator<int>(cout," "));` |
| `std::make_move_iterator(it)` | 解引用时移动而非拷贝 | 见上文5.4节 |

### 6.5 指针与地址操作

| 操作 | 描述 | 示例 |
|------|------|------|
| `&var` | 取变量的地址 | `int* p = &x;` |
| `*ptr` | 解引用指针 | `int val = *p;` |
| `ptr->member` | 通过指针访问成员 | `auto n = p->name;` |
| `nullptr` | 空指针常量（C++11，替代NULL） | `int* p = nullptr;` |
| `sizeof(type)` | 获取类型大小（字节） | `sizeof(int)` 通常为 4 |
| `sizeof(var)` | 获取变量大小 | `sizeof(x)` |
