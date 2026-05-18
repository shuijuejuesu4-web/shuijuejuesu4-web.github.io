# 第5章：容器 (Containers)

> **授课教师**：Rachel Fernandez, Thomas Poimenidis
> **学期**：Stanford CS106L, Fall 2025

---

## 目录

- [5.1 什么是 STL？什么是模板？](#51-什么是-stl什么是模板)
- [5.2 顺序容器 (Sequence Containers)](#52-顺序容器-sequence-containers)
  - [5.2.1 std::vector](#521-stdvector)
  - [5.2.2 std::deque](#522-stddeque)
- [5.3 关联容器 (Associative Containers)](#53-关联容器-associative-containers)
  - [5.3.1 std::map](#531-stdmap)
  - [5.3.2 std::set](#532-stdset)
  - [5.3.3 std::unordered_map 和 std::unordered_set](#533-stdunordered_map-和-stdunordered_set)
- [5.4 容器选择指南](#54-容器选择指南)
- [📚 补充知识点](#-补充知识点)
- [🔧 常用API参考](#-常用api参考)

---

## 5.1 什么是 STL？什么是模板？

### 5.1.1 模板的动机

在没有模板之前，如果你想存储不同类型的列表，需要为每种类型写一个单独的类：

```cpp
class IntVector {
    // 存储整数列表的代码...
};
class DoubleVector {
    // 存储双精度浮点数列表的代码...
};
class StringVector {
    // 存储字符串列表的代码...
};
```

这种方式导致大量重复代码。模板（Templates）解决了这个问题：

```cpp
template <typename T>
class vector {
    // 存储类型为 T 的列表的代码 —— 一次编写，处处使用！
};

vector<int> v1;       // 整数向量
vector<double> v2;    // 双精度浮点数向量
vector<string> v3;    // 字符串向量
```

### 5.1.2 STL：标准模板库

STL（Standard Template Library）由 Alexander Stepanov 创建，他将模板引入 C++ 并构建了著名的标准库。

STL 包含四大组件：

| 组件 | 英文 | 作用 |
|------|------|------|
| 容器 | Containers | 如何存储一组数据？ |
| 迭代器 | Iterators | 如何遍历容器？ |
| 函数对象 | Functors | 如何将函数表示为对象？ |
| 算法 | Algorithms | 如何以通用方式转换和修改容器？ |

**所有 STL 容器都是模板！**

### 5.1.3 C++ 提供的各种容器

```
标准模板库 (STL) 容器：

顺序容器：                    关联容器：
  std::vector                   std::map
  std::deque                    std::set
  std::array                    std::unordered_map
  std::list                     std::unordered_set

容器适配器：
  std::stack
  std::queue
  std::priority_queue
```

---

## 5.2 顺序容器 (Sequence Containers)

顺序容器存储元素的线性序列。

### 5.2.1 std::vector

`#include <vector>`

```cpp
std::vector<int> vec { 1, 2, 3, 4 };
vec.push_back(5);
vec.push_back(6);
vec[1] = 20;

for (size_t i = 0; i < vec.size(); i++) {
    std::cout << vec[i] << " ";
}
// 输出: 1 20 3 4 5 6
```

#### Stanford Vector vs. STL vector

| 操作 | Stanford Vector\<int\> | std::vector\<int\> |
|------|------------------------|---------------------|
| 创建空向量 | `Vector<int> v;` | `std::vector<int> v;` |
| 创建 n 个 0 | `Vector<int> v(n);` | `std::vector<int> v(n);` |
| 创建 n 个值 k | `Vector<int> v(n, k);` | `std::vector<int> v(n, k);` |
| 末尾添加 | `v.add(k);` | `v.push_back(k);` |
| 清空 | `v.clear();` | `v.clear();` |
| 检查是否为空 | `if (v.isEmpty())` | `if (v.empty())` |
| 获取第 i 个元素 | `int k = v.get(i);` / `v[i]` | `int k = v.at(i);` / `v[i]` |
| 替换第 i 个元素 | `v.get(i) = k;` / `v[i] = k` | `v.at(i) = k;` / `v[i] = k` |

#### vector 的内部实现

vector 在内存中使用一段连续的数组，维护两个关键变量：

- **size**：当前存储的元素数量
- **capacity**：当前分配的内存可容纳的元素数量

```
 Index:  0     1     2     3     4     5     6     7
       +-----+-----+-----+-----+-----+-----+-----+-----+
       |  1  |  2  |  3  |  4  |     |     |     |     |
       +-----+-----+-----+-----+-----+-----+-----+-----+
                                  ↑                 ↑
                              size = 4        capacity = 8
```

当 `push_back` 导致 `size == capacity` 时，vector 会：
1. 分配一块更大的内存（通常是原来的 2 倍）
2. 将旧元素拷贝/移动到新内存
3. 释放旧内存

#### ⚠️ operator[] 不做边界检查

```cpp
std::vector<int> vec{5, 6};   // {5, 6}
vec[1] = 3;                   // {5, 3}  ← OK
vec[2] = 4;                   // 未定义行为！(undefined behavior)
vec.at(2) = 4;                // 运行时错误 (抛出 std::out_of_range)
```

#### 几点重要建议

**使用 range-based for 循环：**
```cpp
// 传统方式
for (size_t i = 0; i < vec.size(); i++) {
    std::cout << vec[i] << " ";
}

// 更好的方式：适用于所有可迭代容器
for (auto elem : vec) {
    std::cout << elem << " ";
}
```

**使用 const auto& 避免不必要的拷贝：**
```cpp
std::vector<MassiveType> vec { ... };

// 不好：每个元素都拷贝一次
for (auto elem : vec) { ... }

// 好：避免拷贝，且防止意外修改
for (const auto& elem : vec) { ... }
```

#### vector 的局限性

vector **没有** `push_front()` 方法。在头部插入需要移动所有元素，效率极低（O(n)）。

```cpp
// 假设的 push_front——极其低效！
// 需要将所有元素向右移动一位
```

### 5.2.2 std::deque

`#include <deque>`

**deque**（"deck"，双端队列）允许在两端高效插入/删除。

```cpp
#include <deque>

void receivePrice(std::deque<double>& prices, double price) {
    prices.push_front(price);    // 非常快！
    if (prices.size() > 10000)
        prices.pop_back();       // 移除最早的价格（也非常快！）
}
```

#### deque 与 vector 的接口对比

deque 拥有与 vector 完全相同的接口，**额外拥有**：
- `push_front()`
- `pop_front()`

#### deque 的内部实现

vector 的问题是只有一块连续内存。deque 将其拆分为多个独立分配的子数组（array of arrays）：

```
        +-----+-----+ +-----+-----+ +-----+-----+
        | 1 9 | 3 2 | | 1 9 | 7 3 | | 2 1 | 2 9 |
        +-----+-----+ +-----+-----+ +-----+-----+
              ↑             ↑             ↑
          子数组1       子数组2       子数组3
                        （独立分配）
```

---

## 5.3 关联容器 (Associative Containers)

关联容器通过**唯一键**来组织元素。

### 5.3.1 std::map

`#include <map>`

`std::map<K, V>` 将键映射到值，相当于 Python 中的字典（dictionary），有时也称为关联数组。

```cpp
std::map<std::string, int> map {
    { "Chris", 2 },
    { "CS106L", 42 },
    { "Keith", 14 },
    { "Nick", 51 },
    { "Sean", 35 },
};

int sean = map["Sean"];   // 35
map["Chris"] = 31;        // 覆盖原值
```

#### Stanford Map vs. STL map

| 操作 | Stanford Map\<K,V\> | std::map\<K,V\> |
|------|---------------------|------------------|
| 创建空 map | `Map<K,V> m;` | `std::map<K,V> m;` |
| 插入键值对 | `m.put(k, v);` / `m[k] = v;` | `m.insert({k, v});` / `m[k] = v;` |
| 删除键 | `m.remove(k);` | `m.erase(k);` |
| 检查键是否存在 | `m.containsKey(k)` | `m.count(k)` / `m.contains(k)` (C++20) |
| 检查是否为空 | `m.isEmpty()` | `m.empty()` |
| 访问/修改值 | `m[k]` | `m[k]`（若不存在则默认插入） |

#### ⚠️ operator[] 会自动插入默认值

```cpp
std::map<std::string, int> map;
map["Alex"] = 0;   // "Alex" 不存在时，先默认插入值为 0 的条目
```

#### map 的本质

`std::map<K, V>` 存储的是 `std::pair<const K, V>` 的集合。

注意键（K）是 **const** 的：你不能修改键，因为这可能破坏二叉搜索树的结构。

#### 遍历 map

**使用 range-based for 遍历键值对：**
```cpp
std::map<std::string, int> map;

// 方式一：使用 pair
for (auto kv : map) {
    std::string key = kv.first;
    int value = kv.second;
}

// 方式二：使用结构化绑定 (C++17) —— 更推荐
for (const auto& [key, value] : map) {
    // key 的类型是 const std::string&
    // value 的类型是 const int&
}
```

#### map 的内部实现

map 通常使用**红黑树**（一种自平衡二叉搜索树）实现：

```
                "CS106L" : 42
               /            \
       "Chris" : 31      "Nick" : 51
         /        \      /        \
   "Alex" : 0  "Keith":14     "Sean" : 35
```

- 查找：O(log n)
- 插入/删除：O(log n)

#### ⚠️ std::map 要求键类型有 operator<

```cpp
std::map<int, int> map1;            // OK —— int 有 operator<
std::map<std::ifstream, int> map2;  // 错误 —— ifstream 没有 operator<
```

### 5.3.2 std::set

`#include <set>`

`std::set` 存储唯一元素的集合。

```cpp
std::set<std::string> set {
    "CS106L!",
    "Keith",
    "Sean",
    "Nick",
    "Chris"
};
```

#### Stanford Set vs. STL set

| 操作 | Stanford Set\<T\> | std::set\<T\> |
|------|-------------------|---------------|
| 创建空 set | `Set<T> s;` | `std::set<T> s;` |
| 添加元素 | `s.add(k);` | `s.insert(k);` |
| 删除元素 | `s.remove(k);` | `s.erase(k);` |
| 检查元素是否存在 | `s.contains(k)` | `s.count(k)` / `s.contains(k)` (C++20) |
| 检查是否为空 | `s.isEmpty()` | `s.empty()` |

#### set 的本质

**std::set 就是一个没有值的 std::map** —— 两者的底层实现相同（红黑树），set 只存储键而不存储值。

set 同样使用红黑树实现，元素按键（即元素自身）排序。

### 5.3.3 std::unordered_map 和 std::unordered_set

`#include <unordered_map>` 和 `#include <unordered_set>`

#### unordered_map

可以看作 map 的优化版本，接口与 map 相同：

```cpp
std::unordered_map<std::string, int> map {
    { "Chris", 2 },
    { "Nick", 51 },
    { "Sean", 35 },
};

int sean = map["Sean"];   // 35
map["Chris"] = 31;
```

#### unordered_map 的内部实现 —— 哈希表

与 map 存储一棵树不同，unordered_map 存储 n 个"桶"（buckets），每个桶中存储若干 `pair`：

```
哈希函数 f(x):
  "CS106L" → f(x) → 80489869 → 80489869 mod 5 = 4

桶号:    0             1             2             3             4
    +-----------+ +-----------+ +-----------+ +-----------+ +-----------+
    | "Chris":31| |           | | "Nick":51 | | "Sean":35 | |"CS106L":0 |
    +-----------+ +-----------+ +-----------+ +-----------+ +-----------+
```

#### 哈希函数

- 将键"打散"为一个 `size_t`（64位整数）
- 输入的微小变化应产生输出的巨大变化（雪崩效应）
- 例如：`f("CS106L") = 80489869`，`f("CS106B") = 31580239`

#### 哈希冲突

如果两个键哈希到同一个桶，发生**哈希冲突**。查找时在桶内遍历，并通过键相等性检查来确定：

```
桶号 2（两个键哈希到同一桶）:
    +---------------------------+
    | "Nick":51  |  "Keith":14  |
    +---------------------------+
```

两个具有相同哈希值的键不一定相等！

#### ⚠️ unordered_map 要求键类型可哈希

```cpp
std::unordered_map<int, int> map1;            // OK —— int 可哈希
std::unordered_map<std::ifstream, int> map2;  // 错误 —— ifstream 不可哈希
```

大多数基本类型（int, double, string）默认是可哈希的。

#### 负载因子与重哈希

**负载因子（Load Factor）** = 平均每个桶中的元素数。

```cpp
std::unordered_map<std::string, int> map;
double lf = map.load_factor();     // 获取当前负载因子
map.max_load_factor(2.0);          // 设置最大负载因子（默认 1.0）
// 当负载因子超过 max_load_factor 时会自动重哈希
```

unordered_map 通过保持低负载因子来实现极快的查找（接近 O(1)）。

---

## 5.4 容器选择指南

### 数据结构操作复杂度对比

| 数据结构 | 第 i 个元素 | 搜索 | 插入 | 删除 | 空间 |
|----------|------------|------|------|------|------|
| **std::vector** | 极快 O(1) | 慢 O(n) | 慢 O(n)（尾部 O(1)） | 慢 O(n)（尾部 O(1)） | 低 |
| **std::deque** | 快 O(1) | 慢 O(n) | 两端快 O(1)，其他慢 | 两端快 O(1)，其他慢 | 低 |
| **std::set** | 慢 O(n) | 快 O(log n) | 快 O(log n) | 快 O(log n) | 中 |
| **std::map** | 慢 O(n) | 快 O(log n) | 快 O(log n) | 快 O(log n) | 中 |
| **std::unordered_set** | N/A | 极快 O(1) | 极快 O(1) | 极快 O(1) | 高 |
| **std::unordered_map** | N/A | 极快 O(1) | 极快 O(1) | 极快 O(1) | 高 |

### 空间-时间权衡

> "Space is time"（空间就是时间）—— Bjarne Stroustrup

| | 杂乱的车库（vector）| 井井有条的车库（map）|
|---|---|---|
| **空间效率** | 高 | 低 |
| **查找速度** | 慢 | 快 |
| **类比容器** | `std::vector<T>` | `std::map<std::string, T>` |

### 选择建议

- **unordered_map 通常比 map 更快**，但使用更多内存
- 如果你的键类型没有全序关系（operator<），必须使用 unordered_map
- 如果不确定，**unordered_map 是安全的默认选择**
- 需要有序遍历键时，使用 **map**
- 只需要唯一元素集合时，使用 **set** 而非 map

---

## 📚 补充知识点

### 更多 C++ 容器简介

| 容器 | 头文件 | 描述 |
|------|--------|------|
| `std::array<T, N>` | `<array>` | 固定大小的数组，大小在编译时确定 |
| `std::list<T>` | `<list>` | 双向链表，任意位置插入删除 O(1) |
| `std::forward_list<T>` | `<forward_list>` | 单向链表，内存更省 |
| `std::multiset<T>` | `<set>` | 允许重复元素的 set |
| `std::multimap<K,V>` | `<map>` | 允许同一键对应多值的 map |
| `std::unordered_multiset<T>` | `<unordered_set>` | 允许重复且基于哈希的 set |
| `std::unordered_multimap<K,V>` | `<unordered_map>` | 允许重复且基于哈希的 map |

### 容器适配器

| 适配器 | 底层默认容器 | 特点 |
|--------|-------------|------|
| `std::stack<T>` | `std::deque<T>` | 后进先出 (LIFO) |
| `std::queue<T>` | `std::deque<T>` | 先进先出 (FIFO) |
| `std::priority_queue<T>` | `std::vector<T>` | 最大元素优先出队 |

### vector 的内存管理

```cpp
std::vector<int> v;
v.reserve(1000);           // 预分配空间，避免反复重分配
v.shrink_to_fit();         // 释放多余容量，使 capacity == size
std::cout << v.capacity(); // 查看当前容量
```

### 结构化绑定 (Structured Bindings, C++17)

```cpp
// 遍历 map 的最佳实践
for (const auto& [key, value] : myMap) {
    std::cout << key << " -> " << value << "\n";
}

// 也适用于 pair、tuple
auto [x, y] = std::make_pair(1, 2.5);
```

---

## 🔧 常用API参考

### std::vector\<T\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `push_back(const T&)` | 在末尾添加元素 | `v.push_back(42);` |
| `pop_back()` | 移除末尾元素（不返回） | `v.pop_back();` |
| `emplace_back(Args&&...)` | 在末尾就地构造元素 | `v.emplace_back("hello", 5);` |
| `size()` | 返回元素数量 | `size_t n = v.size();` |
| `capacity()` | 返回分配的容量 | `size_t cap = v.capacity();` |
| `reserve(size_t)` | 预分配至少 n 个元素的空间 | `v.reserve(100);` |
| `shrink_to_fit()` | 释放多余容量 | `v.shrink_to_fit();` |
| `empty()` | 检查是否为空 | `if (v.empty()) ...` |
| `clear()` | 清空所有元素 | `v.clear();` |
| `operator[](size_t)` | 按索引访问（无边界检查） | `int x = v[0];` |
| `at(size_t)` | 按索引访问（带边界检查） | `int x = v.at(0);` |
| `front()` | 返回第一个元素的引用 | `int& f = v.front();` |
| `back()` | 返回最后一个元素的引用 | `int& b = v.back();` |
| `data()` | 返回底层数组的指针 | `int* p = v.data();` |
| `insert(iterator, const T&)` | 在指定位置插入元素 | `v.insert(v.begin(), 10);` |
| `erase(iterator)` | 删除指定位置的元素 | `v.erase(v.begin());` |
| `resize(size_t)` | 调整大小，多出部分默认初始化 | `v.resize(10);` |
| `begin()` / `end()` | 返回起始/结束迭代器 | `auto it = v.begin();` |
| `rbegin()` / `rend()` | 返回反向迭代器 | 用于反向遍历 |

### std::deque\<T\>

拥有 vector 的所有方法，额外提供：

| 方法 | 描述 | 示例 |
|------|------|------|
| `push_front(const T&)` | 在头部添加元素 | `d.push_front(1);` |
| `pop_front()` | 移除头部元素 | `d.pop_front();` |
| `emplace_front(Args&&...)` | 在头部就地构造 | `d.emplace_front("hi");` |

### std::map\<K, V\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `insert({key, value})` | 插入键值对 | `m.insert({"a", 1});` |
| `insert_or_assign(k, v)` (C++17) | 插入；若已有则赋值 | `m.insert_or_assign("a", 2);` |
| `erase(key)` | 删除指定键 | `m.erase("a");` |
| `erase(iterator)` | 删除迭代器指向的元素 | `m.erase(it);` |
| `find(key)` | 查找键，返回迭代器（找不到返回 end()） | `auto it = m.find("a");` |
| `count(key)` | 判断键是否存在（返回 0 或 1） | `if (m.count("a")) ...` |
| `contains(key)` (C++20) | 判断键是否存在 | `if (m.contains("a")) ...` |
| `at(key)` | 访问值（不存在时抛异常） | `int v = m.at("a");` |
| `operator[](key)` | 访问值（不存在时默认插入） | `m["a"] = 5;` |
| `lower_bound(key)` | 返回第一个 >= key 的迭代器 | `auto it = m.lower_bound("b");` |
| `upper_bound(key)` | 返回第一个 > key 的迭代器 | `auto it = m.upper_bound("b");` |
| `equal_range(key)` | 返回等于 key 的范围 [lower, upper) | `auto [l, u] = m.equal_range("a");` |
| `emplace(args...)` | 就地构造键值对 | `m.emplace("x", 10);` |
| `try_emplace(k, args...)` (C++17) | 仅在键不存在时插入 | `m.try_emplace("x", 10);` |
| `size()` | 返回元素数量 | `size_t n = m.size();` |
| `empty()` | 检查是否为空 | `if (m.empty()) ...` |
| `clear()` | 清空 | `m.clear();` |

### std::set\<T\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `insert(T)` | 插入元素 | `s.insert(42);` |
| `erase(T)` | 删除元素 | `s.erase(42);` |
| `erase(iterator)` | 通过迭代器删除 | `s.erase(it);` |
| `find(T)` | 查找，返回迭代器 | `auto it = s.find(42);` |
| `count(T)` | 判断是否存在 | `if (s.count(42)) ...` |
| `contains(T)` (C++20) | 判断是否存在 | `if (s.contains(42)) ...` |
| `lower_bound(T)` | 第一个 >= T 的元素 | `auto it = s.lower_bound(10);` |
| `upper_bound(T)` | 第一个 > T 的元素 | `auto it = s.upper_bound(10);` |
| `equal_range(T)` | 等于 T 的范围 | `auto [l, u] = s.equal_range(10);` |
| `emplace(args...)` | 就地构造元素 | `s.emplace(42);` |
| `size()` | 返回元素数量 | `size_t n = s.size();` |
| `empty()` | 检查是否为空 | `if (s.empty()) ...` |
| `clear()` | 清空 | `s.clear();` |

### std::unordered_map\<K, V\>

拥有 map 的大部分方法（除了 lower_bound/upper_bound/equal_range，因为无序）。

额外拥有：

| 方法 | 描述 | 示例 |
|------|------|------|
| `bucket_count()` | 返回桶的数量 | `size_t n = m.bucket_count();` |
| `bucket(key)` | 返回键所在桶的索引 | `size_t i = m.bucket("a");` |
| `bucket_size(size_t)` | 返回指定桶中的元素数 | `size_t n = m.bucket_size(0);` |
| `load_factor()` | 返回当前负载因子 | `double lf = m.load_factor();` |
| `max_load_factor()` | 获取最大负载因子 | `double ml = m.max_load_factor();` |
| `max_load_factor(float)` | 设置最大负载因子 | `m.max_load_factor(2.0);` |
| `rehash(size_t)` | 手动重哈希到指定桶数 | `m.rehash(100);` |
| `reserve(size_t)` | 预分配空间以容纳 n 个元素 | `m.reserve(1000);` |

### std::stack\<T\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `push(const T&)` | 入栈 | `s.push(42);` |
| `emplace(Args&&...)` | 就地构造并入栈 | `s.emplace(42);` |
| `pop()` | 出栈（不返回值） | `s.pop();` |
| `top()` | 返回栈顶元素引用 | `T& t = s.top();` |
| `empty()` | 检查是否为空 | `if (s.empty()) ...` |
| `size()` | 返回元素数量 | `size_t n = s.size();` |

### std::queue\<T\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `push(const T&)` | 入队（尾部） | `q.push(42);` |
| `emplace(Args&&...)` | 就地构造并入队 | `q.emplace(42);` |
| `pop()` | 出队（头部，不返回值） | `q.pop();` |
| `front()` | 返回队头元素引用 | `T& f = q.front();` |
| `back()` | 返回队尾元素引用 | `T& b = q.back();` |
| `empty()` | 检查是否为空 | `if (q.empty()) ...` |
| `size()` | 返回元素数量 | `size_t n = q.size();` |

### std::priority_queue\<T\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `push(const T&)` | 插入元素 | `pq.push(42);` |
| `emplace(Args&&...)` | 就地构造并插入 | `pq.emplace(42);` |
| `pop()` | 移除最大（默认）元素 | `pq.pop();` |
| `top()` | 返回最大（默认）元素引用 | `const T& t = pq.top();` |
| `empty()` | 检查是否为空 | `if (pq.empty()) ...` |
| `size()` | 返回元素数量 | `size_t n = pq.size();` |

> **提示**：默认最大堆。如需最小堆：`std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;`

### std::list\<T\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `push_back(T)` | 尾部插入 | `l.push_back(1);` |
| `push_front(T)` | 头部插入 | `l.push_front(1);` |
| `pop_back()` | 尾部删除 | `l.pop_back();` |
| `pop_front()` | 头部删除 | `l.pop_front();` |
| `splice(iterator, list&)` | 将另一个 list 的元素移动到此 list | `l1.splice(l1.begin(), l2);` |
| `merge(list&)` | 合并两个已排序的 list | `l1.merge(l2);` |
| `sort()` | 排序 | `l.sort();` |
| `reverse()` | 反转 | `l.reverse();` |
| `unique()` | 删除连续重复元素 | `l.unique();` |
| `remove(T)` | 删除所有等于 T 的元素 | `l.remove(3);` |

### std::array\<T, N\>

| 方法 | 描述 | 示例 |
|------|------|------|
| `size()` | 返回大小（编译时常量） | `size_t n = a.size();` |
| `operator[](size_t)` | 按索引访问 | `int x = a[0];` |
| `at(size_t)` | 按索引访问（带边界检查） | `int x = a.at(0);` |
| `front()` | 第一个元素 | `int& f = a.front();` |
| `back()` | 最后一个元素 | `int& b = a.back();` |
| `data()` | 返回底层数组指针 | `int* p = a.data();` |
| `fill(T)` | 填充所有元素为指定值 | `a.fill(0);` |
| `begin()` / `end()` | 迭代器 | 支持 for-range 遍历 |
