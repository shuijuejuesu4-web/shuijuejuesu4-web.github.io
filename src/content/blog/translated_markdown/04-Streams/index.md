---
title: "第4章：流 (Streams)"
description: "第4章：流 (Streams)"
publishDate: 2024-01-01
tags: [C++, CS106L]
category: "CS106L"
draft: false
comment: true
---
# 第4章：流 (Streams)

> **授课教师**：Rachel Fernandez, Thomas Poimenidis
> **学期**：Stanford CS106L, Fall 2025

---

## 4.1 上节课快速回顾

### 4.1.1 统一初始化

一种通用且安全的初始化方式，使用 `{}` 语法：

```cpp
int x{5};
std::vector<int> v{1, 2, 3};
std::map<std::string, int> m{{"a", 1}, {"b", 2}};
```

### 4.1.2 引用

一种给变量起别名的方式，允许多个变量引用同一块内存：

```cpp
int num = 5;
int& ref = num;  // ref 是 num 的别名
ref = 10;        // num 现在也是 10
```

---

## 4.2 为什么需要流 (Streams)？

> "为一门编程语言设计和实现一个通用的输入/输出设施是出了名的困难。" — Bjarne Stroustrup
>
> "所以我来做了。" — 一个流 (a stream)

流是 C++ 通用的输入/输出（IO）抽象设施。

---

## 4.3 抽象 (Abstraction)

### 4.3.1 什么是抽象？

> **抽象** = 隐藏不必要的细节，只暴露相关的内容。

就像开车时，你不必担心马达、线路和硬件 —— 你只需要用方向盘和踏板，相信汽车处理了剩下的事情。

### 4.3.2 抽象在流中的作用

抽象提供了一致的接口。对于流来说，这个接口用于**读取和写入数据**。

> **核心理念**：流帮助我们读取和写入数据。

---

## 4.4 你已经在使用流了！

### 4.4.1 熟悉的流示例

```cpp
std::cout << "Hello, World" << std::endl;
// ^^^ 这就是一个流！
```

### 4.4.2 控制台输入

```cpp
std::string student_input;
std::cin >> student_input;
// ^^^ 这也是一个流！
```

### 4.4.3 文件输出

```cpp
// 创建一个名为 "data.txt" 的文件
std::ofstream fout("data.txt");
fout << "I'm writing to this file";
```

### 4.4.4 文件输入

```cpp
std::ifstream fin("data.txt");
std::string first_word;
fin >> first_word;  // 存储第一个单词
```

> 注意到 `<<` 和 `>>` 了吗？这就是抽象在起作用！我们可以用一致的接口来处理输入和输出。

---

## 4.5 流的层次结构

### 4.5.1 `ios_base`

`ios_base` 是所有流相关功能的基础。

`ios_base` 维护的数据：
- **状态信息 (State Information)**：标志位，告诉你流的状态/健康状况
  - `failbit`：逻辑错误（如类型错误）
  - `eofbit`：到达字符串末尾
- **控制信息 (Control Information)**：流如何呈现数据
  - 例如：`255` 是显示为 `"255"`、`"FF"` 还是 `"377"`？

### 4.5.2 继承层次

```
ios_base
  └── basic_ios    (确保流正常工作 + 流的来源：控制台/键盘/文件)
        ├── ostream  (用于输出)
        │     ├── std::ofstream       (文件输出流)
        │     ├── std::ostringstream  (字符串输出流)
        │     └── std::cout           (控制台输出流)
        └── istream  (用于输入)
              ├── std::ifstream       (文件输入流)
              ├── std::istringstream  (字符串输入流)
              └── std::cin            (控制台输入流)
```

`ostream` 和 `istream` 的交集被称为 `iostream`，它具有两者的所有特性。

---

## 4.6 流的两大分类

### 4.6.1 输入流 (Input Streams)

- 一种从源读取数据的方式
- 继承自 `std::istream`
- 例如：从控制台读取（`std::cin`）
- 主要操作符：`>>`（称为**提取运算符** extraction operator）

### 4.6.2 输出流 (Output Streams)

- 一种向目标写入数据的方式
- 继承自 `std::ostream`
- 例如：向控制台写入（`std::cout`）
- 主要操作符：`<<`（称为**插入运算符** insertion operator）

---

## 4.7 流如何处理数据

### 4.7.1 数据流转过程

