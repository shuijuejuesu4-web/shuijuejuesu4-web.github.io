# 第7章：类与继承 (Classes and Inheritance)

> Stanford CS106L, Fall 2025 -- Thomas Poimenidis, Rachel Fernandez

---

## 目录 (Table of Contents)

1. [为什么需要类? (Why Classes?)](#1-为什么需要类)
2. [struct 与 class 的对比](#2-struct-与-class-的对比)
3. [头文件与源文件](#3-头文件与源文件)
4. [类的设计](#4-类的设计)
5. [构造函数](#5-构造函数)
6. [析构函数](#6-析构函数)
7. [成员函数实现](#7-成员函数实现)
8. [类型别名](#8-类型别名)
9. [继承 (Inheritance)](#9-继承)
10. [继承类型](#10-继承类型)
11. [菱形问题 (The Diamond Problem)](#11-菱形问题)
12. [本章回顾](#12-本章回顾)
13. [补充知识点](#13-补充知识点)

---

## 1. 为什么需要类?

### 1.1 C语言的局限

C语言没有对象的概念：
- 无法封装数据和操作这些数据的函数
- 无法使用面向对象编程 (OOP) 的设计模式

### 1.2 什么是面向对象编程?

- 面向对象编程以**对象**为中心
- 侧重于类的设计与实现
- **类**是用户定义的类型，可以声明为对象

### 1.3 惊喜：STL容器就是类！

```cpp
std::vector<int>      // vector 是一个类
std::map<std::string, int> // map 是一个类
std::set<double>      // set 是一个类
```

---

## 2. struct 与 class 的对比

### 2.1 回顾 `struct`

```cpp
struct StanfordID {
    std::string name;  // 字段 (fields)
    std::string sunet;
    int idNumber;
};

StanfordID s;
s.name = "Thomas Poimenidis";
s.sunet = "tpoimen";
s.idNumber = 01243425;
```

`struct` 的问题：
- **所有字段都是公开的 (public)**，用户可以直接修改
- **没有访问控制** -- 任何人都可以设置不合法的值：

```cpp
s.idNumber = -123451234512345;  // 这合理吗？
```

### 2.2 引入 `class`

```cpp
class ClassName {
private:
    // 私有成员：只有类内部可以访问

public:
    // 公共成员：所有人都可以访问
};
```

类和结构体的关键区别：

| 特性 | `struct` | `class` |
|------|----------|---------|
| 默认访问权限 | public | private |
| 成员函数 | 可以 | 可以 |
| 访问控制 (public/private/protected) | 可以 | 可以 |
| 继承默认 | public | private |

---

## 3. 头文件与源文件

### 3.1 文件分工

| | 头文件 (.h) | 源文件 (.cpp) |
|---|-----------|-------------|
| **用途** | 定义接口 | 实现类函数 |
| **包含内容** | 函数原型、类声明、类型定义、宏、常量 | 函数实现、可执行代码 |
| **访问方式** | 在源文件间共享 | 编译为目标文件 |
| **示例** | `void someFunction();` | `void someFunction() {...};` |

---

## 4. 类的设计

一个设计良好的类通常包含：

1. **构造函数 (Constructor)** -- 初始化新创建对象的状态
2. **私有成员函数/变量** -- 实现细节，外部不可见
3. **公共成员函数** -- 对用户暴露的接口
4. **析构函数 (Destructor)** -- 清理资源

---

## 5. 构造函数

### 5.1 基本概念

构造函数初始化新创建对象的状态。以 `StanfordID` 类为例，我们需要初始化：
- `name`
- `sunet`
- `idNumber`

### 5.2 头文件声明

```cpp
// StanfordID.h
class StanfordID {
private:
    std::string name;
    std::string sunet;
    int idNumber;

public:
    // 构造函数 -- 语法就是类名本身
    StanfordID(std::string name, std::string sunet, int idNumber);

    // 获取器 (getter) 方法
    std::string getName();
    std::string getSunet();
    int getID();
};
```

### 5.3 带参数的构造函数实现

```cpp
// StanfordID.cpp
#include "StanfordID.h"
#include <string>

StanfordID::StanfordID(std::string name, std::string sunet, int idNumber) {
    name = name;
    sunet = sunet;
    idNumber = idNumber;
}
```

但这里有**问题**！注意到什么了吗？

### 5.4 使用 `this` 关键字消除歧义

```cpp
// 问题：name = name; -- 哪个是参数，哪个是成员变量？
// 编译器分不清楚！

// 解决方案：使用 this 指针
StanfordID::StanfordID(std::string name, std::string sunet, int idNumber) {
    this->name = name;      // this->name 是成员变量
    this->sunet = sunet;    // name 是参数
    this->idNumber = idNumber;
}
```

可以用构造函数添加**合法性检查**：

```cpp
StanfordID::StanfordID(std::string name, std::string sunet, int idNumber) {
    this->name = name;
    this->sunet = sunet;
    if (idNumber > 0)           // 只接受合法的ID号
        this->idNumber = idNumber;
}
```

### 5.5 列表初始化构造函数 (C++11)

```cpp
// 更优雅的写法 -- 成员初始化列表
StanfordID::StanfordID(std::string name, std::string sunet, int idNumber)
    : name{name}, sunet{sunet}, idNumber{idNumber} {}
```

这是**推荐**的初始化方式，对于某些类型（如const成员、引用成员）这是唯一的方式。

### 5.6 默认构造函数

```cpp
// 不带参数的构造函数 -- 使用默认值初始化
StanfordID::StanfordID() {
    name = "John Appleseed";
    sunet = "jappleseed";
    idNumber = 00000001;
}
```

### 5.7 构造函数重载

可以定义多个构造函数，编译器根据参数自动选择：

```cpp
// 默认构造函数
StanfordID::StanfordID() {
    name = "John Appleseed";
    sunet = "jappleseed";
    idNumber = 00000001;
}

// 带参数构造函数
StanfordID::StanfordID(std::string name, std::string sunet, int idNumber) {
    this->name = name;
    this->sunet = sunet;
    this->idNumber = idNumber;
}

// 使用：
StanfordID s1;                                     // 调用默认构造函数
StanfordID s2("Thomas", "tpoimen", 01243425);      // 调用带参数构造函数
```

---

## 6. 析构函数

### 6.1 声明与实现

```cpp
// StanfordID.cpp
StanfordID::~StanfordID() {
    // 释放/解除分配数据
    // 例如：delete [] my_array;
}
```

### 6.2 重要注意事项

- 析构函数**不是显式调用**的，当对象离开作用域时自动调用
- 在 `StanfordID` 示例中我们没有使用 `new` 动态分配数据，所以析构函数可以为空
- 但如果类管理了动态资源（如 `new` 分配的数组），必须在析构函数中释放

### 6.3 对象生命周期

```
创建对象 --> 构造函数被调用
   |
使用对象
   |
离开作用域 --> 析构函数自动被调用
```

---

## 7. 成员函数实现

### 7.1 Getter 方法

```cpp
// StanfordID.cpp
std::string StanfordID::getName() {
    return this->name;  // this 可以省略
}

std::string StanfordID::getSunet() {
    return this->sunet;
}

int StanfordID::getID() {
    return this->idNumber;
}
```

### 7.2 Setter 方法（带验证）

```cpp
void StanfordID::setName(std::string name) {
    this->name = name;
}

void StanfordID::setSunet(std::string sunet) {
    this->sunet = sunet;
}

void StanfordID::setID(int idNumber) {
    if (idNumber >= 0) {           // 验证：ID必须为非负数
        this->idNumber = idNumber;
    }
}
```

---

## 8. 类型别名

使用 `using` 可以创建类型的同义词：

```cpp
// StanfordID.h
class StanfordID {
private:
    // 类型别名示例
    using String = std::string;
    String name;
    String sunet;
    int idNumber;

public:
    StanfordID(String name, String sunet, int idNumber);
    String getName();
    String getSunet();
    int getID();
};
```

这种技术在以下场景特别有用：
- 缩短冗长的类型名
- 在模板类中定义迭代器类型
- 抽象实现细节，便于后续修改

---

## 9. 继承 (Inheritance)

### 9.1 为什么需要继承?

- **动态多态 (Dynamic Polymorphism)**：不同类型的对象可能需要相同的接口
- **可扩展性 (Extensibility)**：通过创建具有特定属性的子类来扩展现有类

### 9.2 形状类层级示例

```cpp
// Shape.h
class Shape {
public:
    virtual double area() const = 0;  // 纯虚函数
};
```

关键概念：
- `virtual` 关键字表示该函数可以在子类中被**覆写 (override)**
- `= 0` 表示**纯虚函数 (pure virtual function)** -- 它在基类中声明但不实现，必须在子类中实现

### 9.3 Circle 子类

```cpp
class Circle : public Shape {        // Circle 公开继承 Shape
public:
    // 构造函数 -- 使用列表初始化
    Circle(double radius) : _radius{radius} {}

    // 覆写基类的 area() 函数
    double area() const {
        return 3.14 * _radius * _radius;
    }

private:
    double _radius;                  // 封装：私有成员变量
};
```

### 9.4 Rectangle 子类

```cpp
class Rectangle : public Shape {
public:
    Rectangle(double height, double width)
        : _height{height}, _width{width} {}

    double area() const {
        return _width * _height;
    }

private:
    double _width, _height;
};
```

### 9.5 继承的好处

1. **代码复用**：公共逻辑放在基类，子类共享
2. **接口统一**：通过基类指针/引用操作不同子类
3. **封装**：子类成员变量受 `private` 保护
4. **多态**：运行时根据实际类型调用适当的方法

---

## 10. 继承类型

### 10.1 三种继承方式对比

| 继承类型 | public | protected | private |
|---------|--------|-----------|---------|
| **语法** | `class B : public A` | `class B : protected A` | `class B : private A` |
| **基类public成员在子类中** | public | protected | private |
| **基类protected成员在子类中** | protected | protected | private |
| **基类private成员在子类中** | 不可访问 | 不可访问 | 不可访问 |

### 10.2 使用建议

- **public 继承**最为常见，它正确表达了 "is-a" 关系
  - "一个 Circle 是一个 Shape" -- 直观且合理
- `protected` 和 `private` 继承较少使用，通常意味着 "is-implemented-in-terms-of" 关系

---

## 11. 菱形问题 (The Diamond Problem)

### 11.1 问题描述

```
           A
          / \
         /   \
        B     C
         \   /
          \ /
           D
```

由于 B 和 C 都继承自 A，它们各自调用 A 的构造函数。D 继承自 B 和 C，最终得到**两份 A 的副本**。

### 11.2 代码示例

```cpp
class A {
public:
    A();
    void hello() {
        // 打印 "hello from A"
    }
};

class B : public A {
public:
    B();
};

class C : public A {
public:
    C();
};

class D : public B, public C {
public:
    D();
};

// 问题：D 调用哪个 hello()？
D obj {};
obj.hello();         // 含糊不清！编译错误
obj.B::hello();      // 明确调用 B 的 hello
obj.C::hello();      // 明确调用 C 的 hello
```

### 11.3 解决方案：虚继承 (Virtual Inheritance)

```cpp
class B : virtual public A {  // 虚继承
public:
    B();
};

class C : virtual public A {  // 虚继承
public:
    C();
};

// 现在 B 和 C 共享同一个 A 的实例

class D : public B, public C {
public:
    D();
};

D obj {};
obj.hello();  // 不再含糊！B和C共享同一个A实例
```

虚继承意味着派生类（D）只应拥有基类（A）的单一实例。

---

## 12. 本章回顾

| 概念 | 要点 |
|------|------|
| **类** | 允许封装功能和数据，配合访问保护 |
| **继承** | 允许设计强大且灵活的抽象，帮助建模复杂的代码关系 |
| **多态** | 虚函数 + 继承 = 运行时根据实际类型调用正确方法 |

---

## 13. 补充知识点

### 13.1 成员初始化列表 vs 构造函数体

```cpp
// 方式一：构造函数体内赋值（效率较低）
Point::Point(int x, int y) {
    this->x = x;  // 先默认初始化，再赋值
    this->y = y;
}

// 方式二：成员初始化列表（推荐，效率更高）
Point::Point(int x, int y) : x{x}, y{y} {}
// 直接初始化，避免先默认构造再赋值
```

对于以下类型，**必须**使用成员初始化列表：
- `const` 成员变量
- 引用成员变量
- 没有默认构造函数的成员对象

### 13.2 编译器默认生成的特殊成员函数

C++编译器会自动生成以下函数（如果需要的话）：

| 函数 | 签名 | 何时自动生成 |
|------|------|------------|
| 默认构造函数 | `T()` | 没有用户定义构造函数时 |
| 析构函数 | `~T()` | 没有用户定义析构函数时 |
| 拷贝构造函数 | `T(const T&)` | 没有用户定义拷贝构造时 |
| 拷贝赋值运算符 | `T& operator=(const T&)` | 没有用户定义拷贝赋值时 |
| 移动构造函数 (C++11) | `T(T&&)` | 没有用户定义移动操作时 |
| 移动赋值运算符 (C++11) | `T& operator=(T&&)` | 没有用户定义移动操作时 |

### 13.3 三/五法则 (Rule of Three/Five)

- **三法则**：如果需要自定义析构函数、拷贝构造函数或拷贝赋值运算符中的任何一个，很可能需要自定义全部三个
- **五法则** (C++11)：加上移动构造函数和移动赋值运算符

```cpp
class Resource {
    int* data;
public:
    // 五法则全部实现
    Resource() : data{new int{0}} {}
    ~Resource() { delete data; }
    Resource(const Resource& other) : data{new int{*other.data}} {}
    Resource& operator=(const Resource& other) {
        if (this != &other) *data = *other.data;
        return *this;
    }
    Resource(Resource&& other) noexcept : data{other.data} {
        other.data = nullptr;
    }
    Resource& operator=(Resource&& other) noexcept {
        if (this != &other) { delete data; data = other.data; other.data = nullptr; }
        return *this;
    }
};
```

### 13.4 `= default` 和 `= delete` (C++11)

```cpp
class MyClass {
public:
    MyClass() = default;                  // 显式要求编译器生成默认构造函数
    MyClass(const MyClass&) = delete;     // 禁止拷贝构造
    MyClass& operator=(const MyClass&) = delete;  // 禁止拷贝赋值
};
```

### 13.5 友元 (friend) 声明

`friend` 允许外部函数或其他类访问私有成员：

```cpp
class MyClass {
private:
    int secret;

    friend void printSecret(const MyClass& obj);  // 友元函数
    friend class AnotherClass;                     // 友元类
};

void printSecret(const MyClass& obj) {
    std::cout << obj.secret;  // 可以访问私有成员
}
```

### 13.6 抽象类与接口

包含至少一个纯虚函数的类是**抽象类 (abstract class)**，不能实例化：

```cpp
class AbstractShape {
public:
    virtual double area() const = 0;     // 纯虚函数
    virtual double perimeter() const = 0;
    virtual ~AbstractShape() = default;  // 虚析构函数（重要！）
};

// AbstractShape s;  // ❌ 错误：抽象类不能实例化
```

### 13.7 虚析构函数的重要性

当通过基类指针删除派生类对象时，必须将基类析构函数声明为 `virtual`：

```cpp
class Base {
public:
    virtual ~Base() { /* ... */ }  // ✅ 虚析构函数
};

class Derived : public Base {
    int* data;
public:
    Derived() : data{new int[100]} {}
    ~Derived() { delete[] data; }
};

Base* ptr = new Derived();
delete ptr;  // 正确调用 Derived::~Derived() 然后 Base::~Base()
             // 如果 Base::~Base() 不是 virtual，只调用 Base::~Base() -- 内存泄漏！
```
