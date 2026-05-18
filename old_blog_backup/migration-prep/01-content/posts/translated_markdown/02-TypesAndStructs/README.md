# 第2章：类型与结构体

> **授课教师**：Rachel Fernandez, Thomas Poimenidis
> **学期**：Stanford CS106L, Fall 2025

---

## 2.1 上次课程回顾

- 课程介绍
- 为什么你应该选修 CS106L？
- 课程安排
- C++ 的演化历史

> 课件可从 cs106l.stanford.edu 获取

---

## 2.2 编译时 vs 运行时

### 2.2.1 解释型语言 (Interpreted Languages)

解释型语言逐行读取代码，每读一行就翻译一行并立即执行。

```
源代码 (Source Code) → 解释器 (Interpreter) → 机器码 (Machine Code) → 输出 (Output)
```

示例（Python）：

```python
print("Hello World")
print("Welcome to ")
for ch in "CS106L":
    print(ch)
```

> **核心理念**：解释型语言逐行读取代码，一行一行地翻译，然后立即执行。

### 2.2.2 编译型语言 (Compiled Languages)

编译器将**整个**程序翻译后，打包成一个可执行文件，然后执行。

```
源代码 → 编译器 (Compiler) → 机器码 → 可执行文件 (Executable File) → 输出
```

示例（C++）：

```cpp
std::cout << "Hello World" << std::endl;
std::cout << "Welcome to " << std::endl;
for (char ch : "CS106L")
{
    std::cout << ch << std::endl;
}
```

> **核心理念**：编译器翻译整个程序，将其打包成可执行文件，然后执行。

### 2.2.3 编译时 vs 运行时总结

**编译型语言**（如 C++）：先编译时 (Compile Time)，再运行时 (Run Time)

**解释型语言**（如 Python）：一切都在运行时 (Run Time)

```
🖥 编译时 (Compile Time)     →     🏃 运行时 (Run Time)
源代码 → 编译器 → 机器码        →     可执行文件 → 输出
```

---

## 2.3 C++ 是编译型语言

### 2.3.1 错误发生在什么时候？

**Python（解释型语言）**：

```python
print("Running...")
hello = "Hello "
world = "World!"
print(hello * world)
```

```bash
$ python3 program.py
Running...
TypeError: can't multiply sequence by non-int of type 'str'
```

错误发生在**运行时**（Run Time Error）—— 程序先打印了 "Running..."，然后才崩溃。

**C++（编译型语言）**：

```cpp
int main() {
    std::cout << "Running..." << std::endl;
    std::string hello = "Hello ";
    std::string world = "World!";
    std::cout << hello * world << std::endl;
    return 0;
}
```

```bash
$ g++ main.cpp
error: no match for 'operator*' (operand types are 'std::string' and 'std::string')
```

错误发生在**编译时**（Compile Time Error）—— 程序根本没有运行！

> **核心理念**：在 C++ 中，编译器在翻译阶段就发现两个字符串不能相乘，直接报错。这意味着在编译时就捕获了类型错误，而不是等到运行时。

### 2.3.2 C++ 编译器为什么这么"冗长"？

因为编译器在处理所有类型信息。当类型的嵌套层次很深时，错误信息会非常冗长：

```
rtmap.cpp:19: invalid conversion from 'int' to
'std::_Rb_tree_node<std::pair<const int, double> >*'
...
```

这不是编译器的 bug —— 它在尽可能详细地告诉你问题出在哪里。

---

## 2.4 类型 (Types)

### 2.4.1 什么是类型？

类型指的是变量的"类别"。C++ 具有以下基本类型：

| 类型 | 示例 | 说明 |
|------|------|------|
| `int` | `106` | 整数 |
| `double` | `71.4` | 双精度浮点数 |
| `std::string` | `"Welcome to CS106L!"` | 字符串 |
| `bool` | `true` / `false` | 布尔值 |
| `size_t` | `12` | 非负整数（通常用于大小和索引） |

