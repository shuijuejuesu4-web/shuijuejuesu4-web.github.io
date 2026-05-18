# 第12章: 运算符重载

> **CS106L Fall 2025 - Lecture 12: Operator Overloading**

## 目录

- [12.1 为什么需要运算符重载](#121-为什么需要运算符重载)
- [12.2 哪些运算符可以/不可以重载](#122-哪些运算符可以不可以重载)
- [12.3 成员运算符重载](#123-成员运算符重载)
- [12.4 非成员运算符重载](#124-非成员运算符重载)
- [12.5 `friend` 关键字](#125-friend-关键字)
- [12.6 运算符重载的设计哲学](#126-运算符重载的设计哲学)
- [12.7 常用运算符重载模式](#127-常用运算符重载模式)
- [12.8 补充知识点](#128-补充知识点)

---

## 12.1 为什么需要运算符重载

### 问题背景

在C++中，`std::map<K, V>` 要求键类型 `K` 必须支持 `operator<`。为什么？因为 `std::map` 内部使用红黑树，需要通过 `<` 比较来进行查找和排序。

```cpp
std::map<StanfordID, std::string> studentMap;
// 编译错误！StanfordID 没有定义 operator<
```

更广泛地说，运算符重载使我们能够为自定义类型赋予"原生"的操作语义。

> **核心哲学**："运算符允许你传达关于类型的含义，这是普通函数无法做到的。" (来源：CppCon演讲)

对比以下两种方式：

```cpp
// 不好：看起来像是随机函数调用，看不出"加法"的含义
Money result = add(money1, money2);

// 好：使用运算符，自然传达"Money 具有数值般行为"的含义
Money result = money1 + money2;
```

---

## 12.2 哪些运算符可以/不可以重载

### 可以重载的大多数运算符

C++中绝大多数运算符都可以重载：

`+`, `-`, `*`, `/`, `%`, `^`, `&`, `|`, `~`, `!`, `=`, `<`, `>`, `+=`, `-=`, `*=`, `/=`, `%=`, `^=`, `&=`, `|=`, `<<`, `>>`, `<<=`, `>>=`, `==`, `!=`, `<=`, `>=`, `&&`, `||`, `++`, `--`, `,`, `->*`, `->`, `()`, `[]`, `new`, `new[]`, `delete`, `delete[]`

### 不可以重载的运算符

| 运算符 | 名称 | 原因 |
|--------|------|------|
| `::` | 作用域解析 | 语言核心机制 |
| `? :` | 三元条件 | 唯一的三目运算符，语法特殊 |
| `.` | 成员访问 | 语言核心机制 |
| `.*` | 指向成员的指针访问 | 使用频率低，语法特殊 |
| `sizeof()` | 对象大小 | 编译期常量 |
| `typeid()` | 类型信息 | 运行时类型识别核心 |
| `cast()` | 类型转换 | 强制类型转换核心 |

---

## 12.3 成员运算符重载

### 语法

```cpp
// 在类内部声明
return_type operator<symbol>(parameter_list);

// 对于二元运算符，成员函数接受一个参数（右操作数），左操作数是 this
```

### 完整示例

假设我们有一个 `StanfordID` 类，想按 `idNumber` 比较两个对象。

**头文件 (`StanfordID.h`)**：

```cpp
class StanfordID {
private:
    std::string name;
    std::string sunet;
    int idNumber;

public:
    StanfordID(std::string name, std::string sunet, int idNumber);
    int getIdNumber() const;

    // 成员运算符重载
    bool operator<(const StanfordID& other) const;
};
```

**实现文件 (`StanfordID.cpp`)**：

```cpp
#include "StanfordID.h"

int StanfordID::getIdNumber() const {
    return idNumber;
}

// 方式一：通过公共 getter 访问
bool StanfordID::operator<(const StanfordID& other) const {
    return idNumber < other.getIdNumber();
}

// 方式二：直接访问私有成员（成员函数本来就能访问）
bool StanfordID::operator<(const StanfordID& other) const {
    return idNumber < other.idNumber;
}
```

**使用**：

```cpp
StanfordID rachel("Rachel", "rfern", 12345);
StanfordID thomas("Thomas", "tpoimen", 67890);

auto minID = std::min(rachel, thomas);  // 现在可以用了！
// 编译器内部调用: rachel < thomas -> operator<
```

### `const` 的重要性

运算符重载通常应该声明为 `const`，因为比较操作不应修改对象状态：

```cpp
bool operator<(const StanfordID& other) const;
//                                         ^^^^^
//                                         这个 const 表示成员函数不修改 *this
```

---

## 12.4 非成员运算符重载

### 为什么需要非成员重载？

成员重载有一个局限：左操作数必须是类的实例。非成员重载更灵活。

**非成员重载的优势**：

1. **允许左操作数为非类类型**（例如 `5 + myObject`）
2. **允许为不拥有的类重载运算符**（可以对STL类型和自定义类型之间的操作进行重载）
3. **更加符合STL的习惯用法**，是现代C++的推荐做法

### 非成员重载语法

```cpp
// 非成员运算符重载（推荐方式）
bool operator<(const StanfordID& lhs, const StanfordID& rhs);

// 注意：两个操作数都作为参数传入
```

### 对比

```cpp
// 非成员版本
bool operator<(const StanfordID& lhs, const StanfordID& rhs);

// 成员版本
bool StanfordID::operator<(const StanfordID& rhs) const { ... }
```

**最好的做法**：使用非成员函数重载运算符，这样可以在两个方向上进行有意义的比较，而且不需要修改别人的类。

---

## 12.5 `friend` 关键字

### 问题

非成员函数无法访问类的私有成员：

```cpp
// 编译错误！非成员函数不能访问 private 成员
bool operator<(const StanfordID& lhs, const StanfordID& rhs) {
    return lhs.idNumber < rhs.idNumber;  // idNumber 是 private！
}
```

### 解决方案一：使用公共getter

```cpp
bool operator<(const StanfordID& lhs, const StanfordID& rhs) {
    return lhs.getIdNumber() < rhs.getIdNumber();  // 使用公共接口
}
```

### 解决方案二：`friend` 关键字

`friend` 关键字允许非成员函数或类访问另一个类的私有成员。

**头文件**：

```cpp
class StanfordID {
private:
    std::string name;
    std::string sunet;
    int idNumber;

public:
    StanfordID(std::string name, std::string sunet, int idNumber);

    // 声明非成员函数为友元
    friend bool operator<(const StanfordID& lhs, const StanfordID& rhs);
};
```

**实现文件**：

```cpp
bool operator<(const StanfordID& lhs, const StanfordID& rhs) {
    return lhs.idNumber < rhs.idNumber;  // 现在可以访问 private 成员了！
}
```

**注意**：如果有公共getter方法，就不需要 `friend`。`friend` 只在必须直接访问私有成员时使用。

---

## 12.6 运算符重载的设计哲学

### 最小惊讶原则 (Principle of Least Astonishment, PoLA)

运算符的含义应该显而易见：
- 重载的运算符行为应该与内置类型对应操作相似
- 不要用 `operator+` 来做集合减法
- 如果含义不够明显，应该使用命名函数而不是运算符

### 对立法则 (Rule of Contrariety)

当你定义一个运算符时，应该同时定义其"对立"运算符：

```cpp
bool StanfordID::operator==(const StanfordID& other) const {
   return (name == other.name) &&
          (sunet == other.sunet) &&
          (idNumber == other.idNumber);
}

// 对立法则：定义了 == 就应该定义 !=
bool StanfordID::operator!=(const StanfordID& other) const {
   return !(*this == other);
}
```

### 关于 `<<` 流插入运算符

`<<` 运算符的实现方式会影响使用体验：

```cpp
// 紧凑型
std::ostream& operator<<(std::ostream& out, const StanfordID& sid) {
    out << sid.name << " " << sid.sunet << " " << sid.idNumber;
    return out;
}

// 可读型
std::ostream& operator<<(std::ostream& out, const StanfordID& sid) {
    out << "Name: " << sid.name
        << " sunet: " << sid.sunet
        << " idnumber: " << sid.idNumber;
    return out;
}
```

**注意**：`<<` 和 `>>` 必须为非成员函数（因为左操作数是 `std::ostream&`/`std::istream&`，不是你自定义的类型）。

### 最终建议

1. 运算符重载为对象解锁了新的功能和含义层次
2. 运算符应该有意义——重点在于传达类型本身的含义
3. **按需重载**：如果你不在流中使用你的类型，就不需要重载 `<<` 或 `>>`
4. 不能同时定义成员和非成员版本的同一个运算符（会造成歧义）

---

## 12.7 常用运算符重载模式

### 比较运算符

```cpp
class Fraction {
    int num, den;
public:
    // 通常只需要定义 < 和 ==，其余可以通过它们推导
    bool operator<(const Fraction& rhs) const {
        return num * rhs.den < rhs.num * den;
    }

    bool operator==(const Fraction& rhs) const {
        return num == rhs.num && den == rhs.den;
    }

    bool operator!=(const Fraction& rhs) const { return !(*this == rhs); }
    bool operator>(const Fraction& rhs) const  { return rhs < *this; }
    bool operator<=(const Fraction& rhs) const { return !(rhs < *this); }
    bool operator>=(const Fraction& rhs) const { return !(*this < rhs); }
};
```

### 算术运算符

```cpp
class Fraction {
public:
    // 复合赋值运算符（成员函数）
    Fraction& operator+=(const Fraction& rhs) {
        num = num * rhs.den + rhs.num * den;
        den *= rhs.den;
        simplify();
        return *this;
    }

    // 二元算术运算符（非成员函数，通过 += 实现）
    friend Fraction operator+(Fraction lhs, const Fraction& rhs) {
        lhs += rhs;  // 利用 += 的已有逻辑
        return lhs;
    }
};
```

**模式说明**：用 `+=` 实现 `+`，用 `-=` 实现 `-`，以此类推。这样可以复用代码，减少错误。

### 下标运算符 `[]`

```cpp
template <typename T>
class Vector {
    T* _data;
    size_t _size;
public:
    // 可读写版本
    T& operator[](size_t index) { return _data[index]; }

    // 只读版本（const 重载）
    const T& operator[](size_t index) const { return _data[index]; }
};
```

### 递增/递减运算符

```cpp
class Iterator {
public:
    // 前置 ++  (++it)
    Iterator& operator++() {
        ++ptr;
        return *this;
    }

    // 后置 ++  (it++)——注意额外的 int 参数用于区分重载
    Iterator operator++(int) {
        Iterator tmp = *this;
        ++ptr;
        return tmp;
    }
};
```

### 函数调用运算符 `()`——创建Functor

```cpp
struct MultiplyBy {
    int factor;
    MultiplyBy(int f) : factor(f) {}
    int operator()(int x) const { return x * factor; }
};

MultiplyBy times3(3);
int result = times3(10);  // 30 —— 像函数一样使用！
```

---

## 12.8 补充知识点

### 1. 运算符重载与三路比较 (C++20 `<=>`)

C++20 引入了"飞船运算符" `<=>`（三路比较），可以自动生成所有比较运算符：

```cpp
#include <compare>

class StanfordID {
    int idNumber;
public:
    // 定义 <=> 后编译器自动生成 <, <=, >, >=  ==, !=
    auto operator<=>(const StanfordID&) const = default;
};
```

### 2. `std::rel_ops` (已废弃但值得了解)

`std::rel_ops` 曾经提供从 `<` 和 `==` 自动生成其他比较运算符的方法，但在C++20中已被 `<=>` 取代。

### 3. 运算符重载与隐式转换

注意运算符重载与隐式转换的交互。例如，如果你有一个接受 `const char*` 的构造函数（非 `explicit`），可能会发生意外的隐式转换：

```cpp
class String {
public:
    String(const char* s);  // 非 explicit
    friend bool operator==(const String& a, const String& b);
};

String s("hello");
if (s == "world")  // OK: "world" 隐式转换为 String
```

建议：对于单参数构造函数，使用 `explicit` 来防止意外的隐式转换。

### 4. 返回类型优化

```cpp
// 不好：返回副本（额外的拷贝）
Fraction operator+(const Fraction& a, const Fraction& b) {
    Fraction result = a;
    result += b;
    return result;  // 可能有两次拷贝
}

// 好：按值传第一个参数，利用移动语义
Fraction operator+(Fraction a, const Fraction& b) {
    a += b;
    return a;  // 编译器可以优化（RVO/NRVO）
}
```

### 5. 常见运算符对

| 如果定义 | 也应该定义 | 原因 |
|----------|-----------|------|
| `==` | `!=` | 对立关系 |
| `<` | `>`, `<=`, `>=` | 全序关系 |
| `+` | `+=` | 复合赋值优先 |
| `-` | `-=` | 复合赋值优先 |
| `*` | `*=` | 复合赋值优先 |
| `[]` | `const []` | const 正确性 |
| `<<` (输出) | `>>` (输入) | 对称性 |

---

> **本章总结**：运算符重载是C++中一项强大的功能，它允许自定义类型表现得像内置类型一样自然。关键是要遵循最小惊讶原则——运算符的行为应该与人们对其符号的直觉理解一致。优先使用非成员函数重载（配合 `friend` 或公共接口），这不仅是STL的习惯做法，也更灵活。定义比较运算符时，考虑使用C++20的 `<=>` 运算符来自动生成全套比较操作。
