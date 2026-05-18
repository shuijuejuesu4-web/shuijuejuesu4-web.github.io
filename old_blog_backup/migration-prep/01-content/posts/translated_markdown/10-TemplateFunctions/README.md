# 第10章：模板函数 (Template Functions)

> **授课教师**：Rachel Fernandez, Thomas Poimenidis
> **学期**：Stanford CS106L, Fall 2025

---

## 目录

- [10.1 模板函数基础](#101-模板函数基础)
- [10.2 调用模板函数：显式与隐式实例化](#102-调用模板函数显式与隐式实例化)
- [10.3 实战：编写通用的 find 函数](#103-实战编写通用的-find-函数)
- [10.4 概念 (Concepts)](#104-概念-concepts)
- [10.5 可变参数模板 (Variadic Templates)](#105-可变参数模板-variadic-templates)
- [10.6 模板元编程 (Template Metaprogramming)](#106-模板元编程-template-metaprogramming)
- [📚 补充知识点](#-补充知识点)
- [🔧 常用API参考](#-常用api参考)

---

## 10.1 模板函数基础

### 10.1.1 问题：重复的函数重载

假设我们要实现一个 `min` 函数：

```cpp
int min(int a, int b) {
    return a < b ? a : b;
}
```

`min` 对不止整数有意义。我们需要：

```cpp
min(106, 107);             // int，返回 106
min(1.2, 3.4);             // double，返回 1.2
min("Thomas", "Rachel");   // string，返回 "Rachel"（字母序小的）
```

传统做法是重载：

```cpp
int min(int a, int b) { return a < b ? a : b; }
double min(double a, double b) { return a < b ? a : b; }
std::string min(std::string a, std::string b) { return a < b ? a : b; }
```

太多重复代码了！模板可以解决：

```cpp
// 这是模板（不是函数）
template <typename T>
T min(T a, T b) {
    return a < b ? a : b;
}
```

### 10.1.2 模板就像工厂

```
                template <typename T>
    int ---------> T min(T a, T b) ---------> min<int>
                                              min<string>
    string ----------------------------------> 等等
```

- **模板**不是函数——它是一个蓝图
- **模板实例化**才是具体的函数——编译器按需生成

### 10.1.3 使用引用避免拷贝

```cpp
// 改进版：使用 const 引用
template <typename T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}
```

---

## 10.2 调用模板函数：显式与隐式实例化

### 10.2.1 显式实例化

直接将类型参数传给模板：

```cpp
min<int>(106, 107);          // 返回 106
min<double>(1.2, 3.4);       // 返回 1.2
```

编译器为此生成代码：

```cpp
// 编译器生成的函数
int min(int a, int b) { return a < b ? a : b; }
double min(double a, double b) { return a < b ? a : b; }
```

### 10.2.2 隐式实例化

让编译器推断类型：

```cpp
min(106, 107);    // T 推断为 int
min(1.2, 3.4);    // T 推断为 double
```

这类似于 `auto`：类型仍然是确定的，只是让编译器帮你推导。

### 10.2.3 隐式实例化的陷阱

**陷阱一：C 风格字符串**

```cpp
min("Thomas", "Rachel");  // T 推断为 const char*！
                          // 比较的是指针地址，不是字符串内容！
```

解决方案——显式指定类型：

```cpp
min<std::string>("Thomas", "Rachel");  // const char* 转换为 std::string
```

**陷阱二：参数类型不匹配**

```cpp
min(106, 3.14);  // T 是 int 还是 double？编译错误！
```

解决方案——让模板更灵活：

```cpp
template <typename T, typename U>
auto min(const T& a, const U& b) {
    return a < b ? a : b;  // 返回类型由 auto 推断
}
```

> **提示**：使用 IDE 可以查看模板实例化后的实际类型（VSCode、QtCreator 等）。

---

## 10.3 实战：编写通用的 find 函数

### 10.3.1 动机

不同容器有不同的迭代器类型：

```
vector<T>::iterator      deque<T>::iterator
map<K,V>::iterator       unordered_map<K,V>::iterator
```

但它们共享相同的接口：

```cpp
auto it = c.begin();      // 拷贝构造
++it;                     // 前向递增
auto elem = *it;          // 解引用（it == end() 时是未定义行为）
if (it == c.end()) ...    // 相等比较
```

### 10.3.2 模板化的 find

```cpp
template <typename Iterator, typename TElem>
Iterator find(Iterator begin, Iterator end, TElem value) {
    Iterator it = begin;
    while (it != end) {
        if (*it == value) break;
        ++it;
    }
    return it;
}

// 使用示例
std::vector<int> v { 106, 111, 42, 112 };
auto it = find(v.begin(), v.end(), 42);
*it = 107;  // v = { 106, 111, 107, 112 }
```

> `find` 是 `<algorithm>` 头文件中的标准库函数，你已经具备读懂 C++ 标准的工具了！

---

## 10.4 概念 (Concepts)

### 10.4.1 问题：模板的错误信息难以理解

回到 `min` 函数：

```cpp
template <typename T>
T min(const T& a, const T& b) {
    return a < b ? a : b;  // T 必须有 operator<
}

struct StanfordID;  // 没有 operator<

min<StanfordID>(thomas, rachel);  // 编译器错误：几十行难以理解的信息
```

编译器只有在**实例化之后**才发现错误，此时错误信息已经深入到模板的内部实现中。

### 10.4.2 核心思想：给模板加约束

我们希望**提前声明**模板对类型参数的要求。其他语言已有此特性：

- **C#**：`where T : IComparable<T>`
- **Java**：`<T extends Comparable<T>>`

### 10.4.3 C++20 Concepts

Concept 是一组命名的约束（constraints）：

```cpp
template <typename T>
concept Comparable = requires(const T a, const T b) {
    { a < b } -> std::convertible_to<bool>;
    // 要求：a < b 能够编译，且结果可转换为 bool
};
```

使用 concept：

```cpp
template <typename T> requires Comparable<T>
T min(const T& a, const T& b);

// 更简洁的写法
template <Comparable T>
T min(const T& a, const T& b);
```

### 10.4.4 Concepts 的优势

1. **更好的编译器错误信息**：错误在实例化之前就被发现
2. **更好的 IDE 支持**：智能提示和自动补全更准确

### 10.4.5 C++ 内置的 Concepts

C++20 标准库包含许多内置 concepts：

- 迭代器 concepts：`std::input_iterator`, `std::output_iterator`, `std::forward_iterator`, `std::bidirectional_iterator`, `std::random_access_iterator`

```cpp
// 使用 concepts 约束 find 函数
template <std::input_iterator It, typename T>
It find(It begin, It end, const T& value);
```

### 10.4.6 Concepts 总结

使用 concepts 的两个主要原因：
1. 更好的编译器错误信息
2. 更好的 IDE 支持（IntelliSense / 自动补全等）

> Concepts 仍是新特性，STL 尚未完全支持。

---

## 10.5 可变参数模板 (Variadic Templates)

### 10.5.1 问题：接受任意数量参数

```cpp
min(2.4, 7.5);            // 这可以
min(2.4, 7.5, 5.3);       // 这个呢？
min(2.4, 7.5, 5.3, 1.2);  // 或者这个？
```

### 10.5.2 方案一：vector + 递归

```cpp
template <Comparable T>
T min(const std::vector<T>& values) {
    if (values.size() == 1) return values[0];        // 基础情况
    const auto& first = values[0];
    std::vector<T> rest(++values.begin(), values.end());
    auto m = min(rest);                               // 递归情况
    return first < m ? first : m;
}
```

缺点：
- 每次递归都会拷贝 vector
- 每次调用都需要分配 vector

### 10.5.3 方案二：可变参数模板

```cpp
// 基础情况：停止递归
template <Comparable T>
T min(const T& v) { return v; }

// 可变参数模板：匹配 0 个或更多类型
template <Comparable T, Comparable... Args>
T min(const T& v, const Args&... args) {
    auto m = min(args...);         // 包展开
    return v < m ? v : m;
}
```

**关键概念**：

| 语法 | 含义 |
|------|------|
| `Comparable... Args` | **可变模板参数**：匹配 0 个或更多类型 |
| `const Args&... args` | **参数包**：0 个或更多函数参数 |
| `min(args...)` | **包展开**：将 `...args` 替换为实际参数 |

### 10.5.4 可变参数模板的工作过程

```
min(2, 7, 5, 1) 的调用链：

min<int, int, int, int>(2, 7, 5, 1)    // T=int, Args=[int,int,int]
    → min(a0, a1, a2)                    // 生成 min<int,int,int>
        → min(a0, a1)                    // 生成 min<int,int>
            → min(a0)                    // 基础情况，返回 a0
```

**一次调用生成了 4 个函数！** 编译器在编译时通过递归自动生成了所有需要的重载。

### 10.5.5 类型不必相同：实现 format 函数

```cpp
format("Queen {}, Protector of the {} Kingdoms", "Rhaenyra", 7);
// 输出: Queen Rhaenyra, Protector of the 7 Kingdoms
```

实现：

```cpp
void format(const std::string& fmt) {
    std::cout << fmt << std::endl;
}

template <typename T, typename... Args>
void format(const std::string& fmt, T value, Args... args) {
    auto pos = fmt.find("{}");
    if (pos == std::string::npos) throw std::runtime_error("Extra arg");
    std::cout << fmt.substr(0, pos);
    std::cout << value;
    format(fmt.substr(pos + 2), args...);
}
```

调用链：

```
format("Lecture {}: {} (Week {})", 9, "Templates", 5)
    → format<int, string, int>()
    → format<string, int>()
    → format<int>()
    → format()  // 基础情况
```

### 10.5.6 可变参数模板总结

- 编译器通过递归生成任意数量的重载
- 实例化发生在**编译时**

---

## 10.6 模板元编程 (Template Metaprogramming)

### 10.6.1 核心思想：在编译时做计算

```cpp
// 主模板（递归情况）
template <size_t N>
struct Factorial {
    enum { value = N * Factorial<N - 1>::value };
};

// 模板特化：基础情况 N=0（必须在主模板之后）
template <>
struct Factorial<0> {
    enum { value = 1 };       // enum：编译时常量
};

std::cout << Factorial<7>::value << std::endl;  // 输出 5040
```

### 10.6.2 编译时求值过程

```
Factorial<7>::value  = 5040
Factorial<6>::value  = 720
Factorial<5>::value  = 120
Factorial<4>::value  = 24
Factorial<3>::value  = 6
Factorial<2>::value  = 2
Factorial<1>::value  = 1
Factorial<0>::value  = 1        ← 全部在编译时完成！
```

生成的汇编代码中，`5040` 是硬编码的常量——运行时零开销。

### 10.6.3 现代方案：constexpr 和 consteval（C++20）

```cpp
constexpr size_t factorial(size_t n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}
// constexpr: "编译器，请尽量在编译时运行我"

consteval size_t factorial(size_t n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}
// consteval: "编译器，你必须在编译时运行我"
```

### 10.6.4 何时使用模板

| 需求 | 解决方案 |
|------|----------|
| 让编译器自动化重复编码 | 模板函数、可变参数模板 |
| 需要更好的错误信息 | Concepts |
| 不想等到运行时 | 模板元编程、constexpr/consteval |

---

## 📚 补充知识点

### 模板 vs. 重载：何时用哪个？

- **模板**：当逻辑对多种类型完全相同
- **重载**：当不同类型需要不同的实现逻辑
- **模板特化**：当大多数类型共享逻辑，但某些类型需要特殊处理

```cpp
// 通用模板
template <typename T>
T abs(T x) { return x < 0 ? -x : x; }

// 对 std::string 的特殊处理（假设 abs 对字符串无意义）
template <>
std::string abs(std::string x) = delete;  // 禁止使用！
```

### SFINAE（Substitution Failure Is Not An Error）

在 concepts 出现之前，C++ 使用 SFINAE 机制来实现类型约束：

```cpp
// 传统做法：当 T 不能比较时，模板被静默排除
template <typename T>
auto min(const T& a, const T& b) -> decltype(a < b, T{}) {
    return a < b ? a : b;
}
```

C++20 concepts 大大简化了这类代码。

### template \<typename T\> vs template \<class T\>

两者完全等价。`typename` 强调类型参数可以是任何类型，`class` 是历史遗留。推荐用 `typename` 表示任何类型，`class` 用于需要类的场景（语义上）。

### 显式与隐式实例化的选择

| 场景 | 推荐 |
|------|------|
| 类型明确无歧义 | 隐式实例化 `min(3, 5)` |
| 类型有歧义 | 显式实例化 `min<double>(3, 3.14)` |
| C 字符串 | 显式实例化 `min<std::string>(...)` |

---

## 🔧 常用API参考

### 模板声明语法

| 语法 | 说明 | 示例 |
|------|------|------|
| `template <typename T>` | 单类型参数 | `template <typename T> class vector;` |
| `template <typename T, typename U>` | 多类型参数 | `template <typename K, typename V> class map;` |
| `template <typename T, size_t N>` | 类型参数 + 非类型参数 | `template <typename T, size_t N> class array;` |
| `template <typename... Args>` | 可变参数模板 | `template <typename... Args> void f(Args...);` |

### C++20 Concepts 常用语法

```cpp
// 定义一个 concept
template <typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};

// 使用 concept（四种等价写法）
template <Comparable T>
T min(const T& a, const T& b);

template <typename T> requires Comparable<T>
T min(const T& a, const T& b);

template <typename T>
T min(const T& a, const T& b) requires Comparable<T>;

Comparable auto x = min(3, 5);  // 约束 auto
```

### C++20 内置 Concepts

| Concept | 说明 |
|---------|------|
| `std::same_as<T>` | 类型相同 |
| `std::derived_from<Base>` | 派生自 Base |
| `std::convertible_to<T>` | 可转换为 T |
| `std::integral<T>` | 整数类型 |
| `std::floating_point<T>` | 浮点类型 |
| `std::copyable<T>` | 可拷贝 |
| `std::movable<T>` | 可移动 |
| `std::input_iterator<T>` | 输入迭代器 |
| `std::output_iterator<T>` | 输出迭代器 |
| `std::forward_iterator<T>` | 前向迭代器 |
| `std::bidirectional_iterator<T>` | 双向迭代器 |
| `std::random_access_iterator<T>` | 随机访问迭代器 |

### 编译时计算

```cpp
// constexpr 函数（C++11+）
constexpr int square(int x) { return x * x; }
static_assert(square(5) == 25);  // 编译时断言

// consteval 函数（C++20+）—— 必须在编译时求值
consteval int cube(int x) { return x * x * x; }

// constinit 变量（C++20+）—— 必须在编译时初始化
constinit int arr[] = {1, 2, 3};

// if constexpr（C++17+）—— 编译时分枝
template <typename T>
auto get_value(T t) {
    if constexpr (std::is_pointer_v<T>)
        return *t;
    else
        return t;
}
```