### 2.4.2 C++ 是静态类型语言 (Statically Typed Language)

编译器在生成机器码之前会检查类型。这意味着 **C++ 是一门静态类型语言**。

---

## 2.5 动态类型 vs 静态类型

### 2.5.1 动态类型 (Dynamic Typing) — Python 示例

```python
a = 3
b = "test"

def foo(c):
    d = 106
    d = "hello world!"  # d 的类型在运行时动态改变
```

解释器在运行时根据变量的当前值来分配类型。变量 `d` 一开始是 `int`，后来变成了 `string` —— 完全没问题！

### 2.5.2 静态类型 (Static Typing) — C++ 示例

```cpp
int a = 3;
std::string b = "test";

void foo(string c)
{
    int d = 106;
    d = "hello world!";  // ❌ 编译错误！
}
```

- 每个变量必须声明一个类型
- 一旦声明，类型就不能改变

### 2.5.3 为什么需要静态类型？

| 优势 | 说明 |
|------|------|
| **更高效** | 编译器可以进行更多优化 |
| **更易理解** | 代码意图更加明确 |
| **更好的错误检查** | 类型错误在编译时捕获 |

**更好的错误检查示例**：

```python
# Python — 运行时错误
def add_3(x):
    return x + 3

add_3("CS106L")  # Oops，这是字符串。运行时错误！
```

```cpp
// C++ — 编译时错误
int add_3(int x) {
    return x + 3;
}

add_3("CS106L");  // 不能将字符串传递给期望 int 的函数。编译时错误！
```

---

## 2.6 类型推断练习

请填写下列变量或函数返回值的正确类型（注意 `(int) x` 表示将 `x` 强制转换为 `int`，丢弃小数部分）：

```cpp
______   a = "test";           // string
______   b = 3.2 * 5 - 1;     // double
______   c = 5 / 2;           // int (整数除法，结果为 2)
______   d(int foo) { return foo / 2; }      // int
______   e(double foo) { return foo / 2; }   // double
______   f(double foo) { return (int)(foo + 0.5); }  // int
______   g(double c) { std::cout << c << std::endl; }  // void
```

---

## 2.7 函数重载 (Function Overloading)

定义两个名称相同但参数不同的函数。

```cpp
double axolotl(int x) {       // (1)
   return (double) x + 3;     // 类型转换: int → double
}

double axolotl(double x) {    // (2)
   return x * 3;
}

axolotl(2);      // 使用版本 (1)，返回 5.0
axolotl(2.0);    // 使用版本 (2)，返回 6.0
```

编译器根据传入的参数类型来决定调用哪个版本的函数。

> **总结**：C++ 是一门 **编译型、静态类型** 的语言。

---

## 2.8 结构体 (Structs)

### 2.8.1 问题：如何返回多个值？

假设我们要返回一个学生的 ID 信息（姓名、SUNet、ID 号），一个函数只能返回一个值：

```cpp
return type issueNewID() {
   // 如何返回三个东西？
   // 返回类型应该是什么？

   // 在 Python 中可以这样：
   // return "Stanford Tree", "theTREE", 0000002  // 注意：前导 0 表示八进制，这里 0000002₈ = 2₁₀
}
```

### 2.8.2 引入结构体！

结构体将数据捆绑在一起：

```cpp
struct StanfordID {
   std::string name;       // 这些叫做字段 (fields)
   std::string sunet;      // 每个字段有名称和类型
   int idNumber;
};

StanfordID id;                     // 初始化结构体
id.name = "THE Stanford Tree";     // 用 '.' 访问字段
id.sunet = "theTREE";
id.idNumber = 0000002  // 注意：前导 0 表示八进制，这里 0000002₈ = 2₁₀;
```

### 2.8.3 返回多个值

```cpp
StanfordID issueNewID() {
   StanfordID id;
   id.name = "THE Stanford Tree";
   id.sunet = "theTREE";
   id.idNumber = 0000002  // 注意：前导 0 表示八进制，这里 0000002₈ = 2₁₀;
   return id;
}
```

