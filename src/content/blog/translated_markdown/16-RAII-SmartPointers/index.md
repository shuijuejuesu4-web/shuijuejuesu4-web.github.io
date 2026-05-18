---
title: "第16章：RAII、智能指针与 C++ 项目构建"
description: "第16章：RAII、智能指针与 C++ 项目构建"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第16章：RAII、智能指针与 C++ 项目构建

> **授课教师**：Thomas Poimenidis, Rachel Fernandez
> **学期**：Stanford CS106L, Fall 2025

---

## 目录

- [16.1 异常与资源泄漏问题](#161-异常与资源泄漏问题)
- [16.2 RAII（资源获取即初始化）](#162-raii资源获取即初始化)
- [16.3 智能指针](#163-智能指针)
  - [16.3.1 std::unique_ptr](#1631-stdunique_ptr)
  - [16.3.2 std::shared_ptr](#1632-stdshared_ptr)
  - [16.3.3 std::weak_ptr](#1633-stdweak_ptr)
- [16.4 构建 C++ 项目](#164-构建-c-项目)
  - [16.4.1 Makefile 与 make](#1641-makefile-与-make)
  - [16.4.2 CMake](#1642-cmake)
- [📚 补充知识点](#-补充知识点)
- [🔧 常用API参考](#-常用api参考)

---

## 16.1 异常与资源泄漏问题

### 16.1.1 C++ 的异常机制

异常是一种在代码中处理错误的方式：

```cpp
try {
    // 需要检查异常的代码
} catch (const std::exception& e) {
    // 处理异常的代码（"if"）
} catch (const OtherException& e) {
    // 处理其他类型的异常（"else if"）
} catch (...) {
    // 捕获所有异常（"else"）——兜底
}
```

- 异常被**抛出（throw）**
- 我们可以**捕获（catch）** 异常，从而在不崩溃的情况下继续执行

### 16.1.2 资源泄漏的隐患

在一个函数中，可能存在许多代码路径（例如 23 个以上的代码路径！），其中任何一个都可能抛出异常：

```cpp
void riskyFunction() {
    Pet* pet = new Pet();         // 动态分配
    // ... 很多可能抛出异常的代码 ...
    // 如果中途抛出异常，delete pet 永远不会执行！
    delete pet;
}
```

**如果异常发生在 `new` 和 `delete` 之间，就会导致内存泄漏！**

这不仅限于指针——任何需要释放的资源（文件、锁、网络连接等）都有同样的问题。

**关键问题**：如何在异常发生时确保资源被正确释放？

---

## 16.2 RAII（资源获取即初始化）

### 16.2.1 什么是 RAII？

> **RAII** = Resource Acquisition Is Initialization（资源获取即初始化）

由 Bjarne Stroustrup 提出，是 C++ 中极具代表性的设计理念。

**RAII 原则**：
1. 类所使用的所有资源都应该在**构造函数**中获取
2. 类所使用的所有资源都应该在**析构函数**中释放

### 16.2.2 为什么需要 RAII？

1. **避免"半有效"状态**：对象要么完全可用，要么不存在
2. **析构函数保证被调用**：无论以何种方式离开作用域（正常返回、异常），析构函数**总是**会被调用
3. **对象创建后立即可用**

### 16.2.3 非 RAII 示例

```cpp
// 不是 RAII：ifstream 的打开和关闭在代码中，不是在构造/析构中
void badExample() {
    std::ifstream file;
    file.open("data.txt");
    // ... 中间的代码可能抛出异常 ...
    file.close();  // 如果发生异常，这行不会执行！
}
```

```cpp
// 也不是 RAII：锁的获取和释放不在构造/析构中
std::mutex mtx;
void badLock() {
    mtx.lock();
    // ... 关键区域 ...（如果这里抛异常，锁永远不会释放！）
    mtx.unlock();
}
```

### 16.2.4 RAII 解决方案

```cpp
// RAII 版本：ifstream 在构造时打开，析构时自动关闭
void goodExample() {
    std::ifstream file("data.txt");  // 构造时打开
    // ... 使用文件 ...
}  // file 离开作用域，析构函数自动关闭文件

// RAII 版本：lock_guard 在构造时获取锁，析构时释放
void goodLock() {
    std::lock_guard<std::mutex> guard(mtx);  // 构造时获取锁
    // ... 关键区域 ...
}  // guard 离开作用域，析构函数自动释放锁
```

**RAII 对应表**：

| 资源类型 | 非 RAII | RAII 封装 |
|---------|---------|-----------|
| 内存 | `new` / `delete` | 智能指针 |
| 互斥锁 | `lock()` / `unlock()` | `std::lock_guard`, `std::unique_lock` |
| 文件 | `open()` / `close()` | `std::ifstream`, `std::ofstream` |
| 动态数组 | `new[]` / `delete[]` | `std::vector`, `std::array` |

---

## 16.3 智能指针

RAII for memory → **智能指针**！

智能指针是一种 RAII 兼容的"包装指针"，在构造函数中获取内存，在析构函数中释放。

### 16.3.1 std::unique_ptr

`#include <memory>`

**独占所有权** —— 不能被拷贝。

```cpp
std::unique_ptr<Pet> pet = std::make_unique<Pet>("Fido");
// pet 独占 Pet 对象的所有权
// 当 pet 离开作用域时，Pet 自动被 delete
```

**为什么 unique_ptr 不能被拷贝？**

如果允许拷贝，原始 unique_ptr 的析构函数会释放内存，而拷贝的指针将指向已释放的内存（dangling pointer）。

```cpp
std::unique_ptr<Pet> p1 = std::make_unique<Pet>("Fido");
// std::unique_ptr<Pet> p2 = p1;  // 编译错误！不能拷贝

// 但可以移动（转移所有权）
std::unique_ptr<Pet> p2 = std::move(p1);  // p1 变为 nullptr，p2 拥有对象
```

### 16.3.2 std::shared_ptr

**共享所有权** —— 可以被拷贝。底层内存直到**所有** shared_ptr 都离开作用域后才被释放。

```cpp
std::shared_ptr<Pet> pet1 = std::make_shared<Pet>("Fido");
std::shared_ptr<Pet> pet2 = pet1;   // 可以拷贝！
std::shared_ptr<Pet> pet3 = pet1;   // 又一个拷贝

// 只有当 pet1, pet2, pet3 都离开作用域，Pet 才被释放
```

原理：使用**引用计数（reference counting）** —— 每多一个 shared_ptr，计数器 +1；每释放一个，计数器 -1。计数器归零时，释放资源。

### 16.3.3 std::weak_ptr

**弱指针**：一种设计用来**打破循环依赖**的指针。

#### 循环依赖问题

```
class A {                          class B {
    shared_ptr<B> b;                   shared_ptr<A> a;
};                                };
```

两个对象互相持有 shared_ptr → 引用计数永远不会归零 → **内存泄漏**！

#### 使用 weak_ptr 解决

```cpp
class B;  // 前向声明

class A {
    std::shared_ptr<B> b;  // 拥有
};

class B {
    std::weak_ptr<A> a;    // 仅观察，不增加引用计数
};
```

`weak_ptr` 不增加引用计数，因此不会阻止对象被释放。使用时需要通过 `.lock()` 获取临时的 `shared_ptr`：

```cpp
if (auto sp = weakPtr.lock()) {  // 如果原对象还存在
    sp->doSomething();
} else {
    // 原对象已被释放
}
```

### 16.3.4 始终使用 make_unique / make_shared

```cpp
// 不好的做法：显式使用 new
std::unique_ptr<Pet> pet(new Pet("Fido"));           // 直接使用 new（不推荐，异常安全性差）
std::shared_ptr<Pet> pet2(new Pet("Fido"));          // 两次分配！

// 好的做法：使用 make_ 函数
auto pet = std::make_unique<Pet>("Fido");             // 一次分配
auto pet2 = std::make_shared<Pet>("Fido");            // 一次分配
```

**原因**：
1. **性能**：直接使用 `new` 会分配两次（一次为指针控制块，一次为 T），`make_` 系列只分配一次
2. **一致性**：始终使用 `make_unique`/`make_shared` 保持代码风格一致
3. **异常安全**：函数调用中不会因为求值顺序问题导致内存泄漏

---

## 16.4 构建 C++ 项目

### 16.4.1 Makefile 与 make

`make` 是一个**构建系统**程序，帮助你编译项目。通过 `Makefile` 来配置：

```makefile
# 编译器
CXX = g++

# 编译标志
CXXFLAGS = -std=c++20

# 源文件和目标
SRCS = $(wildcard *.cpp)
TARGET = main

# 默认目标
all:
    $(CXX) $(CXXFLAGS) $(SRCS) -o $(TARGET)

# 清理
clean:
    rm -f $(TARGET)
```

### 16.4.2 CMake

CMake 是一个**构建系统生成器**——用 CMake 生成 Makefile。它是比 Makefile 更高层的抽象。

#### CMakeLists.txt 示例

```cmake
cmake_minimum_required(VERSION 3.10)
project(cs106l_classes)
set(CMAKE_CXX_STANDARD 20)
file(GLOB SRC_FILES "*.cpp")
add_executable(main ${SRC_FILES})
```

指令解释：

| 指令 | 含义 |
|------|------|
| `cmake_minimum_required(VERSION 3.10)` | 指定 CMake 最低版本 |
| `project(cs106l_classes)` | 定义项目名称 |
| `set(CMAKE_CXX_STANDARD 20)` | 设置 C++ 标准为 C++20 |
| `file(GLOB SRC_FILES "*.cpp")` | 通配符搜索所有 .cpp 文件 |
| `add_executable(main ${SRC_FILES})` | 将所有源文件编译为可执行文件 main |

#### 使用 CMake 的步骤

```
1. 项目根目录下创建 CMakeLists.txt
2. 创建 build 文件夹：mkdir build
3. 进入 build 文件夹：cd build
4. 运行 cmake ..（使用父目录的 CMakeLists.txt 生成 Makefile）
5. 运行 make（编译）
6. 运行程序：./main
```

---

## 📚 补充知识点

### RAII 与 C 语言对比

在 C 语言中管理资源：

```c
// C 语言：手动管理，容易出错
FILE* f = fopen("data.txt", "r");
if (!f) return;
// ... 使用文件 ...
// 如果这中间有多个 return 或 goto，很容易忘记 fclose
fclose(f);
```

在 C++ 中，RAII 自动处理：

```cpp
{
    std::ifstream f("data.txt");
    // ... 使用文件 ...
    // 无论怎么离开作用域（return、异常、goto），f 都会自动关闭
}
```

### unique_ptr 与所有权转移

```cpp
// 工厂函数返回 unique_ptr —— 转移所有权给调用方
std::unique_ptr<Widget> createWidget() {
    auto w = std::make_unique<Widget>();
    w->initialize();
    return w;  // 自动移动（RVO / NRVO）
}

// 函数接受 unique_ptr —— 接管所有权
void consumeWidget(std::unique_ptr<Widget> w) {
    w->process();
}  // w 在此销毁
```

### shared_ptr 的引用计数原理

```
shared_ptr<Foo> sp1 = make_shared<Foo>();  // refcount = 1
shared_ptr<Foo> sp2 = sp1;                 // refcount = 2
shared_ptr<Foo> sp3 = sp1;                 // refcount = 3
sp1.reset();                               // refcount = 2
sp2.reset();                               // refcount = 1
sp3.reset();                               // refcount = 0 → Foo 被释放！
```

### 智能指针使用建议

| 场景 | 推荐 |
|------|------|
| 独占所有权 | `std::unique_ptr` |
| 共享所有权 | `std::shared_ptr` |
| 打破循环引用 | `std::weak_ptr` |
| 观察但不拥有 | 裸指针 `T*`（非 owning） |
| 可选语义 | `std::optional<T>`（值语义）优于指针 |

### 现代 CMake 最佳实践

```cmake
cmake_minimum_required(VERSION 3.16)
project(MyProject VERSION 1.0.0 LANGUAGES CXX)

# 使用 target-based 方式，而不是全局设置
add_library(my_lib STATIC
    src/lib.cpp
)
target_include_directories(my_lib PUBLIC include)
target_compile_features(my_lib PUBLIC cxx_std_20)

add_executable(main src/main.cpp)
target_link_libraries(main PRIVATE my_lib)
```

---

## 🔧 常用API参考

### std::unique_ptr\<T\>

`#include <memory>`

| 操作 | 描述 | 示例 |
|------|------|------|
| `std::make_unique<T>(args...)` | 创建 unique_ptr（推荐） | `auto p = std::make_unique<Foo>(a, b);` |
| `get()` | 返回裸指针（不释放所有权） | `T* raw = p.get();` |
| `release()` | 释放所有权，返回裸指针 | `T* raw = p.release();`（需手动 delete） |
| `reset()` | 释放资源，置为 nullptr | `p.reset();` |
| `reset(T*)` | 释放当前资源，接管新指针 | `p.reset(new Foo());` |
| `operator*()` | 解引用 | `(*p).method();` |
| `operator->()` | 访问成员 | `p->method();` |
| `operator bool()` | 检查是否非空 | `if (p)` |
| `swap(unique_ptr&)` | 交换两个 unique_ptr | `p1.swap(p2);` |
| `std::move(p)` | 转移所有权 | `auto p2 = std::move(p1);` |

### std::shared_ptr\<T\>

| 操作 | 描述 | 示例 |
|------|------|------|
| `std::make_shared<T>(args...)` | 创建 shared_ptr（推荐） | `auto p = std::make_shared<Foo>(a, b);` |
| `get()` | 返回裸指针 | `T* raw = p.get();` |
| `reset()` | 释放当前引用 | `p.reset();` |
| `reset(T*)` | 释放当前引用，接管新指针 | `p.reset(new Foo());` |
| `use_count()` | 返回引用计数 | `long n = p.use_count();` |
| `unique()` | 是否唯一拥有（C++17 弃用，C++20 移除） | `if (p.unique())` — 请用 `use_count() == 1` |
| `operator*()`, `operator->()` | 解引用，访问成员 | `*p`, `p->method()` |
| `operator bool()` | 检查是否非空 | `if (p)` |

### std::weak_ptr\<T\>

| 操作 | 描述 | 示例 |
|------|------|------|
| `lock()` | 获取临时的 shared_ptr | `if (auto sp = wp.lock())` |
| `expired()` | 检查原对象是否已被释放 | `if (wp.expired())` |
| `use_count()` | 返回所观察对象的引用计数 | `long n = wp.use_count();` |
| `reset()` | 清空 weak_ptr | `wp.reset();` |
| `owner_before(weak_ptr)` | 所有者排序比较 | 用于容器排序 |

### 其他智能指针相关

| 功能 | 描述 | 示例 |
|------|------|------|
| `std::make_unique<T>` (C++14) | 创建 unique_ptr | `auto p = std::make_unique<int>(42);` |
| `std::make_shared<T>` (C++11) | 创建 shared_ptr | `auto p = std::make_shared<int>(42);` |
| `std::enable_shared_from_this<T>` | 从 this 安全获取 shared_ptr | `class Foo : public std::enable_shared_from_this<Foo>` |
| `shared_from_this()` | 获取指向 this 的 shared_ptr | `shared_ptr<Foo> sp = shared_from_this();` |

### RAII 标准库组件

| 组件 | 头文件 | 管理的资源 |
|------|--------|-----------|
| `std::unique_ptr<T>` | `<memory>` | 独占所有权的动态内存 |
| `std::shared_ptr<T>` | `<memory>` | 共享所有权的动态内存 |
| `std::weak_ptr<T>` | `<memory>` | 弱引用（打破循环） |
| `std::lock_guard<Mutex>` | `<mutex>` | 互斥锁（简单） |
| `std::unique_lock<Mutex>` | `<mutex>` | 互斥锁（可延迟锁定、手动解锁） |
| `std::scoped_lock<Mutexes...>` | `<mutex>` | 多互斥锁（C++17，死锁避免） |
| `std::ifstream` / `std::ofstream` | `<fstream>` | 文件 |
| `std::vector<T>` | `<vector>` | 动态数组 |
| `std::string` | `<string>` | 字符串内存 |