```
外部源 → 流 → 类型转换 → 程序中的值

"3.14" (字符串表示)  →  3.14 (double 在你的程序中)
```

### 4.7.2 为什么这有用？

流允许一种**通用的方式**来处理外部数据。无论数据来自键盘、文件还是网络，都可以用相同的接口。

---

## 4.8 `std::stringstream`

### 4.8.1 定义

**是什么**：一种将字符串当作流来处理的方式

**用途**：`stringstream` 在混合数据类型的场景中非常有用 —— 它既是 `ostream` 也是 `istream`。

### 4.8.2 基本示例

```cpp
void foo() {
    /// 部分 Bjarne 名言
    std::string initial_quote = "Bjarne Stroustrup C makes it easy to shoot yourself in the foot\n";

    /// 创建 stringstream
    std::stringstream ss(initial_quote);

    /// 数据目的地
    std::string first;
    std::string last;
    std::string language, extracted_quote;

    ss >> first >> last >> language >> extracted_quote;

    std::cout << first << " " << last << " said this: " << language << " " << extracted_quote << std::endl;
}
```

### 4.8.3 流的可视化

```
字符串: "Bjarne Stroustrup C makes it easy to shoot yourself in the foot\n"

提取过程:
  ss >> first >> last >> language >> extracted_quote;

  first  = "Bjarne"
  last   = "Stroustrup"
  language = "C"
  extracted_quote = ???
```

### 4.8.4 问题：`>>` 只读到下一个空白字符

`>>` 运算符**只读到下一个空白字符（whitespace）**为止。

所以 `extracted_quote` 只会得到 `"makes"`，而不是整个引文！

---

## 4.9 使用 `std::getline()`

### 4.9.1 函数签名

```cpp
istream& getline(istream& is, string& str, char delim);
```

### 4.9.2 关键特性

- `getline()` 从输入流 `is` 读取数据，直到遇到分隔符 `delim`，并存储到缓冲区 `str` 中
- 分隔符默认为 `'\n'`
- **注意**：`getline()` **会消费**分隔符！

### 4.9.3 修复 stringstream 问题

```cpp
void foo() {
    std::string initial_quote = "Bjarne Stroustrup C makes it easy to shoot yourself in the foot\n";

    std::stringstream ss(initial_quote);

    std::string first;
    std::string last;
    std::string language, extracted_quote;

    ss >> first >> last >> language;
    std::getline(ss, extracted_quote);  // 读取剩余的行

    std::cout << first << " " << last << " said this: '" << language << " " << extracted_quote + "'" << std::endl;
}
```

现在 `extracted_quote` 会包含 `" makes it easy to shoot yourself in the foot"`。

---

## 4.10 输出流深入

### 4.10.1 缓冲机制

输出流中的字符被存储在一个**中间缓冲区**中，在被**刷新 (flush)** 到目标之前不会显示。

```
double tao = 6.28;
std::cout << tao;
//   缓冲区中有 "6.28"，但控制台还没显示！
```

### 4.10.2 什么时候刷新？

- `std::cout << std::flush` — 手动刷新
- `std::cout << std::endl` — 换行并刷新
- 程序结束时 — 自动刷新
- 缓冲区满时 — 自动刷新
- 绑定流交互时 — 例如 `cout` 必须在 `cin` 获取输入前刷新

### 4.10.3 `std::endl` vs `'\n'`

```cpp
// 使用 std::endl
for (int i=1; i <= 5; ++i) {
    std::cout << i << std::endl;  // 每行输出并刷新
}
// 输出:
// 1
// 2
// 3
// 4
// 5

// 使用 '\n'
for (int i=1; i <= 5; ++i) {
    std::cout << i << '\n';  // 仅换行，不刷新
}
// 输出: 同样，但可能更高效
```

**建议**：在不需要立即刷新的场景下使用 `'\n'` 而不是 `std::endl`，可以获得更好的性能。

### 4.10.4 `cerr` 和 `clog`

| 流 | 用途 | 缓冲 |
|----|------|------|
| `std::cerr` | 输出错误信息 | **无缓冲**（立即发送） |
| `std::clog` | 非关键事件日志 | 有缓冲 |

### 4.10.5 性能提示：`std::ios::sync_with_stdio`

```cpp
int main()
{
    std::ios::sync_with_stdio(false);  // 可能获得巨大的性能提升
    for (int i=1; i <= 5; ++i) {
        std::cout << i << '\n';
    }
    return 0;
}
```