### 2.8.4 统一初始化 (Uniform Initialization)

```cpp
// 顺序取决于结构体中字段的顺序，'=' 是可选的
StanfordID tree = { "THE Stanford Tree", "theTREE", 0000002  // 注意：前导 0 表示八进制，这里 0000002₈ = 2₁₀ };
StanfordID lelandjr { "Leland Stanford Jr", "thejunior", 5430282 };
```

> **核心理念**：结构体将命名的变量捆绑成一个新的类型。

---

## 2.9 多种结构体示例

```cpp
struct Name {
    std::string first;
    std::string last;
};
Name rf = { "Rachel", "Fernandez" };

struct Order {
    std::string item;
    int quantity;
};
Order dozen = { "Eggs", 12 };

struct Point {
    double x;
    double y;
};
Point origin { 0.0, 0.0 };

struct Circle {
    Point center;
    double radius;
};
Circle circle { {0, 0} , 50000000 };
```

---

## 2.10 `std::pair`

### 2.10.1 基本用法

`std::pair` 是一个通用结构体，包含两个字段：

```cpp
// 不用 std::pair
struct Order {
    std::string item;
    int quantity;
};
Order dozen = { "Eggs", 12 };

// 用 std::pair
std::pair<std::string, int> dozen { "Eggs", 12 };
std::string item = dozen.first;      // "Eggs"
int quantity = dozen.second;         // 12
```

### 2.10.2 `std::pair` 是一个模板

```cpp
template <typename T1, typename T2>
struct pair {
   T1 first;
   T2 second;
};
```

这就解释了为什么我们可以用不同的类型来实例化 `std::pair`。

---

## 2.11 `std` —— C++ 标准库

### 2.11.1 什么是 `std`？

- C++ 提供的内置类型、函数和其他功能
- 需要使用 `#include` 引入相关头文件：
  - `#include <string>` → `std::string`
  - `#include <utility>` → `std::pair`
  - `#include <iostream>` → `std::cout`, `std::endl`
- 标准库名称前缀 `std::`
  - 如果写 `using namespace std;` 可以省略前缀，但这被认为是糟糕的风格，因为它会引入歧义
  - （如果我们定义了自己的 `string` 会发生什么？）

### 2.11.2 参考资源

- 查阅官方标准：cppreference.com
- 避免使用 cplusplus.com —— 它已过时且充满广告

### 2.11.3 `#include` 做了什么？

```cpp
#include <utility>

// 现在可以在代码中使用 std::pair

std::pair<double, double> p { 1.0, 2.0 };
```

`#include` 本质上是将头文件的内容"复制粘贴"到你的代码中：

```cpp
// #include <utility> 的效果相当于：

namespace std {
    template <typename T1, typename T2>
    struct pair {
         T1 first;
         T2 second;
    };
    // 其他 utility 代码...
}

std::pair<double, double> p { 1.0, 2.0 };
```

---

## 2.12 代码演示：求解二次方程

### 2.12.1 问题定义

给定系数 a, b, c，求解二次方程  $ax^2 + bx + c = 0$  的解：

```cpp
std::pair<bool, std::pair<double, double>> solveQuadratic(double a, double b, double c);
//        ↑是否有解？    ↑两个根
```

返回值示例：
- `{ true, { 1.0, 2.0 } }` — 有解，两根为 1.0 和 2.0
- `{ false, { 0.0, 0.0 } }` — 无解

> `sqrt` 函数来自 `<cmath>` 头文件，用于计算平方根。

---

## 2.13 改善代码

### 2.13.1 `using` 关键字

输入长类型名称很累。我们可以用 `using` 关键字创建类型别名：

```cpp
// 之前
std::pair<bool, std::pair<double, double>> solveQuadratic(double a, double b, double c);

// 之后
using Zeros = std::pair<double, double>;
using Solution = std::pair<bool, Zeros>;
Solution solveQuadratic(double a, double b, double c);
```

