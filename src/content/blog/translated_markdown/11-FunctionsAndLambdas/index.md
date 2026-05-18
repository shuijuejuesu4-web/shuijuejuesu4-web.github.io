---
title: "第11章: 函数与Lambda表达式"
description: "第11章: 函数与Lambda表达式"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第11章: 函数与Lambda表达式

> **CS106L Fall 2025 - Lecture 11: Functions and Lambdas**

## 目录

- [11.1 回顾：模板函数](#111-回顾模板函数)
- [11.2 谓词 (Predicates)](#112-谓词-predicates)
- [11.3 将函数作为参数传递](#113-将函数作为参数传递)
- [11.4 Lambda表达式](#114-lambda表达式)
- [11.5 Lambda语法详解](#115-lambda语法详解)
- [11.6 Lambda的底层原理：函数对象](#116-lambda的底层原理函数对象)
- [11.7 `std::function` 类型](#117-stdfunction-类型)
- [11.8 STL算法](#118-stl算法)
- [11.9 实战：用STL算法实现Soundex](#119-实战用stl算法实现soundex)
- [11.10 Range与View](#1110-range与view)
- [11.11 补充知识点](#1111-补充知识点)
- [11.12 常用API参考](#1112-常用api参考)

---

## 11.1 回顾：模板函数

在上一讲中，我们学习了模板函数的基本概念。模板允许我们编写与类型无关的泛型代码。

```cpp
// 为非模板版本——针对不同类型需要重复编写
int min(int a, int b) {
  return a < b ? a : b;
}

double min(double a, double b) {
  return a < b ? a : b;
}

// 模板版本——一套代码适用于所有可比较的类型
template <typename T>
T min(T a, T b) {
  return a < b ? a : b;
}
```

模板函数的实例化可以是**显式的**（explicit）或**隐式的**（implicit）：

```cpp
min<int>(106, 107);          // 显式实例化：返回 106
min<double>(1.2, 3.4);       // 显式实例化：返回 1.2

int m = min(106, 107);       // 隐式实例化：编译器从参数类型推导出 T = int
```

### 泛型 `find` 函数

```cpp
template <typename It, typename T>
It find(It begin, It end, const T& value) {
  for (auto it = begin; it != end; ++it) {
     if (*it == value) return it;
  }
  return end;
}
```

**为什么传迭代器而不是传整个容器？** 因为传迭代器允许我们只在容器的子范围（subrange）中搜索：

```cpp
std::vector<int> v { 106, 107, 106, 143, 149, 106 };

// 搜索 106，跳过第一个和最后一个元素
auto it = find(v.begin() + 1, v.end() - 1, 106);

// 使用 std::distance 获取索引
std::cout << std::distance(v.begin(), it);  // 打印 2，不是 0
```

---

## 11.2 谓词 (Predicates)

### 定义

**谓词**（Predicate）是返回布尔值的函数。谓词分为一元谓词（unary predicate）和二元谓词（binary predicate）。

```cpp
// 一元谓词示例
bool isVowel(char c) {
   c = toupper(c);
   return c == 'A' || c == 'E' || c == 'I' || c == 'O' || c == 'U';
}

bool isPrime(size_t n) {
   if (n < 2) return false;
   for (auto i = 2; i <= sqrt(n); i++)
       if (n % i == 0) return false;
   return true;
}

// 二元谓词示例
bool isLessThan(int x, int y) {
   return x < y;
}

bool isDivisible(int n, int d) {
   return n % d == 0;
}
```

---

## 11.3 将函数作为参数传递

### 从 `find` 到 `find_if` 的进化

原先的 `find` 函数只能按值相等来查找，但如果我们想按某种条件查找，比如查找第一个元音字母或第一个素数，该怎么办？答案是：将谓词作为参数传递。

```cpp
// 进化后的 find_if 函数
template <typename It, typename Pred>
It find_if(It first, It last, Pred pred) {
  for (auto it = first; it != last; ++it) {
     if (pred(*it)) return it;  // 用谓词替换硬编码的 == value
  }
  return last;
}
```

**关键思想**：将函数作为参数传递，使我们能够用用户自定义的行为来泛化算法。

### 使用 `find_if`

```cpp
std::string corlys = "Lord of the Tides";
auto it = find_if(corlys.begin(), corlys.end(), isVowel);
*it = '0';  // "L0rd of the Tides"

std::vector<int> ints = {1, 0, 6};
auto it2 = find_if(ints.begin(), ints.end(), isPrime);
assert(it2 == ints.end());  // 没有找到素数
```

### 谓词的类型：函数指针

当我们将普通函数作为谓词传入时，`Pred` 实际上是一个**函数指针**（function pointer）：

```cpp
find_if(corlys.begin(), corlys.end(), isVowel);
// Pred = bool(*)(char)

find_if(ints.begin(), ints.end(), isPrime);
// Pred = bool(*)(int)
```

- `bool`：函数返回值类型
- `(*)`：函数指针标记
- `(char)` 或 `(int)`：参数类型

---

## 11.4 Lambda表达式

### 函数指针的局限性

考虑这个场景：我们想在一个 `vector` 中找小于 N 的数，但 N 在编译时不知道，只有在运行时才能从用户输入获取。

```cpp
// 函数指针无法优雅地解决这个问题
bool lessThan5(int x) { return x < 5; }
bool lessThan6(int x) { return x < 6; }
bool lessThan7(int x) { return x < 7; }
// ... 如果 N 从运行时读取，我们难道要写无穷多个函数吗？
```

**问题**：函数指针无法携带额外的状态。我们的 `isLessThan` 需要两个参数（元素值和 N），但 `find_if` 的谓词接口只接受一个参数（元素值）。

### Lambda：捕获状态的函数

**Lambda表达式** 是可以从外层作用域捕获状态的函数。它们解决了"给函数附加额外状态，而不引入额外参数"的问题。

```cpp
int n;
std::cin >> n;

auto lessThanN = [n](int x) { return x < n; };
//  n 被捕获（capture）到 lambda 内部

find_if(begin, end, lessThanN);
```

---

## 11.5 Lambda语法详解

```
 auto lessThanN = [n](int x) { return x < n; };
                   ^  ^^^^^  ^^^^^^^^^^^^^^^^
                   |    |           |
        捕获子句 ---+    |           +--- 函数体
                         |
          参数列表 ------+
```

### 捕获子句 (Capture Clause) 的完整语法

| 捕获方式 | 含义 |
|----------|------|
| `[x]` | 按值捕获 x（复制一份） |
| `[x&]` | 按引用捕获 x |
| `[x, y]` | 按值捕获 x 和 y |
| `[&]` | 按引用捕获所有变量 |
| `[=]` | 按值捕获所有变量 |
| `[&, x]` | 除 x 按值外，其余按引用捕获 |
| `[=, &x]` | 除 x 按引用外，其余按值捕获 |

### Lambda可以没有捕获

Lambda也可以用作"现场定义函数"的便捷方式：

```cpp
std::string corlys = "Lord of the tides";
auto it = find_if(corlys.begin(), corlys.end(),
  [](auto c) {
     c = toupper(c);
     return c == 'A' || c == 'E' ||
            c == 'I' || c == 'O' || c == 'U';
  });
```

### `auto` 参数 = 模板的简写

```cpp
auto lessThanN = [n](auto x) { return x < n; };
// 等价于：
template <typename T>
auto lessThanN = [n](T x) { return x < n; };
```

`auto` 参数在 lambda 中实际上是隐式模板参数，编译器通过隐式实例化推导类型。

---

## 11.6 Lambda的底层原理：函数对象

### 函数对象 (Functor)

**定义**：函数对象（Functor）是任何定义了 `operator()` 的对象。通俗地说，就是"行为像函数的对象"。

```cpp
// STL 中的 functor 示例：std::greater<T>
template <typename T>
struct std::greater {
   bool operator()(const T& a, const T& b) const {
     return a > b;
   }
};

std::greater<int> g;
g(1, 2);  // false
```

另一个重要例子是 `std::hash<T>`：

```cpp
template <>
struct std::hash<MyType> {
   size_t operator()(const MyType& v) const {
      // 自定义的哈希函数
      return ...;
   }
};
```

### Functor可以携带状态

```cpp
struct my_functor {
  bool operator()(int a) const {
     return a * value;
  }
  int value;  // Functor 的数据成员——这就是"状态"
};

my_functor f;
f.value = 5;
f(10);  // 50
```

### Lambda的本质：编译器自动生成Functor

**关键秘密**：当你写一个lambda时，编译器实际上为你生成了一个匿名的functor类。

```cpp
// 你写的代码：
int n = 10;
auto lessThanN = [n](int x) { return x < n; };

// 编译器实际生成的等价物：
class __lambda_6_18 {
public:
    bool operator()(int x) const { return x < n; }
    __lambda_6_18(int& _n) : n{_n} {}
private:
    int n;  // 捕获的变量变成了类的成员字段
};

int n = 10;
auto lessThanN = __lambda_6_18{n};  // 通过构造函数"捕获"变量
```

这解释了为什么lambda可以携带状态：捕获的变量变成了functor对象的成员变量。完全相同的道理就像range-for是迭代器循环的语法糖一样，lambda是functor的语法糖。

> 对底层实现感兴趣？访问 https://cppinsights.io/ 来查看C++代码的编译器展开！

---

## 11.7 `std::function` 类型

`std::function` 是C++标准库中用于表示可调用对象（函数、lambda、functor、函数指针）的通用类型包装器。

```cpp
#include <functional>

std::function<bool(int, int)> less = std::less<int>{};
std::function<bool(char)> vowel = isVowel;
std::function<int(int)> twice = [](int x) { return x * 2; };
```

**注意**：
- `std::function` 有轻微的性能开销（类型擦除的代价）
- 大多数情况下，使用 `auto` 或模板参数来接受 lambda/functor 类型会更高效
- `std::function` 适合用于需要存储不同类型可调用对象的场景（例如存到容器中）

---

## 11.8 STL算法

### STL的四大支柱

```
容器 (Containers)     迭代器 (Iterators)
 如何存储数据？          如何遍历容器？

函数对象 (Functors)    算法 (Algorithms)
 如何将函数表示为对象？   如何以泛型方式转换和修改容器？
```

### `<algorithm>` 常用算法

`<algorithm>` 头文件提供了丰富的模板函数，全部操作于迭代器之上：

| 算法 | 功能 | 签名 |
|------|------|------|
| `std::count_if` | 统计满足谓词的元素个数 | `(InputIt first, InputIt last, UnaryPred p)` |
| `std::sort` | 按比较器排序 | `(RandomIt first, RandomIt last, Compare comp)` |
| `std::max_element` | 找最大元素 | `(ForwardIt first, ForwardIt last, Compare comp)` |
| `std::copy_if` | 复制满足谓词的元素 | `(InputIt r1, InputIt r2, OutputIt o, UnaryPred p)` |
| `std::transform` | 对每个元素应用一元操作 | `(InputIt1 r1, InputIt1 r2, OutputIt o, UnaryOp op)` |
| `std::unique_copy` | 移除连续重复后复制 | `(InputIt i1, InputIt i2, OutputIt o, BinaryPred p)` |

### STL算法能做的事情

binary search, heap building, min/max, lexicographical comparisons, merge, set union, set difference, set intersection, partition, sort, nth sorted element, shuffle, selective removal, selective copy, for-each, random sample

——**所有这些都以最通用的形式提供！**

---

## 11.9 实战：用STL算法实现Soundex

### Soundex是什么？

Soundex是一种将名字转换为语音编码的算法。例如，"Roberts"和"Rupert"都编码为"R163"。

### 实现步骤

1. 从字符串中提取字母
2. 将每个字母替换为Soundex编码数字
3. 合并相邻重复数字（222025 变成 2025）
4. 用大写首字母替换第一个数字
5. 舍弃所有零
6. 确保编码恰好为4个字符（截断或补零）

### 用STL算法实现（只需3个算法函数）

```cpp
// 步骤1: 提取字母 —— 使用 std::copy_if
// 步骤2: 编码转换 —— 使用 std::transform
// 步骤3: 合并重复 —— 使用 std::unique_copy
// 步骤4-6: 字符串处理
```

---

## 11.10 Range与View

### 什么是Range？

**定义**：Range 是任何有 `begin()` 和 `end()` 的东西。

```
std::vector<T>          std::unordered_set<K,V>

          什么是Range？
               |
    std::map<K,V>      你自定义的有begin/end的类型
               |
          std::set<K>
```

### Range算法：STL v2

C++20/23/26 引入了 `std::ranges` 命名空间，提供接受整个容器（而不仅是迭代器对）的算法版本：

```cpp
// 传统 STL 方式
auto it = std::find(v.begin(), v.end(), 'c');

// Range 方式
auto it = std::ranges::find(v, 'c');
```

Range算法使用C++20的**concepts**进行约束，能产生更好的编译错误信息：

```cpp
template<class T>
concept range = requires(T& t) { ranges::begin(t); ranges::end(t); };
```

### Range算法 vs View：急切 vs 惰性

- **Range算法**是**急切（eager）的**：调用时立即执行
- **View**是**惰性（lazy）的**：只在需要结果时才真正执行

```cpp
// 急切求值：立即排序
std::ranges::sort(v);

// 惰性求值：先构建计算管线，最后才求值
auto view = letters
   | std::ranges::views::filter(isVowel)
   | std::ranges::views::transform(toupper);
std::vector<char> upperVowel = std::ranges::to<std::vector<char>>(view);
```

### View的管道语法

```cpp
std::vector<char> letters = {'a','b','c','d','e'};

std::vector<char> upperVowel = letters
  | std::ranges::views::filter(isVowel)    // 过滤出元音
  | std::ranges::views::transform(toupper) // 转为大写
  | std::ranges::to<std::vector<char>>();  // 物化为 vector

// upperVowel = { 'A', 'E' }
```

### View与Python生成器的类比

```cpp
// C++ (惰性求值)
auto view = letters
   | std::ranges::views::filter(isVowel)
   | std::ranges::views::transform(toupper);
auto upperVowel = std::ranges::to<std::vector<char>>(view);
```

```python
# Python (惰性求值)
view = (l for l in letters if isVowel(l))
view = (l.upper() for l in view)
upperVowel = list(view)
```

### Soundex的未来：C++26视图写法

```cpp
namespace rng = std::ranges;
namespace rv = std::ranges::views;

auto ch = *rng::find_if(s, isalpha);
auto sx = s | rv::filter(isalpha)
             | rv::transform(soundexEncode)
             | rv::unique
             | rv::filter(notZero)
             | rv::concat("0000")
             | rv::drop(1)
             | rv::take(3)
             | rng::to<std::string>();
return toupper(ch) + v;
```

---

## 11.11 补充知识点

### 1. Lambda捕获与悬垂引用

按引用捕获时务必注意生命周期问题：

```cpp
// 危险！lambda 在函数返回后可能调用，但 n 已被销毁
auto createLambda() {
    int n = 42;
    return [&n]() { return n; };  // n 是局部变量，返回后变成悬垂引用
}

// 安全：按值捕获
auto createLambdaSafe() {
    int n = 42;
    return [n]() { return n; };   // n 被复制到 lambda 内部
}
```

### 2. 可变Lambda (`mutable`)

默认情况下，按值捕获的变量在lambda中是const的，不可修改。使用 `mutable` 关键字可以解除这个限制：

```cpp
int counter = 0;
auto increment = [counter]() mutable {
    return ++counter;  // 没有 mutable 会编译错误
};
increment();  // 返回 1
increment();  // 返回 2（注意：原始 counter 仍为 0）
```

### 3. 泛型Lambda (C++14)

C++14引入了泛型lambda，使用 `auto` 作为参数类型：

```cpp
// C++14 泛型 lambda
auto genericLambda = [](auto a, auto b) { return a + b; };

genericLambda(1, 2);         // int + int = 3
genericLambda(1.5, 2.5);     // double + double = 4.0
genericLambda(std::string("hello "), std::string("world")); // 字符串拼接
```

### 4. 立即调用的Lambda (IILE)

Lambda可以在定义处立即调用，类似于JavaScript的IIFE：

```cpp
const auto result = [](int x) {
    // 复杂的初始化逻辑
    int sum = 0;
    for (int i = 0; i < x; ++i) sum += i * i;
    return sum;
}(100);  // 立即调用
// result 是常量，但初始化可以很复杂
```

### 5. Lambda作为Comparator

```cpp
std::vector<std::pair<std::string, int>> data = {{"B", 2}, {"A", 1}, {"C", 3}};

// 按 second 值排序
std::sort(data.begin(), data.end(),
    [](const auto& a, const auto& b) { return a.second < b.second; });

// 按 first 降序排序
std::sort(data.begin(), data.end(),
    [](const auto& a, const auto& b) { return a.first > b.first; });
```

### 6. Range/View的注意事项

- Range/View 非常新（C++20+），编译器支持可能不完整
- View 链式调用可能存在性能损失（相比手写循环），详见"The Terrible Problem of Incrementing a Smart Iterator"
- View 是惰性的，这意味着在物化之前不会有任何操作实际执行
- 在C++26之前，某些特性（如 `concat`）可能不可用

---

## 11.12 常用API参考

### Lambda相关

| API/语法 | 描述 | 示例 |
|----------|------|------|
| `[capture](params) { body }` | Lambda表达式 | `[](int x) { return x > 0; }` |
| `[=]` | 按值捕获所有变量 | `[=]() { return a + b; }` |
| `[&]` | 按引用捕获所有变量 | `[&]() { a += 1; }` |
| `[x]` | 按值捕获 x | `[x]() { return x * 2; }` |
| `[&x]` | 按引用捕获 x | `[&x]() { x *= 2; }` |
| `mutable` | 允许修改按值捕获的副本 | `[n]() mutable { return ++n; }` |
| `auto` 参数 | 泛型Lambda (C++14) | `[](auto a, auto b) { return a + b; }` |

### `<algorithm>` 常用算法

| 函数 | 描述 |
|------|------|
| `std::find(first, last, value)` | 查找等于value的第一个元素 |
| `std::find_if(first, last, pred)` | 查找满足谓词的第一个元素 |
| `std::count(first, last, value)` | 统计等于value的元素个数 |
| `std::count_if(first, last, pred)` | 统计满足谓词的元素个数 |
| `std::copy(first, last, dest)` | 复制元素 |
| `std::copy_if(first, last, dest, pred)` | 复制满足谓词的元素 |
| `std::transform(first, last, dest, op)` | 对每个元素应用op并存入dest |
| `std::sort(first, last)` | 排序（默认升序） |
| `std::sort(first, last, comp)` | 按比较器comp排序 |
| `std::max_element(first, last)` | 返回最大元素的迭代器 |
| `std::min_element(first, last)` | 返回最小元素的迭代器 |
| `std::unique_copy(first, last, dest)` | 去重后复制 |
| `std::for_each(first, last, f)` | 对每个元素执行操作f |
| `std::all_of/any_of/none_of` | 检查是否所有/任一/没有元素满足条件 |

### `<ranges>` (C++20) 常用API

| 函数/View | 描述 |
|-----------|------|
| `std::ranges::find(range, value)` | Range版find |
| `std::ranges::sort(range)` | Range版sort |
| `std::ranges::views::filter(range, pred)` | 惰性过滤 |
| `std::ranges::views::transform(range, op)` | 惰性变换 |
| `std::ranges::views::take(range, n)` | 取前n个元素 |
| `std::ranges::views::drop(range, n)` | 跳过前n个元素 |
| `std::ranges::to<T>(range)` | 将range物化为容器T |

### `<functional>` 常用类型

| 类型 | 描述 |
|------|------|
| `std::function<R(Args...)>` | 通用可调用对象包装器 |
| `std::less<T>` | 小于比较 functor |
| `std::greater<T>` | 大于比较 functor |
| `std::hash<T>` | 哈希 functor |

---

> **本章总结**：Lambda表达式是C++中表示可调用对象的强大工具，它本质上是编译器自动生成的函数对象。结合STL算法和C++20的Range/View，我们可以写出简洁、声明式、高度可复用的代码。理解lambda的捕获机制和底层原理，是掌握现代C++的关键一步。