**注意**：这仅在输出流是非交互式的时候才有效（如文件、Unix 管道）。如果是终端交互式输出，`'\n'` 仍然可能触发刷新。

---

## 4.11 输出文件流 (`std::ofstream`)

### 4.11.1 基本操作

```cpp
int main() {
    /// 构造时关联文件
    std::ofstream ofs("hello.txt");

    if (ofs.is_open()) {                     // 检查文件是否打开
        ofs << "Hello CS106L!" << '\n';      // 写入文件
    }

    ofs.close();                             // 关闭文件流

    ofs << "this will not get written";      // 静默失败

    ofs.open("hello.txt");                   // 重新打开
    ofs << "this will though! It's open again";

    return 0;
}
```

### 4.11.2 关键方法

| 方法 | 说明 |
|------|------|
| `is_open()` | 检查文件是否打开 |
| `open(filename)` | 打开文件 |
| `close()` | 关闭文件 |
| `fail()` | 检查流是否失败 |

### 4.11.3 追加模式

```cpp
// 默认：truncate（覆盖）
std::ofstream ofs("hello.txt");

// 追加模式
std::ofstream ofs("hello.txt", std::ios::app);
ofs << "This will be appended to the file";
```

---

## 4.12 输入文件流 (`std::ifstream`)

```cpp
int inputFileStreamExample() {
    std::ifstream ifs("input.txt");
    if (ifs.is_open()) {
        std::string line;
        std::getline(ifs, line);
        std::cout << "Read from the file: " << line << '\n';
    }
    if (ifs.is_open()) {
        std::string lineTwo;
        std::getline(ifs, lineTwo);
        std::cout << "Read from the file: " << lineTwo << '\n';
    }
    return 0;
}
```

**注意**：输入和输出流在相同的源/目标类型上是互补的！

---

## 4.13 输入流深入 (`std::cin`)

### 4.13.1 `std::cin` 的特性

- `std::cin` 是**有缓冲的**
- 可以把它想象成一个地方，用户可以在那里存储一些数据，然后从中读取
- `std::cin` 缓冲在**空白字符**处停止

### 4.13.2 C++ 中的空白字符

- `" "` — 空格字符
- `'\n'` — 换行字符
- `'\t'` — 制表字符

### 4.13.3 `std::cin` 基本使用

```cpp
int main()
{
    double pi;
    std::cin >> pi;                    // 从控制台读取
    std::cout << "pi is: " << pi << '\n';
    return 0;
}
```

流程：
1. `cin` 缓冲区为空 → 提示用户输入
2. 用户输入 `3.14\n`
3. `cin >> pi` 从缓冲区读取直到遇空白字符
4. `pi` 被赋值为 `3.14`

---

## 4.14 当 `std::cin` 失败时

### 4.14.1 问题场景

```cpp
int main()
{
    double pi;
    double tao;
    std::string name;

    std::cin >> pi;       // 用户输入: 3.14\n
    std::cin >> name;     // 用户输入: Rachel Fernandez\n
    std::cin >> tao;      // 尝试从缓冲区读取...

    std::cout << "my name is: " << name << " tao is: " << tao << " pi is: " << pi << '\n';
    return 0;
}
```

**问题**：
1. `cin >> pi` 读取 `3.14`，缓冲区剩余 `\nRachel Fernandez\n`
2. `cin >> name` 读取 `Rachel`（到空白字符），缓冲区剩余 `Fernandez\n`
3. `cin >> tao` 尝试读取 `Fernandez`... 但 `tao` 是 `double`！读取失败，`tao` 变为 `0`！

### 4.14.2 另一个陷阱：`getline()` 与 `cin` 混用

```cpp
void cinGetlineBug() {
    double pi;
    double tao;
    std::string name;

    std::cin >> pi;                     // 用户输入: 3.14\n
    std::getline(std::cin, name);       // 消费了残留的 \n！
    std::cin >> tao;                    // 现在尝试读取 name 的下一个词...

    std::cout << "my name is : " << name << " tao is : " << tao << " pi is : " << pi << '\n';
}
```

**问题**：
1. `cin >> pi` 读取 `3.14`，缓冲区残留 `\n`
2. `getline()` **立即消费**了 `\n`，所以 `name` 变成空字符串！
3. `cin >> tao` 尝试读取下一个输入

### 4.14.3 修复方式

**方案 1：双 `getline` 调用**

