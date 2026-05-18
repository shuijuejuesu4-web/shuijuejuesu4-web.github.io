---
title: "第9章：模板类 (Template Classes)"
description: "第9章：模板类 (Template Classes)"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第9章：模板类 (Template Classes)

> Stanford CS106L, Fall 2025 -- Rachel Fernandez and Thomas Poimenidis

---

## 目录 (Table of Contents)

1. [为什么需要模板? (What Are Templates?)](#1-为什么需要模板)
2. [模板类基础 (Template Classes)](#2-模板类基础)
3. [模板实例化 (Template Instantiation)](#3-模板实例化)
4. [模板类的实现细节](#4-模板类的实现细节)
5. [三个模板 "怪癖"](#5-三个模板怪癖)
6. [非类型模板参数](#6-非类型模板参数)
7. [Const 正确性 (Const Correctness)](#7-const-正确性)
8. [const 重载与 const_cast](#8-const-重载与-const_cast)
9. [mutable 关键字](#9-mutable-关键字)
10. [本章回顾](#10-本章回顾)
11. [补充知识点](#11-补充知识点)

---

## 1. 为什么需要模板?

### 1.1 问题场景

假设你需要存储整数列表，于是写了一个 `IntVector`：

```cpp
class IntVector {
public:
    IntVector();
    ~IntVector();
    size_t size();
    bool empty();
    void push_back(const int& elem);
    int& operator[](size_t index);
};
```

然后你需要存储 `double` 列表，于是又写了 `DoubleVector`。接下来你需要存储字符串、自定义类型、vector的vector......

```cpp
class IntVector {       class DoubleVector {       class StringVector {
    // 存储int的代码       // 存储double的代码        // 存储string的代码
};                      };                          };
```

**大量重复代码！** 逻辑完全相同，只是类型不同。

### 1.2 模板的优雅方案

如果能保持逻辑不变，只改变类型，那该多好！

```cpp
template <typename T>
class vector {
    // 一份代码，所有类型
};

vector<int> v1;
vector<double> v2;
vector<string> v3;
```

---

## 2. 模板类基础

### 2.1 模板简史

早期 C++ 程序员使用**预处理器宏 (Preprocessor Macros)** 来生成不同类型版本的代码：

```cpp
#define GENERATE_VECTOR(MY_TYPE)            \
  class MY_TYPE##Vector {                  \
  public:                                   \
      MY_TYPE& at(size_t index);           \
      void push_back(const MY_TYPE& elem); \
  private:                                  \
      MY_TYPE* elems;                      \
      size_t logical_size;                 \
      size_t array_size;                   \
  };

// 使用：
GENERATE_VECTOR(int)
intVector v1;
v1.push_back(5);
```

**宏的问题**：
- 笨拙的语法
- 难以进行类型检查
- 容易忘记调用宏或多次调用
- 错误信息难以理解

### 2.2 现代模板语法

```cpp
template <typename T>          // 模板声明
class Vector {                 // Vector 是一个接受类型名 T 的模板
public:
    T& at(size_t index);
    void push_back(const T& elem);
private:
    T* elems;                  // 当 Vector 被实例化时，T 被替换
};
```

---

## 3. 模板实例化 (Template Instantiation)

### 3.1 基本概念

**实例化**是指编译器根据具体类型按需生成代码的过程：

```cpp
Vector<int> intVec;                    // 编译器生成 IntVector
Vector<double> doubleVec;              // 编译器生成 DoubleVector
Vector<std::string> strVec;            // 编译器生成 StringVector
Vector<Vector<int>> vecVec;            // 嵌套！
struct MyCustomType {};
Vector<MyCustomType> structVec;        // 支持自定义类型！

// 编译器行为等价于：
// 你写 Vector<int> v;  --> 编译器生成 class IntVector { int& at(...); ... };
```

### 3.2 模板 vs 类型 -- 关键区别

```cpp
template <typename T>
class Vector            // 这是模板 (template)，不是类型 (type)

Vector<std::string>     // 这是类型 (type)，也叫模板实例化 (template instantiation)
```

**重要区别**：`Vector<double>` 和 `Vector<int>` 是两个**完全不同的类型**（无论在编译时还是运行时）：

```cpp
void foo(std::vector<int> v);

int main() {
    std::vector<double> v;
    foo(v);  // ❌ 编译错误：无法将 vector<double> 转换为 vector<int>
}
```

> 思考：与Java对比 -- Java中 `ArrayList<Integer>` 和 `ArrayList<Double>` 共享相同的运行时类型（类型擦除）。

---

## 4. 模板类的实现细节

### 4.1 正常类的文件组织

对于非模板类：

```
// StrVector.h                          // StrVector.cpp
class StrVector {                       #include "StrVector.h"
public:
    string& at(size_t i);               string& StrVector::at(size_t i) {
};                                           // 实现...
                                        }
```

### 4.2 模板类的文件组织（与众不同！）

对于模板类，**.h 文件必须包含 .cpp 文件**：

```
// Vector.h                             // Vector.cpp
template <typename T>                   template <typename T>
class Vector {                          T& Vector<T>::at(size_t i) {
public:                                     // 实现...
    T& at(size_t i);                    }
};

#include "Vector.cpp"    // 注意：在 .h 底部包含 .cpp！
```

**原因**：由于模板代码生成在编译器和链接器中的实现方式，模板定义必须在实例化时可见。头文件必须包含完整的实现。

### 4.3 实现模板成员函数

在 `.cpp` 文件中实现模板成员函数时，需要重复模板声明，并使用 `Vector<T>` 而非 `Vector`：

```cpp
// Vector.cpp

// ✅ 正确：包含模板声明 + 使用 Vector<T>
template <typename T>
T& Vector<T>::at(size_t i) {
    // 实现...
}

// ❌ 错误：缺少模板声明
T& Vector::at(size_t i) {  // 编译器：我不知道 T 是什么！
}

// ❌ 错误：Vector 不是类型，Vector<T> 才是
```

---

## 5. 三个模板怪癖

### 5.1 怪癖一：.cpp 中必须复制 `template <...>` 语法

如上节所述，每个成员函数实现前必须写 `template <typename T>`。

### 5.2 怪癖二：.h 必须在文件底部包含 .cpp

这是编译器/链接器实现方式的限制。有一些方法可以绕过（例如将实现直接写在头文件中）。

### 5.3 怪癖三：`typename` 等价于 `class`

以下四种写法**完全相同**：

```cpp
template <typename K, typename V>    template <class K, class V>
struct pair;                         struct pair;

template <class K, typename V>       template <typename K, class V>
struct pair;                         struct pair;
```

两者可以混用。`typename` 和 `class` 在模板参数声明中语义完全等价。

---

## 6. 非类型模板参数

模板不仅可以接受类型，还可以接受**值**作为参数：

```cpp
// 接受 size_t 值作为模板参数
template <size_t N>
class SizeTemplate {};
SizeTemplate<5> s;          // N = 5

// 接受 bool 值
template <bool B>
class BoolTemplate {};
BoolTemplate<true> b;       // B = true
```

### 6.1 `std::array` -- 经典例子

```cpp
template<typename T, std::size_t N>
struct std::array { /* ... */ };

// 一个恰好包含 5 个 string 的数组
std::array<std::string, 5> arr;
```

**为什么使用 array 而非 vector？**

- `array` 避免了堆分配
- 编译器确切知道 `array<string, 5>` 占用多少空间（大小编译进类型）
- 可以**栈分配**，更快、对缓存更友好

**关键代价**：数组大小是类型的一部分！`array<int, 5>` 和 `array<int, 6>` 是不同的类型。

---

## 7. Const 正确性 (Const Correctness)

### 7.1 问题

```cpp
void printVec(const Vector<int>& v) {
    for (size_t i = 0; i < v.size(); i++) {
        std::cout << v.at(i) << " ";
    }
}  // ❌ 编译错误：const Vector<int> 没有 size 方法！
```

问题分析：
- 将 `v` 传递为 `const`，承诺不修改 `v`
- 编译器无法保证 `size()` 和 `at()` 这样的方法不会修改 `v`
- 成员函数可以访问成员变量 -- 编译器必须保守

### 7.2 解决方案：const 成员函数

```cpp
template<class T>
class Vector {
public:
    size_t size() const;           // const 方法：承诺不修改对象
    bool empty() const;            // 编译器会强制执行这个承诺

    T& operator[] (size_t index);
    T& at(size_t index) const;     // const 方法
    void push_back(const T& elem);
};
```

const 方法的含义："亲爱的编译器，我承诺不在这个方法内部修改对象。请监督我。"

### 7.3 实现 const 成员函数

```cpp
template <class T>
size_t Vector<T>::size() const {
    // this->logical_size = 106;  // ❌ 编译错误！
    return logical_size;           // ✅ 只读访问
}
```

在 const 方法内部，`this` 的类型是 `const Vector<T>*`，而非 `Vector<T>*`。

### 7.4 const 接口

| | `Vector<T>` | `const Vector<T>` |
|---|------------|-------------------|
| 可调用的方法 | 所有方法 | 仅 const 方法 |
| 成员变量类型 | `T* elems` 等 | `const T* elems` 等 |

const 对象只能使用 **const 接口** -- 即标记为 `const` 的函数。

### 7.5 问题一：const 消费者仍可修改！

```cpp
T& at(size_t index) const;  // 返回非const引用！

void oops(const Vector<int>& v) {
    v.at(0) = 42;           // ⚠️ 我们可以修改 const Vector！
    // const 承诺被破坏了！
}
```

**解决方案**：返回 const 引用：

```cpp
const T& at(size_t index) const;  // 返回 const 引用
```

### 7.6 问题二：非const消费者无法修改

```cpp
const T& at(size_t index) const;  // 现在非const Vector 也只能读！

void ooh(Vector<int>& v) {
    v.at(0) = 42;                  // ❌ 不能赋值给 const int&
}
```

---

## 8. const 重载与 const_cast

### 8.1 解决方案：const 重载

定义两个版本的 `at` 方法 -- 编译器根据对象的const性自动选择：

```cpp
template<class T>
class Vector {
public:
    const T& at(size_t index) const;  // 对 const Vector 调用这个
    T& at(size_t index);              // 对非 const Vector 调用这个
};
```

### 8.2 实现

```cpp
template <class T>
const T& Vector<T>::at(size_t index) const {
    return elems[index];
}

template <class T>
T& Vector<T>::at(size_t index) {
    return elems[index];
}
```

两个方法实现相同 -- 对于一行代码来说可以接受，但如果有更多逻辑（如 `findElement`）：

```cpp
template<class T>
class Vector {
public:
    T& at(size_t index);
    const T& at(size_t index) const;
    T& findElement(const T& value);
    const T& findElement(const T& value) const;  // 庞大冗余！
};
```

### 8.3 `const_cast` -- 消除 const 重载冗余

```cpp
template <typename T>
T& Vector<T>::findElement(const T& value) {
    for (size_t i = 0; i < logical_size; i++) {
        if (elems[i] == value) return elems[i];
    }
    throw std::out_of_range("Element not found");
}

// const 版本委托给非const版本
template <typename T>
const T& Vector<T>::findElement(const T& value) const {
    return const_cast<Vector<T>&>(*this).findElement(value);
}
```

### 8.4 const_cast 详解

```cpp
const_cast<Vector<T>&>(*this).findElement(value);

// const_cast<Vector<T>&>: 移除 const 属性
// *this: 解引用 const Vector<T>*，得到 const Vector<T>&
// 结果类型: Vector<T>& （非const引用）
// .findElement(value): 调用非const版本
```

`const_cast` 告诉编译器："别担心，我有分寸" -- 强制选择非const重载。

### 8.5 何时使用 const_cast？

**简短回答**：几乎永远不需要。

- `const_cast` 绕过编译器的保护
- 如果你需要可变值，一开始就不要用 `const`
- `const_cast` 的合法使用场景非常少（消除 const 重载冗余是少数例外之一）

---

## 9. `mutable` 关键字

比 `const_cast` 更细粒度的选择：

```cpp
struct MutableStruct {
    int dontTouchThis;
    mutable double iCanChange;  // 即使在 const 对象中也可以修改
};

const MutableStruct cm;
// cm.dontTouchThis = 42;    // ❌ 不允许，cm 是 const
cm.iCanChange = 3.14;         // ✅ 允许，iCanChange 是 mutable
```

### 9.1 mutable 的实际应用：调试信息

```cpp
struct CameraRay {
    Point origin;
    Direction direction;
    mutable Color debugColor;    // 调试用：即使在 const 函数中也可修改
};

void renderRay(const CameraRay& ray) {
    ray.debugColor = Color::Yellow;  // 显示调试光线
    // 渲染逻辑...
}
```

**使用建议**：`mutable` 和 `const_cast` 一样，应该**谨慎使用**。合理场景包括：
- 缓存/记忆化（memoization）
- 调试/日志元数据
- 互斥锁（mutex）成员（锁定操作需要修改mutex，但从概念上不改变对象状态）

---

## 10. 本章回顾

| 概念 | 要点 |
|------|------|
| **模板类** | 模板类将逻辑泛化到不同类型；消除代码冗余 |
| **const 正确性** | `const` 使整个对象只读；将不修改对象的方法标记为 `const` |
| **const 重载** | 为 const 和非 const 对象提供不同版本；`const_cast` 可消除实现冗余 |
| **mutable** | 允许 const 对象中特定成员可变；在极少数场景下绕过 const 保护 |

---

## 11. 补充知识点

### 11.1 模板特化 (Template Specialization)

可以为特定类型提供专门的实现：

```cpp
// 通用模板
template <typename T>
class Vector {
    // 通用实现
};

// 全特化：为 bool 提供特殊实现（优化存储）
template <>
class Vector<bool> {
    // 每个 bool 只占 1 位而非 1 字节
    // 特殊实现...
};
```

`std::vector<bool>` 就是一个著名的特化例子（也因其行为差异而臭名昭著）。

### 11.2 偏特化 (Partial Specialization)

```cpp
// 通用模板
template <typename T, typename U>
struct Pair {
    T first;
    U second;
};

// 偏特化：两个类型相同时
template <typename T>
struct Pair<T, T> {
    T first;
    T second;
    // 可以有不同的实现
};
```

### 11.3 模板模板参数

```cpp
template <typename T, template <typename> class Container>
class Stack {
    Container<T> data;
public:
    void push(const T& elem) { data.push_back(elem); }
    T pop() { auto elem = data.back(); data.pop_back(); return elem; }
};

Stack<int, std::vector> vecStack;
Stack<int, std::deque> dequeStack;
```

### 11.4 默认模板参数

```cpp
template <typename T, typename Container = std::deque<T>>
class Stack {
    Container data;
    // ...
};

Stack<int> s1;  // 默认使用 deque
Stack<int, std::vector<int>> s2;  // 显式指定 vector
```

### 11.5 成员函数模板

```cpp
template <typename T>
class Vector {
public:
    // 成员函数本身也可以是模板
    template <typename InputIt>
    void assign(InputIt first, InputIt last) {
        clear();
        while (first != last) {
            push_back(*first++);
        }
    }
};

std::set<int> s {1, 2, 3};
Vector<int> v;
v.assign(s.begin(), s.end());  // T=int, InputIt=set<int>::iterator
```

### 11.6 别名模板 (Alias Templates, C++11)

```cpp
// 使用 using 创建模板别名
template <typename T>
using StringMap = std::map<std::string, T>;

StringMap<int> m1;          // 等价于 std::map<std::string, int>
StringMap<double> m2;       // 等价于 std::map<std::string, double>

// typedef 做不到这一点
```

### 11.7 变量模板 (Variable Templates, C++14)

```cpp
template <typename T>
constexpr T pi = T(3.1415926535897932385);

double circumference = 2 * pi<double> * radius;
float circumference_f = 2 * pi<float> * radius_f;
```

### 11.8 SFINAE -- "替换失败不是错误"

```cpp
// 仅当 T 有 size() 方法时此重载才有效
template <typename T>
auto len(const T& t) -> decltype(t.size()) {
    return t.size();
}

// 对于没有 size() 的类型，回退到此重载
template <typename T>
size_t len(const T& t) {
    return std::distance(std::begin(t), std::end(t));
}
```

SFINAE（Substitution Failure Is Not An Error）是C++模板系统的核心原则之一。C++20的 concepts 使这一技术更加直观。