`using` 为类型创建别名，就像给变量起别名一样！

### 2.13.2 `auto` 关键字

`auto` 关键字告诉编译器去推断类型：

```cpp
// 之前
std::pair<bool, std::pair<double, double>> result = solveQuadratic(a, b, c);

// 之后
auto result = solveQuadratic(a, b, c);
// 完全等价！result 仍然是 std::pair<bool, std::pair<double, double>>
// 我们只是让编译器帮我们搞清楚了！
```

**重要提醒**：`auto` 仍然是静态类型的！

```cpp
auto i = 1;      // int 被推断
i = "hello!";    // ❌ 编译不通过！
```

### 2.13.3 `auto` vs 显式类型——哪个更清晰？

```cpp
std::pair<bool, std::pair<double, double>> result = ...;  // 啰嗦
auto result = ...;                                         // 简洁
```

```cpp
auto i = 1;     // 不太清晰
int i = 1;      // 更清晰
```

**使用建议**：当类型极其冗长或显而易见时使用 `auto`，当能增强代码可读性时显式写出类型。

---

## 2.14 本章回顾

- C++ 是一门**编译型、静态类型**的语言
- **结构体**将数据捆绑成单个对象
- `std::pair` 是一个具有两个字段的通用结构体
- 使用 `#include` 从 C++ 标准库引入内置类型（记得用 `std::` 前缀）
- 提升代码品质的特性：
  - `using` — 创建类型别名
  - `auto` — 让编译器推断变量类型

---

## 📚 补充知识点

### 编译器 vs 解释器的深入对比

| 特性 | 编译器 (C++) | 解释器 (Python) |
|------|-------------|----------------|
| 错误检测时机 | 编译时 | 运行时 |
| 执行速度 | 快（直接机器码） | 慢（额外解释开销） |
| 开发迭代速度 | 慢（需要编译） | 快（无编译步骤） |
| 类型系统 | 静态 | 动态 |
| 内存管理 | 手动/RAII | 自动（GC） |

### C++ 中的其他初始化方式

虽然本章展示了两种主要的初始化方式，但 C++ 实际上有多种初始化语法：

```cpp
int a = 5;        // 拷贝初始化 (Copy Initialization)
int b(5);         // 直接初始化 (Direct Initialization)
int c{5};         // 统一初始化/列表初始化 (Uniform/List Initialization)
int d = {5};      // 拷贝列表初始化 (Copy List Initialization)
```

推荐优先使用统一初始化 `{}`，因为它不允许窄化转换（narrowing conversion），更安全。详细内容将在第 3 章中介绍。

### `auto` 的使用准则

使用 `auto` 的推荐场景：
- 迭代器类型（极其冗长）
- Lambda 表达式类型（无法直接写出）
- 模板代码中的未知类型
- 当类型从初始化表达式中显而易见时

避免使用 `auto` 的场景：
- 降低代码可读性时
- 需要确保特定类型时（如 `uint32_t` vs `int`）
- 在公开接口中（头文件中的函数声明）

### 结构体与 class 的区别

在 C++ 中，`struct` 和 `class` 几乎相同，唯一的区别是：
- `struct` 的成员默认为 `public`
- `class` 的成员默认为 `private`

对于简单的数据聚合（如本节中的示例），使用 `struct` 是惯用法；对于具有不变式（invariant）和私有数据成员的类型，使用 `class`。

### `std::pair` vs `std::tuple`

`std::pair` 只能存储两个值。如果需要存储三个或更多值，可以使用 `std::tuple`（C++11）：

```cpp
#include <tuple>

std::tuple<std::string, std::string, int> student =
    std::make_tuple("Stanford Tree", "theTREE", 0000002  // 注意：前导 0 表示八进制，这里 0000002₈ = 2₁₀);

// 访问
std::string name = std::get<0>(student);
```

然而，如果字段有明确的语义名称，最好定义一个结构体而不是使用 `std::pair` 或 `std::tuple`，这样代码更具可读性。
