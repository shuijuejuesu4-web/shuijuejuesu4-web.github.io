---
title: "第14章: 移动语义"
description: "CS106L Fall 2025 - Lecture 14: Move Semantics"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第14章: 移动语义

> **CS106L Fall 2025 - Lecture 14: Move Semantics**

## 目录

- [14.1 为什么需要移动语义](#141-为什么需要移动语义)
- [14.2 拷贝的代价](#142-拷贝的代价)
- [14.3 从拷贝到移动：核心思想](#143-从拷贝到移动核心思想)
- [14.4 lvalue 与 rvalue](#144-lvalue-与-rvalue)
- [14.5 左值引用与右值引用](#145-左值引用与右值引用)
- [14.6 移动构造函数与移动赋值运算符](#146-移动构造函数与移动赋值运算符)
- [14.7 `std::move` 详解](#147-stdmove-详解)
- [14.8 Rule of Five 完整版](#148-rule-of-five-完整版)
- [14.9 补充知识点](#149-补充知识点)

---

## 14.1 为什么需要移动语义

在深入移动语义之前，先回顾第13章学到的特殊成员函数：

```cpp
class Photo {
public:
    Photo(int width, int height);                    // 构造函数
    Photo(const Photo& other);                       // 拷贝构造函数
    Photo& operator=(const Photo& other);             // 拷贝赋值运算符
    ~Photo();                                         // 析构函数
private:
    int width;
    int height;
    int* data;   // 指向像素数据的指针（堆内存）
};
```

### 拷贝构造函数

```cpp
Photo::Photo(const Photo& other)
   : width(other.width)
   , height(other.height)
   , data(new int[width * height])    // 分配新内存
{
    std::copy(other.data, other.data + width * height, data);  // 逐像素复制
}
```

### 拷贝赋值运算符

```cpp
Photo& Photo::operator=(const Photo& other) {
    if (this == &other) return *this;  // 自赋值检查

    delete[] data;                     // 清理旧像素数据

    width = other.width;
    height = other.height;
    data = new int[width * height];
    std::copy(other.data, other.data + width * height, data);
    return *this;
}
```

### 析构函数

```cpp
Photo::~Photo() {
   delete[] data;   // 释放内存，防止内存泄漏
}
```

### 问题场景

```cpp
Photo takePhoto();  // 返回一个临时 Photo 对象

int main() {
    Photo selfie = takePhoto();   // (A) 拷贝构造 + 临时对象析构
    Photo retake(0, 0);
    retake = takePhoto();         // (B) 拷贝赋值 + 临时对象析构
}
```

**核心观察**：`takePhoto()` 的返回值是临时的——它在下一行执行前就会被销毁。但是，拷贝构造函数仍然会完整地复制所有像素数据，然后临时对象的析构函数又会立即释放原始数据。**这是巨大的浪费！**

---

## 14.2 拷贝的代价

```
拷贝过程（浪费）：

  takePhoto() 返回值:
  • width = 3840
  • height = 2160
  • data = 0x1024c3bd     (8.3M 像素的数据！)

  拷贝构造:
  • width = 3840
  • height = 2160
  • data = 0x133210f1     (全新的内存副本)

  然后是析构:
  • 释放 0x1024c3bd       (临时对象的数据被销毁)
```

对于一个 3840x2160 的图片（约8.3百万像素，每个像素4字节int），这意味着拷贝了约33MB的数据，而这份数据在下一行就立即被丢弃了。

---

## 14.3 从拷贝到移动：核心思想

### 移动语义的直觉

**移动 = 窃取资源，而非复制资源。**

```
移动过程（高效）：

  takePhoto() 返回值:
  • width = 3840
  • height = 2160
  • data = 0x1024c3bd

  移动构造:
  • width = 3840
  • height = 2160
  • data = 0x1024c3bd     (直接窃取指针！)

  然后：
  • 将源对象的 data 设为 nullptr

  最后是析构:
  • delete[] nullptr;     (释放空指针，什么都不做！)
```

**移动构造的关键步骤**：
1. 将目标对象的 `data` 指针直接指向源对象的像素数据
2. 将源对象的 `data` 指针设为 `nullptr`
3. 源对象的析构函数调用 `delete[]` 时，因为指针是 `nullptr`，不会产生任何效果

整个过程没有新建任何像素数据。**我们创建了一个全新的 `Photo` 对象，但没有进行任何拷贝！**

---

## 14.4 lvalue 与 rvalue

### 基本概念

**lvalue** 和 **rvalue** 是C++中泛化"临时性"概念的方式：

```cpp
void foo(Photo pic) {
   Photo beReal = pic;           // pic 是 lvalue（有确定地址的持久对象）
   Photo insta = takePhoto();   // takePhoto() 是 rvalue（临时对象）
}
```

### 直观理解

| 属性 | lvalue | rvalue |
|------|--------|--------|
| 地址 | 有确定的地址 | 没有确定的地址 |
| 生命周期 | 持续到作用域结束 | 只持续到当前行结束 |
| 本质 | 持久对象 | 临时对象 |
| 可以取地址？ | `&lvalue` 有效 | `&rvalue` 报错 |
| 出现在等号哪边？ | 等号两边都可以 | 只能出现在等号右边 |

```cpp
void foo(Photo pic) {
   Photo* p1 = &pic;              // 可以取 lvalue 的地址
   Photo* p2 = &takePhoto();     // 编译错误！不能取 rvalue 的地址
}

x = y;    // lvalue = lvalue
y = 5;    // lvalue = rvalue
x = 5;    // OK
5 = y;    // 编译错误！rvalue 不能出现在 = 左边
```

### 练习题

判断以下右侧表达式的值类别：

```cpp
int             a = 4;            // 4 是 rvalue
int&            b = a;            // a 是 lvalue
vector<int>     c = {1, 2, 3};    // {1, 2, 3} 是 rvalue
int             d = c[1];         // c[1] 是 lvalue（返回引用）
int*            e = &c[2];        // &c[2] 是 rvalue（临时地址）
size_t          f = c.size();     // c.size() 是 rvalue（临时值）
```

---

## 14.5 左值引用与右值引用

### 左值引用 (`Type&`)

```cpp
void upload(Photo& pic);        // 接受左值引用

int main() {
    Photo selfie = takePhoto(); // selfie 是 lvalue
    upload(selfie);              // OK：绑定 lvalue 到 lvalue reference
    upload(takePhoto());         // 编译错误！不能将 rvalue 绑定到 lvalue reference
}
```

### 右值引用 (`Type&&`)

```cpp
void upload(Photo&& pic);       // 接受右值引用

int main() {
    upload(takePhoto());        // OK：绑定 rvalue 到 rvalue reference
    // upload(selfie);          // 编译错误！不能将 lvalue 绑定到 rvalue reference
}
```

### 关键区别

| 参数类型 | 语法 | 含义 | 典型用途 |
|----------|------|------|----------|
| 左值引用 | `Type&` | 持久对象，函数结束后必须保持有效状态 | 修改传入的对象 |
| 常量左值引用 | `const Type&` | 持久或临时对象，不能修改 | 只读访问，避免拷贝 |
| 右值引用 | `Type&&` | 临时对象，可以窃取其资源 | 移动语义 |

> "我们可以对 `Photo&& pic` 做任何想做的事，因为它是临时的——反正很快就会被销毁！"

### 函数重载：区分 lvalue 和 rvalue

```cpp
void upload(Photo& pic);        // lvalue 版本
void upload(Photo&& pic);       // rvalue 版本

int main() {
    Photo selfie = takePhoto();
    upload(selfie);              // 调用 upload(Photo&)
    upload(takePhoto());        // 调用 upload(Photo&&)
}
// 编译器根据参数是 lvalue 还是 rvalue 来选择调用哪个版本！
```

---

## 14.6 移动构造函数与移动赋值运算符

### 两个新的特殊成员函数

| SMF | 签名 | 用途 |
|-----|------|------|
| 移动构造函数 | `Type::Type(Type&& other)` | 从临时对象"窃取"资源创建新对象 |
| 移动赋值运算符 | `Type& Type::operator=(Type&& other)` | 从临时对象"窃取"资源赋值 |

### 拷贝 vs 移动对比

**拷贝构造函数**：
```cpp
Photo::Photo(const Photo& other)
    : width(other.width)
    , height(other.height)
    , data(new int[width * height])   // 分配新内存
{
    std::copy(other.data, other.data + width * height, data);  // 逐元素复制
}
```

**移动构造函数**：
```cpp
Photo::Photo(Photo&& other)
    : width(other.width)
    , height(other.height)
    , data(other.data)                // 直接窃取指针！
{
    other.data = nullptr;             // 将源对象指针置空
}
```

**拷贝赋值运算符**：
```cpp
Photo& Photo::operator=(const Photo& other) {
    if (this == &other) return *this;
    delete[] data;
    width = other.width;
    height = other.height;
    data = new int[width * height];
    std::copy(other.data, other.data + width * height, data);
    return *this;
}
```

**移动赋值运算符**：
```cpp
Photo& Photo::operator=(Photo&& other) {
    if (this == &other) return *this;
    delete[] data;                     // 清理自己的旧数据
    width = other.width;
    height = other.height;
    data = other.data;                 // 窃取资源
    other.data = nullptr;              // 置空源指针
    return *this;
}
```

### 移动 vs 拷贝的场景选择

```
Photo selfie = pic;
// pic 是 lvalue（持久对象），之后可能还会被使用
// => 使用拷贝语义

Photo selfie = takePhoto();
// takePhoto() 是 rvalue（临时对象），即将被销毁
// => 使用移动语义（窃取资源）
```

---

## 14.7 `std::move` 详解

### 为什么需要 `std::move`

有时，我们明确知道一个 lvalue 之后不会再被使用，但编译器不会自动帮我们移动它（因为它是 lvalue）。这时需要 `std::move` 来"告诉"编译器可以移动。

### 示例：数组元素移动

```cpp
// 低效版本：每个元素都做拷贝
void PhotoCollection::insert(const Photo& pic, int pos) {
    for (int i = size(); i > pos; i--)
        myPhotos[i] = myPhotos[i - 1];  // 拷贝赋值
    myPhotos[pos] = pic;
}
```

```cpp
// 高效版本：使用 std::move 进行移位
void PhotoCollection::insert(const Photo& pic, int pos) {
    for (int i = size(); i > pos; i--)
        myPhotos[i] = std::move(myPhotos[i - 1]);  // 移动赋值！
    myPhotos[pos] = pic;
}
```

### `std::move` 的真相

**`std::move` 本质上只是一个类型转换**——它将 lvalue 强制转换为 rvalue 引用。

它不做任何实际"移动"操作。真正的移动由移动构造函数或移动赋值运算符完成。

```cpp
// std::move 等价于：
template <typename T>
T&& move(T& x) {
    return static_cast<T&&>(x);  // 只是类型转换！
}
```

### 使用 `std::move` 的注意事项

1. **被移动后的对象处于有效但未指定的状态（valid but unspecified state）**：被移动的对象仍可以安全析构或重新赋值，但不应该再被使用

```cpp
Photo takePhoto();

void foo(Photo whoAmI) {
    Photo selfie = std::move(whoAmI);
    // whoAmI 现在处于未知状态！
    whoAmI.get_pixel(21, 24);  // 未定义行为！
}
```

2. **避免在返回值上使用 `std::move`**：这会阻碍 NRVO（命名返回值优化）

```cpp
// 错误：阻碍编译器优化
Photo bad() {
    Photo p(100, 100);
    return std::move(p);  // 不要这样做！
}

// 正确：编译器会自动优化（NRVO）
Photo good() {
    Photo p(100, 100);
    return p;  // 编译器会自动使用移动或 NRVO
}
```

3. **除非性能真的至关重要，或者你确信对象之后不会被使用，否则不要显式使用 `std::move`**

---

## 14.8 Rule of Five 完整版

### Rule of Zero

如果类不管理内存（或其他外部资源），编译器生成的默认SMF就足够了。

```cpp
struct Post {
    Photo photo;          // Photo 管理自己的内存
    std::string caption;  // std::string 管理自己的内存
};
// 编译器生成的 SMF 会自动调用 Photo 和 string 的对应 SMF
```

### Rule of Three

如果类管理外部资源，必须定义：
- 析构函数
- 拷贝构造函数
- 拷贝赋值运算符

如果不这样做，编译器生成的版本只会做浅拷贝，导致多个对象指向同一块资源。

### Rule of Five

如果定义了拷贝构造/赋值和析构函数，那么也应该定义：
- 移动构造函数（可选但推荐）
- 移动赋值运算符（可选但推荐）

虽然不强制，但没有移动操作意味着容器和算法中的对象将退化为拷贝，导致不必要的性能损失。

---

## 14.9 补充知识点

### 1. 返回值优化 (RVO/NRVO)

编译器可能会优化掉拷贝/移动：

```cpp
Photo takePhoto() {
    return Photo(500, 500);  // RVO: 直接在调用者的位置构造对象
}

Photo takePhoto2() {
    Photo p(500, 500);
    return p;                // NRVO: 同样优化掉拷贝
}
```

因此，下面这行代码可能根本不调用拷贝或移动构造函数：

```cpp
Photo selfie = takePhoto();  // 可能直接原地构造，零开销！
```

### 2. 移动后对象的安全操作

C++标准规定，被移动后的对象处于"有效但未指定"（valid but unspecified）状态。安全操作包括：
- 析构
- 赋值（给一个新值）
- 不带前置条件的成员函数调用

不安全操作包括：
- 解引用被移动的 `unique_ptr`
- 调用依赖于特定状态的成员函数

### 3. 移动操作应为 `noexcept`

移动构造函数和移动赋值运算符应标记为 `noexcept`：

```cpp
Photo(Photo&& other) noexcept : /* ... */ { /* ... */ }
Photo& operator=(Photo&& other) noexcept { /* ... */ }
```

原因：STL容器（如 `std::vector`）在重新分配时，只有当移动操作是 `noexcept` 时才会使用移动而非拷贝。否则，为了强异常安全保证，它们会退化为拷贝。

### 4. 实际性能对比

对于一个包含 `N` 个 `Photo` 的 `std::vector` 的重新分配：

| 操作 | 无移动语义 | 有移动语义 |
|------|-----------|-----------|
| 每个元素 | 分配新内存 + 拷贝所有像素 | 窃取指针，零内存分配 |
| 总耗时 | O(N * pixels) | O(N) |
| 内存使用 | 峰值 2x | 峰值约 1x |

### 5. `std::move` 与 `const` 的交互

对 `const` 对象调用 `std::move` 不会做移动——它会被降级为拷贝：

```cpp
const Photo p(100, 100);
Photo p2 = std::move(p);  // 实际上调用的是拷贝构造函数！
// 因为 const T&& 不能绑定到 T&&，只能绑定到 const T&
```

###  6. 移动后应置空

移动构造函数中关键的一步是 `other.data = nullptr;`。如果不这样做，源对象的析构函数会释放被移走的内存，导致双重释放。

---

> **本章总结**：移动语义是C++11引入的最重要的特性之一。它允许我们避免不必要的深拷贝，通过在临时对象之间"窃取"资源来大幅提升性能。理解 lvalue/rvalue 的区别，以及编译器如何自动在适当的时候选择移动操作，是编写高效现代C++代码的关键。记住：编译器自动处理临时对象的移动，而 `std::move` 是给开发者用来显式说明"这个 lvalue 之后不会再被使用"的工具。
