---
title: "第15章：std::optional 与类型安全"
description: "第15章：std::optional 与类型安全"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第15章：std::optional 与类型安全

> **授课教师**：Rachel Fernandez, Thomas Poimenidis
> **学期**：Stanford CS106L, Fall 2025

---

## 目录

- [15.1 回顾：移动语义与 Rule of Zero/Five](#151-回顾移动语义与-rule-of-zerofive)
- [15.2 类型安全（Type Safety）](#152-类型安全type-safety)
- [15.3 std::optional 基础](#153-stdoptional-基础)
- [15.4 std::optional 的接口](#154-stdoptional-的接口)
- [15.5 Monadic 操作（C++23）](#155-monadic-操作c23)
- [15.6 使用 std::optional 的好处与代价](#156-使用-stdoptional-的好处与代价)
- [📚 补充知识点](#-补充知识点)
- [🔧 常用API参考](#-常用api参考)

---

## 15.1 回顾：移动语义与 Rule of Zero/Five

### Rule of Zero（零规则）

如果成员变量有自我管理能力（如 `std::string`、`std::vector`），且不需要自定义构造/析构函数和运算符，**那就不要定义它们**！

C++ 会自动生成以下函数（如果你没有自定义）：
1. 析构函数：`~Student();`
2. 拷贝构造函数：`Student(const Student& other);`
3. 拷贝赋值运算符：`Student& operator=(const Student& other);`
4. 移动构造函数：`Student(Student&& other);`
5. 移动赋值运算符：`Student& operator=(Student&& other);`

### Rule of Three（三规则）

如果定义了**自定义析构函数**，则必须同时定义：
- 自定义拷贝构造函数
- 自定义拷贝赋值运算符

### Rule of Five（五规则）

如果定义了自定义拷贝构造函数和拷贝赋值运算符，**还应该定义**：
- 移动构造函数
- 移动赋值运算符

---

## 15.2 类型安全（Type Safety）

### 15.2.1 定义

> **类型安全**：一种编程语言防止类型错误的程度。

更具体地说：**函数签名在多大程度上保证了函数的行为**。

### 15.2.2 Python vs. C++

```python
# Python：运行时才会出错
def div_3(x):
    return x / 3

div_3("hello")  # 运行时崩溃！无法用字符串做除法
```

```cpp
// C++：编译时就拒绝
int div_3(int x) {
    return x / 3;
}

div_3("hello");  // 编译错误！这段代码永远不会运行
```

### 15.2.3 一个类型不安全的例子：vector::back()

```cpp
void removeOddsFromEnd(vector<int>& vec) {
    while (vec.back() % 2 == 1) {   // 如果 vec 为空会怎样？
        vec.pop_back();
    }
}
```

问题在于 `vector::back()` 的签名：

```cpp
valueType& vector<valueType>::back() {
    return *(begin() + size() - 1);  // 如果 size()=0，解引用无效指针！
}
```

`back()` **承诺**返回一个 `valueType&` 类型的值，但实际上可能根本不存在"最后一个元素"。这就是**函数签名给出了虚假承诺（false promise）**。

### 15.2.4 尝试修复

```cpp
// 方案一：抛异常
valueType& vector<valueType>::back() {
    if (empty()) throw std::out_of_range{};
    return *(begin() + size() - 1);
}
// 错误时至少会可靠地终止程序

// 方案二：返回 pair（有缺陷）
std::pair<bool, valueType> vector<valueType>::back() {
    if (empty()) {
        return {false, valueType()};  // 需要默认构造！而且开销大！
    }
    return {true, *(begin() + size() - 1)};
}
```

更好的方案：**std::optional**！

---

## 15.3 std::optional 基础

### 15.3.1 什么是 std::optional\<T\>？

`std::optional<T>` 是一个模板类，它**要么包含一个类型为 T 的值，要么什么都不包含**（表示为 `std::nullopt`）。

```cpp
#include <optional>

std::optional<int> num1 = {};       // num1 没有值
num1 = 1;                            // 现在它有了！
num1 = std::nullopt;                 // 现在又没有了
```

> **注意**：是 `nullopt`，不是 `nullptr`！
> - `nullptr`：可转换为任意指针类型的空值
> - `nullopt`：可转换为任意 optional 类型的空值

### 15.3.2 改进后的 back()

```cpp
std::optional<valueType> vector<valueType>::back() {
    if (empty()) {
        return {};           // 或 return std::nullopt;
    }
    return *(begin() + size() - 1);
}
```

现在 `back()` **诚实地声明**：结果可能存在，也可能不存在。

---

## 15.4 std::optional 的接口

### 15.4.1 .value() 方法

返回包含的值，如果不存在则抛出 `std::bad_optional_access` 异常。

```cpp
void removeOddsFromEnd(vector<int>& vec) {
    while (vec.back().value() % 2 == 1) {
        vec.pop_back();
    }
}
// 如果访问空 vector，至少可靠地获得 bad_optional_access 错误
```

### 15.4.2 .value_or(defaultValue) 方法

返回包含的值，如果不存在则返回默认值。

```cpp
std::optional<int> opt = std::nullopt;
int result = opt.value_or(42);  // result = 42
```

### 15.4.3 .has_value() 方法

检查是否包含值。

```cpp
void removeOddsFromEnd(vector<int>& vec) {
    while (vec.back().has_value() && vec.back().value() % 2 == 1) {
        vec.pop_back();
    }
}
// 不再出错，但代码比较冗长
```

### 15.4.4 布尔上下文

`std::optional` 可以直接用于布尔判断——`nullopt` 为 false，有值时为 true。

```cpp
void removeOddsFromEnd(vector<int>& vec) {
    while (vec.back() && vec.back().value() % 2 == 1) {
        vec.pop_back();
    }
}
// 更简洁的写法
```

---

## 15.5 Monadic 操作（C++23）

### 15.5.1 .and_then(function f)

如果包含值，返回 `f(value)` 的结果；否则返回 `nullopt`。`f` 必须返回 `optional`。

```cpp
void removeOddsFromEnd(vector<int>& vec) {
    auto isOdd = [](int num) -> std::optional<bool> {
        return num % 2 == 1;
    };
    while (vec.back().and_then(isOdd)) {
        vec.pop_back();
    }
}
```

### 15.5.2 .transform(function f)

如果包含值，返回 `f(value)` 包装在 `optional` 中的结果；否则返回 `nullopt`。

```cpp
std::optional<int> opt = 42;
auto result = opt.transform([](int x) { return x * 2; });
// result = optional<int>{84}
```

### 15.5.3 .or_else(function f)

如果包含值，返回该值；否则返回 `f()` 的结果。

```cpp
std::optional<int> opt = std::nullopt;
auto result = opt.or_else([] { return std::optional<int>(0); });
// result = optional<int>{0}
```

### 15.5.4 什么是 Monadic？

Monadic（单子）是一种软件设计模式，它将程序片段（函数）组合起来，并将返回值包装在带有额外计算功能的类型中。

这些操作让你能够**链式调用**函数，优雅地处理"可能存在也可能不存在"的值。

---

## 15.6 使用 std::optional 的好处与代价

### 优点

- 函数签名创建了更具信息量的契约
- 函数调用的行为有保证且可用

### 缺点

- 需要到处使用 `.value()`
- 仍然可能出现 `bad_optional_access` 异常
- `*optional`（不检查直接解引用）可能产生未定义行为
- **没有 `std::optional<T&>`**（引用必须指向有效对象，而 optional 不保证这一点）

```cpp
// std::optional<T&> 不可用
// operator[] 最好的做法还是像 .at() 那样抛出异常
valueType& vector<valueType>::operator[](size_t index) {
    return *(begin() + index);        // 不检查，快速
}
valueType& vector<valueType>::at(size_t index) {
    if (index >= size()) throw std::out_of_range{};  // 检查，安全
    return *(begin() + index);
}
```

### 为什么 vector::back() 实际上不返回 optional？

（而且大概率永远不会）

这与 C++ 的设计哲学一致：
- 不牺牲性能，除非万不得已
- 只在编译时强制执行安全性（optional 会引入运行时开销）

---

## 📚 补充知识点

### 其他语言中的 Optional

Rust、Swift、JavaScript 等语言大量使用 optional/monadic 模式：

- **Rust**：`Option<T>` —— 系统语言，保证内存和线程安全
- **Swift**：`Optional<T>` —— Apple 的语言，专门为应用开发设计
- **JavaScript**：可选链（Optional Chaining）`?.`

### 实际应用场景

```cpp
// 场景一：查找可能不存在的元素
std::optional<Student> findStudent(int id) {
    if (auto it = students.find(id); it != students.end())
        return it->second;
    return std::nullopt;
}

// 场景二：解析可能失败的操作
std::optional<int> parseInteger(const std::string& str) {
    try {
        return std::stoi(str);
    } catch (...) {
        return std::nullopt;
    }
}

// 场景三：配置项可能不存在
std::optional<std::string> getConfig(const std::string& key) {
    if (configMap.contains(key))
        return configMap[key];
    return std::nullopt;
}
```

### optional 与指针的对比

| | `std::optional<T>` | `T*` |
|---|---|---|
| 表达"可能没有值" | 是 | 是（nullptr） |
| 所有权语义 | 值语义（栈上） | 指针语义（可能堆上） |
| 内存开销 | 额外一个 bool + 可能的 padding | 指针大小 |
| 安全性 | 有 `.value()` 检查 | 无强制检查 |
| 是否可能为 dangling | 否（值语义） | 是（悬垂指针） |

### 为什么没有 optional\<T&\>？

标准委员会的讨论仍在进行中。核心问题：
- 引用必须始终绑定到有效对象
- `optional<T&>` 提供了 `nullopt` 状态，此时引用"悬空"
- 这使得 `optional<T&>` 的行为更接近指针，淡化了引用和指针的语义区别

---

## 🔧 常用API参考

### std::optional\<T\>

需要 `#include <optional>`

#### 创建与赋值

| 操作 | 说明 | 示例 |
|------|------|------|
| 默认构造 | 创建空的 optional | `std::optional<int> opt;` |
| 值构造 | 创建包含值的 optional | `std::optional<int> opt = 42;` |
| `std::nullopt` | 空的 optional | `std::optional<int> opt = std::nullopt;` |
| `std::make_optional<T>(args...)` | 就地构造 | `auto opt = std::make_optional<std::string>(3, 'a');` |
| `std::in_place` | 就地构造标记 | `std::optional<std::string> opt(std::in_place, 3, 'a');` |
| `operator=(T)` | 赋值 | `opt = 42;` |
| `operator=(nullopt)` | 清空 | `opt = std::nullopt;` |
| `emplace(args...)` | 就地构造新值 | `opt.emplace(3, 'a');` |
| `reset()` | 清空 | `opt.reset();` |

#### 访问值

| 操作 | 说明 | 示例 |
|------|------|------|
| `value()` | 返回值，空时抛出 `bad_optional_access` | `int x = opt.value();` |
| `value_or(T)` | 返回值，空时返回默认值 | `int x = opt.value_or(0);` |
| `*opt` | 解引用（无检查，空时为 UB） | `int x = *opt;` |
| `opt->member` | 访问成员的指针语法 | `opt->size();` |

#### 状态检查

| 操作 | 说明 | 示例 |
|------|------|------|
| `has_value()` | 是否包含值 | `if (opt.has_value())` |
| `operator bool()` | 隐式转换为 bool | `if (opt)` |
| `operator!()` | 与 nullopt 比较 | `if (!opt)` |

#### 比较操作

| 操作 | 说明 |
|------|------|
| `==`, `!=` | 两个 optional 比较，或与 `nullopt` 比较 |
| `<`, `<=`, `>`, `>=` | 按值比较（空值小于任何有值） |

#### Monadic 操作（C++23）

| 操作 | 说明 | 签名 |
|------|------|------|
| `and_then(F)` | f(value) 如果存在，否则 nullopt | F 返回 optional |
| `transform(F)` | f(value) 包装为 optional，否则 nullopt | F 返回任意类型 |
| `or_else(F)` | value 如果存在，否则 f() | F 返回 optional |

```cpp
// Monadic 链式调用示例（C++23）
std::optional<int> result = getValue()
    .and_then(validate)
    .transform([](int x) { return x * 2; })
    .or_else([] { return std::optional<int>(0); });
```

### 相关类型

| 类型 | 头文件 | 说明 |
|------|--------|------|
| `std::optional<T>` | `<optional>` | 可能有、也可能没有值的类型 |
| `std::variant<Ts...>` | `<variant>` | 类型安全的 union，存储多个类型中的一个 |
| `std::expected<T, E>` (C++23) | `<expected>` | 要么包含值 T，要么包含错误 E |
| `std::any` | `<any>` | 类型安全的 void*，可存储任意类型 |