```cpp
void cinGetline() {
    double pi;
    double tao;
    std::string name;

    std::cin >> pi;
    std::getline(std::cin, name);       // 消费残留的 \n
    std::getline(std::cin, name);       // 读取真正的名字
    std::cin >> tao;

    std::cout << "my name is : " << name << " tao is : " << tao << " pi is : " << pi << '\n';
}
```

---

## 4.15 重要警告

**你不应该混合使用 `getline()` 和 `std::cin`！**

原因在于它们处理数据的差异：
- `std::cin >>` — 将换行符**留在**缓冲区中
- `getline()` — **移除**换行符

---

## 4.16 本章回顾

1. **流**是一个通用的接口，用于在程序中读取和写入数据
2. **输入流和输出流**在相同的源/目标类型上是互补的
3. **不要混合使用 `getline()` 和 `std::cin`**，除非确有必要！

---

## 📚 补充知识点

### 流的完整分类

| 头文件 | 类 | 用途 |
|--------|-----|------|
| `<iostream>` | `std::istream` | 通用输入流基类 |
| `<iostream>` | `std::ostream` | 通用输出流基类 |
| `<iostream>` | `std::iostream` | 通用输入输出流 |
| `<iostream>` | `std::cin` | 标准输入流（控制台） |
| `<iostream>` | `std::cout` | 标准输出流（控制台） |
| `<iostream>` | `std::cerr` | 标准错误流（无缓冲） |
| `<iostream>` | `std::clog` | 标准日志流（有缓冲） |
| `<sstream>` | `std::stringstream` | 字符串流（输入+输出） |
| `<sstream>` | `std::istringstream` | 字符串输入流 |
| `<sstream>` | `std::ostringstream` | 字符串输出流 |
| `<fstream>` | `std::ifstream` | 文件输入流 |
| `<fstream>` | `std::ofstream` | 文件输出流 |
| `<fstream>` | `std::fstream` | 文件输入输出流 |

### 流状态标志详解

```cpp
std::ifstream file("data.txt");

if (file.good())  { /* 流状态正常 */ }
if (file.eof())   { /* 已到达文件末尾 */ }
if (file.fail())  { /* 逻辑错误（如类型不匹配） */ }
if (file.bad())   { /* 严重 I/O 错误 */ }

// 重置状态标志
file.clear();
```

### 输入验证常用模式

```cpp
int getIntFromUser() {
    int value;
    while (true) {
        std::cout << "Enter an integer: ";
        if (std::cin >> value) {
            // 输入成功
            return value;
        } else {
            // 输入失败
            std::cin.clear();  // 清除错误标志
            std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');  // 丢弃无效输入
            std::cout << "Invalid input, please try again.\n";
        }
    }
}
```

### `ostringstream` 用于格式化字符串

```cpp
#include <sstream>

std::string formatMessage(const std::string& name, int age) {
    std::ostringstream oss;
    oss << "Hello, " << name << "! You are " << age << " years old.";
    return oss.str();  // 返回构建的字符串
}
```

### 文件流模式标志

```cpp
// 常用打开模式
std::ios::in      // 输入（ifstream 默认）
std::ios::out     // 输出（ofstream 默认），会覆盖
std::ios::app     // 追加（在文件末尾写入）
std::ios::ate     // 打开时定位到文件末尾
std::ios::trunc   // 打开时清空文件
std::ios::binary  // 二进制模式

// 组合使用
std::ofstream file("data.txt", std::ios::out | std::ios::app);
```

### `'\n'` vs `std::endl` 性能对比

对于大量输出，使用 `'\n'` 替代 `std::endl` 可以显著提升性能。因为 `std::endl` 每次都会触发一次 flush，而缓冲区频繁刷新是昂贵的操作。仅在需要确保输出立即显示时使用 `std::endl`（例如在崩溃前打印调试信息）。

### 流派生与多态

由于 `ostream` 和 `istream` 是所有具体流类的基类，你可以编写接受基类引用/指针的函数，从而实现多态：

```cpp
void writeMessage(std::ostream& os, const std::string& msg) {
    os << "Message: " << msg << '\n';
}

// 可以传入任何 ostream 派生类
writeMessage(std::cout, "Hello");       // 输出到控制台

std::ofstream file("output.txt");
writeMessage(file, "Hello");            // 输出到文件

std::ostringstream oss;
writeMessage(oss, "Hello");             // 输出到字符串
```
