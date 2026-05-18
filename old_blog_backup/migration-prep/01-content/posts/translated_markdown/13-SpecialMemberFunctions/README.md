# 第13章: 特殊成员函数

> **CS106L Fall 2025 - Lecture 13: Special Member Functions**

## 目录

- [13.1 什么是特殊成员函数](#131-什么是特殊成员函数)
- [13.2 六大特殊成员函数](#132-六大特殊成员函数)
- [13.3 示例：Widget类](#133-示例widget类)
- [13.4 拷贝构造函数 vs 拷贝赋值运算符](#134-拷贝构造函数-vs-拷贝赋值运算符)
- [13.5 成员初始化列表](#135-成员初始化列表)
- [13.6 为什么需要重写SMF](#136-为什么需要重写smf)
- [13.7 深拷贝 vs 浅拷贝](#137-深拷贝-vs-浅拷贝)
- [13.8 删除特殊成员函数 (`= delete`)](#138-删除特殊成员函数-delete)
- [13.9 Rule of Zero / Three / Five](#139-rule-of-zero--three--five)
- [13.10 补充知识点](#1310-补充知识点)

---

## 13.1 什么是特殊成员函数

**特殊成员函数**（Special Member Functions, SMFs）是编译器在特定条件下自动生成的成员函数。它们控制着类对象的生命周期：创建、复制、赋值和销毁。

之前我们学过的**构造函数**和**析构函数**就是两种特殊成员函数——每次创建类的新实例时调用构造函数，当对象离开作用域时调用析构函数。

---

## 13.2 六大特殊成员函数

这些函数只在被调用时生成（且在你显式定义任何一个之前）：

| 编号 | SMF | 签名 | 用途 |
|------|-----|------|------|
| 1 | **默认构造函数** | `T()` | 无参数创建新对象 |
| 2 | **析构函数** | `~T()` | 对象离开作用域时清理资源 |
| 3 | **拷贝构造函数** | `T(const T&)` | 通过成员级拷贝创建新对象 |
| 4 | **拷贝赋值运算符** | `T& operator=(const T&)` | 将一个已存在对象赋值给另一个 |
| 5 | **移动构造函数** | `T(T&&)` | "窃取"临时对象的资源 |
| 6 | **移动赋值运算符** | `T& operator=(T&&)` | "窃取"临时对象的资源（赋值形式） |

本章重点讲解前四个，移动语义将在第14章深入讨论。

---

## 13.3 示例：Widget类

以 `Vector` 为例，回顾其默认构造函数：

```cpp
template <typename T>
Vector<T>::Vector()
{
  _size = 0;
  _capacity = 4;
  _data = new T[_capacity];
}
```

编译器还会自动生成其他SMF。但自动生成的版本只是做**成员级拷贝**（member-wise copy），这对于管理动态内存的类来说通常是不够的。

---

## 13.4 拷贝构造函数 vs 拷贝赋值运算符

### 调用时机的区别

```cpp
// 拷贝构造函数：用另一个对象初始化新对象
Widget widgetOne;
Widget widgetTwo = widgetOne;  // 拷贝构造函数被调用

// 拷贝赋值运算符：将已存在对象赋值给另一个已存在对象
Widget widgetOne;
Widget widgetTwo;
widgetOne = widgetTwo;  // 拷贝赋值运算符被调用
```

**关键区别**：
- 拷贝构造函数：`Widget widgetTwo = widgetOne;` （新对象被创建）
- 拷贝赋值运算符：`widgetOne = widgetTwo;` （两个对象都已存在）

在拷贝赋值运算符调用时，两个对象都已经构造完成，`=` 操作只是赋值而已。

---

## 13.5 成员初始化列表

### 问题

在构造函数体内初始化成员变量实际上经历了两个步骤：
1. 成员变量先被默认初始化
2. 然后被重新赋值

这是低效的，相当于做了两倍的工作。

```cpp
// 低效方式：两步（默认初始化 + 赋值）
template <typename T>
Vector<T>::Vector()
{
  _size = 0;          // 第2步：赋值
  _capacity = 4;      // 第2步：赋值
  _data = new T[_capacity];  // 第2步：赋值
}
```

### 解决方案：成员初始化列表

```cpp
// 高效方式：一步（直接初始化为目标值）
template <typename T>
Vector<T>::Vector() : _size(0), _capacity(4), _data(new T[_capacity]) { }
```

### 必须使用初始化列表的情况

1. **`const` 成员变量**
2. **引用成员变量**
3. **没有默认构造函数的成员对象**

```cpp
template <typename T>
class MyClass {
    const int _constant;    // const 成员
    int& _reference;        // 引用成员

public:
    // 只能用初始化列表来初始化 const 和引用成员
    MyClass(int value, int& ref)
        : _constant(value), _reference(ref) { }
};
```

`const` 成员和引用成员在初始化后不能重新赋值，所以构造函数体内的赋值是不允许的——必须使用初始化列表。

---

## 13.6 为什么需要重写SMF

编译器默认生成的SMF做的是**成员级拷贝**（member-wise copy）：

```cpp
// 编译器自动生成的拷贝构造函数等同于：
template <typename T>
Vector<T>::Vector(const Vector<T>& other)
    : _size(other._size), _capacity(other._capacity), _data(other._data) { }
```

**这个默认行为在类包含指针时是错误的！**

---

## 13.7 深拷贝 vs 浅拷贝

### 浅拷贝（Shallow Copy）的问题

```
默认拷贝（浅拷贝）：

  vec._data ------> [0][1][2][3][4] <------ copy._data
                          ^
                          |
                    两个指针指向同一块内存！

问题：修改一个会影响到另一个，而且会导致双重释放（double free）。
```

### 深拷贝（Deep Copy）

**深拷贝**：创建一个完整的、独立的副本。

```cpp
template <typename T>
Vector<T>::Vector(const Vector<T>& other)
    : _size(other._size), _capacity(other._capacity),
      _data(new T[other._capacity])      // 分配全新的内存！
{
    for (size_t i = 0; i < _size; ++i) {
        _data[i] = other._data[i];       // 逐个复制元素
    }
}
```

```
深拷贝：

  vec._data ------> [0][1][2][3][4]    (独立的)
  copy._data -----> [0][1][2][3][4]    (独立的)
```

**何时需要深拷贝**：当类管理动态分配的内存（或其他需要手动管理的资源）时，必须重写拷贝构造函数和拷贝赋值运算符来实现深拷贝。

---

## 13.8 删除特殊成员函数 (`= delete`)

### 阻止拷贝

有时你希望某些操作完全不可用——例如一个管理密码的类不应该被拷贝。

```cpp
class PasswordManager {
public:
    // 删除拷贝构造函数和拷贝赋值运算符
    PasswordManager(const PasswordManager&) = delete;
    PasswordManager& operator=(const PasswordManager&) = delete;
};
```

**`= delete`** 移除该函数的任何功能，任何试图使用它的代码都会产生编译错误。

### 使用场景

- 只允许一个实例存在的类（如单例模式）
- 管理独占资源的类
- `std::unique_ptr` 就是通过 `= delete` 来禁止拷贝的

---

## 13.9 Rule of Zero / Three / Five

### Rule of Zero（零法则）

**如果默认的SMF工作正常，就不要定义自己的版本。**

如果你的类只包含"自我管理"的成员变量（如 `std::string`, `std::vector`, `int` 等STL类型或值类型），它们自己的SMF已经实现好了，编译器生成的版本就能正确工作。

```cpp
class Post {
    Photo photo;          // Photo 有自己的 SMF
    std::string caption;  // std::string 有自己的 SMF
    int likes;            // int 的 SMF 很简单
};
// 完全不需要定义任何 SMF！Rule of Zero！
```

### Rule of Three（三法则）

**如果你需要自定义析构函数，那么你几乎一定也需要自定义拷贝构造函数和拷贝赋值运算符。**

为什么？如果你需要析构函数，通常意味着你在手动管理内存/资源。编译器不知道如何正确拷贝这些资源，所以你必须自己实现。

```
Rule of Three：如果需要以下任何一个，就需要全部三个：
  - 析构函数
  - 拷贝构造函数
  - 拷贝赋值运算符
```

### Rule of Five（五法则）

**如果你需要自定义析构函数、拷贝构造/赋值、移动构造/赋值中的任何一个，那么你很可能需要自定义全部五个。**

这不是强制的，但如果不定义移动操作，代码会变慢——容器和算法将退化为拷贝而非移动。

```
Rule of Five：如果需要任何一个，可能全部五个都需要：
  - 析构函数
  - 拷贝构造函数
  - 拷贝赋值运算符
  - 移动构造函数    (可选但推荐)
  - 移动赋值运算符   (可选但推荐)
```

---

## 13.10 补充知识点

### 1. Pop Quiz：识别各种初始化方式

```cpp
vector<int> func(vector<int> vec0) {     // vec0: 拷贝构造函数
    vector<int> vec1;                     // vec1: 默认构造函数
    vector<int> vec2(3);                  // vec2: 自定义构造函数 (非SMF)
    vector<int> vec3{3};                  // vec3: 统一初始化 (非SMF)
    vector<int> vec4();                   // vec4: 等等！这是函数声明！不是对象！
    vector<int> vec5(vec2);              // vec5: 拷贝构造函数
    vector<int> vec6{};                   // vec6: 空初始化列表
    vector<int> vec7{static_cast<int>(vec2.size() + vec6.size())};
                                          // vec7: 列表初始化
    vector<int> vec8 = vec2;             // vec8: 拷贝构造函数
    vec8 = vec2;                          // 拷贝赋值运算符
    return vec8;                          // 返回：移动构造函数（或 NRVO 优化）
}
```

**特别警告**：`vector<int> vec4();` 被解析为函数声明（声明一个返回 `vector<int>` 的函数 `vec4`，无参数），而不是变量定义！这被称为"最令人烦恼的解析"（Most Vexing Parse）。

### 2. 自赋值安全性

在实现拷贝赋值运算符时，务必检查自赋值：

```cpp
MyClass& MyClass::operator=(const MyClass& other) {
    if (this == &other) return *this;  // 防止自赋值

    delete[] data;                      // 清理旧资源
    data = new int[other.size];         // 分配新资源
    std::copy(other.data, other.data + other.size, data);
    size = other.size;
    return *this;
}
```

如果没有自赋值检查，`obj = obj;` 会先 `delete` 自己的数据，然后试图从已被删除的数据中拷贝——结果是未定义行为。

### 3. 复制并交换 (Copy-and-Swap) 惯用法

一种更优雅的实现赋值运算符（同时处理自赋值和异常安全）的方式：

```cpp
class MyClass {
public:
    friend void swap(MyClass& a, MyClass& b) noexcept {
        std::swap(a.data, b.data);
        std::swap(a.size, b.size);
    }

    MyClass& operator=(MyClass other) {  // 按值传参——创建副本
        swap(*this, other);  // 交换 *this 和副本
        return *this;         // 副本离开作用域时自动清理旧资源
    }
};
```

这种模式自动提供了强异常安全保证和自赋值安全性。

### 4. `= default`

如果删除了某些SMF，但希望恢复编译器生成的默认版本，可以使用 `= default`：

```cpp
class MyClass {
public:
    MyClass() = default;                      // 使用编译器生成的默认构造函数
    MyClass(const MyClass&) = delete;         // 但是禁止拷贝
    MyClass(MyClass&&) = default;             // 移动使用默认版本
    ~MyClass() = default;                     // 析构函数使用默认版本
};
```

### 5. 析构函数与虚函数

如果类可能被继承，析构函数通常应该声明为 `virtual`：

```cpp
class Base {
public:
    virtual ~Base() = default;  // 虚析构函数
};

class Derived : public Base {
    // 通过Base指针删除时，正确调用Derived的析构函数
};
```

---

> **本章总结**：特殊成员函数控制着对象的生命周期。理解默认构造函数、析构函数、拷贝构造函数和拷贝赋值运算符的自动生成行为，以及何时需要重写它们，是编写正确C++程序的基础。Rule of Zero 告诉我们"不要做多余的事"，Rule of Three 告诉我们"需要析构函数时，拷贝也要自己管"，Rule of Five 告诉我们"需要拷贝时，移动也一起做了"。这些法则是指南，帮助我们在资源管理的复杂性中找到平衡。
