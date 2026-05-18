# 第3章：初始化与引用

> **授课教师**：Thomas Poimenidis, Rachel Fernandez
> **学期**：Stanford CS106L, Fall 2025

---

## 3.1 上节课快速回顾

### 3.1.1 `auto`

`auto` 是一个关键字，告诉编译器去推断对象或变量的类型：

```cpp
#include <iostream>
#include <string>
#include <map>
#include <unordered_map>
#include <vector>

int main()
{
    std::map<std::string, std::vector<std::pair<int, std::unordered_map<char, double>>>>
    complexType;

    // 令人困惑的迭代器类型
    std::map<std::string, std::vector<std::pair<int, std::unordered_map<char, double>>>>::iterator
    it = complexType.begin();

    // 清晰（多了）的迭代器类型！
    auto it = complexType.begin();

    return 0;
}
```

**使用建议**：
- 酌情使用
- 通常当类型冗长到令人厌烦时才使用 `auto`

### 3.1.2 结构体

结构体是将多个变量捆绑成一个类型的有效方式。

---

## 3.2 初始化 (Initialization)

### 3.2.1 定义

> "初始化"：在构造时提供初始值 -- cppreference.com

### 3.2.2 三种初始化方式

1. **直接初始化** (Direct Initialization)
2. **统一初始化** (Uniform Initialization)
3. **结构化绑定** (Structured Binding)

---

## 3.3 直接初始化 (Direct Initialization)

```cpp
#include <iostream>

int main() {
    int numOne = 12.0;
    int numTwo(12.0);

    std::cout << "numOne is: " << numOne << std::endl;
    std::cout << "numTwo is: " << numTwo << std::endl;

    return 0;
}
```

**输出**：
```
numOne is: 12
numTwo is: 12
```

`12.0` 是 `int` 吗？不是。但使用直接初始化时，C++ 不进行严格的类型检查 —— 允许窄化转换。

### 3.3.1 问题：窄化转换 (Narrowing Conversion)

```cpp
void checkCool() {
    if (temperature > 100.0) {
        std::cout << "Emergency cooling activated!" << std::endl;
    } else {
        std::cout << "Temperature normal. No emergency cooling required.";
    }
}

int main() {
    int criticalTemperature(100.8);  // 危险！
    Reactor reactor(criticalTemperature);
    reactor.checkCool();

    return 0;
}
```

C++ 说："好吧，我把 100.8 当 int 存（变成 100）"，我们可能因此产生了一个 bug。这被称为**窄化转换** —— 当较大类型的值被截断以适应较小类型时。

---

## 3.4 统一初始化 (Uniform Initialization) (C++11)

### 3.4.1 基本用法

统一初始化使用花括号 `{}`，并且 **不允许** 窄化转换：

```cpp
#include <iostream>

int main() {
    // 注意花括号！
    int numOne{12.0};     // ❌ 编译错误！窄化转换不允许
    float numTwo{12.0};   // ✅ OK

    // 修正
    int numOne{12};       // ✅ OK
    float numTwo{12.0};   // ✅ OK

    std::cout << "numOne is: " << numOne << std::endl;
    std::cout << "numTwo is: " << numTwo << std::endl;

    return 0;
}
```

### 3.4.2 统一初始化的优势

1. **安全**：不允许窄化转换，防止意外行为（或严重系统故障！）
2. **通用**：适用于所有类型 —— 向量、映射、自定义类等

### 3.4.3 用统一初始化创建 `std::map`

```cpp
#include <iostream>
#include <map>

int main() {
    // 用统一初始化创建 map
    std::map<std::string, int> ages{
        {"Alice", 25},
        {"Bob", 30},
        {"Charlie", 35}
    };

    // 访问 map 元素
    std::cout << "Alice's age: " << ages["Alice"] << std::endl;
    std::cout << "Bob's age: " << ages.at("Bob") << std::endl;

    return 0;
}
```

### 3.4.4 用统一初始化创建 `std::vector`

```cpp
#include <iostream>
#include <vector>

int main() {
    // 用统一初始化创建 vector
    std::vector<int> numbers{1, 2, 3, 4, 5};

    // 访问 vector 元素
    for (int num : numbers) {
        std::cout << num << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

---

## 3.5 结构化绑定 (Structured Binding) (C++17)

### 3.5.1 定义

- 一种从**编译时大小固定**的数据结构初始化多个变量的实用方式
- 允许一次性获取函数返回的多个值

### 3.5.2 基本示例

```cpp
#include <iostream>
#include <tuple>
#include <string>

std::tuple<std::string, std::string, std::string> getClassInfo() {
    std::string className = "CS106L";
    std::string buildingName = "Thornton 110";
    std::string language = "C++";
    return {className, buildingName, language};
}

int main() {
    auto [className, buildingName, language] = getClassInfo();
    std::cout << "Come to " << buildingName << " and join us for " << className
              << " to learn " << language << "!" << std::endl;
    return 0;
}
```

### 3.5.3 与旧式代码的对比

**不使用结构化绑定**：

```cpp
int main() {
    auto classInfo = getClassInfo();
    std::string className = std::get<0>(classInfo);
    std::string buildingName = std::get<1>(classInfo);
    std::string language = std::get<2>(classInfo);

    std::cout << "Come to " << buildingName << " and join us for " << className
              << " to learn " << language << "!" << std::endl;
    return 0;
}
```

**使用结构化绑定**（更简洁）：

```cpp
int main() {
    auto [className, buildingName, language] = getClassInfo();
    std::cout << "Come to " << buildingName << " and join us for " << className
              << " to learn " << language << "!" << std::endl;
    return 0;
}
```

### 3.5.4 结构化绑定总结

- 从编译时大小已知的数据结构初始化多个变量
- 获取函数返回的多个值
- 可用于编译时大小已知的对象（如 `std::pair`, `std::tuple`, 结构体）

---

## 3.6 引用 (References)

### 3.6.1 定义

> "引用"：声明一个名称变量作为引用。简而言之：引用是一个已存在变量的别名 -- cppreference.com

### 3.6.2 如何使用

使用 `&` 符号声明引用：

```cpp
int num = 5;
int& ref = num;

ref = 10;  // 通过引用赋予新值
std::cout << num << std::endl;  // 输出: 10
```

- `num` 是类型 `int` 的变量，被赋值 `5`
- `ref` 是类型 `int&` 的变量，是 `num` 的别名
- 当给 `ref` 赋值 `10` 时，`num` 的值也改变了，因为 `ref` 是 `num` 的别名

### 3.6.3 可视化理解

```
初始状态:
  num → [5]

声明 int& ref = num 后:
  num → [5] ← ref   (两个名字, 同一个内存位置)

执行 ref = 10 后:
  num → [10] ← ref  (两个名字, 同一个更新后的值)
```

---

## 3.7 引用传递 (Pass by Reference)

在函数参数中使用引用，函数可以直接修改传入的变量：

```cpp
#include <iostream>
#include <cmath>

// 注意 &
void squareN(int& n) {
    // 计算 n 的平方
    n = n * n;  // 不要用 std::pow 做整数平方（可能产生浮点误差）
}

int main() {
    int num = 5;
    squareN(num);
    std::cout << num << std::endl;  // 输出: 25
    return 0;
}
```

`n` 被通过引用传入 `squareN`，由 `&` 表示。这意味着 `n` 实际上指向 `num` 的内存，因此在 `squareN` 内部对 `n` 的修改会直接影响 `main` 中的 `num`。

### 3.7.1 值传递 vs 引用传递

**值传递 (Pass by Value)**：
- "嘿，复制一份，不要用原始变量！"
- 函数内修改不影响原始变量
- 对于大对象，拷贝成本可能很高

**引用传递 (Pass by Reference)**：
- "嘿，直接用实际的内存，别复制！"
- 函数内修改直接影响原始变量
- 无拷贝开销

---

## 3.8 经典的引用-拷贝 Bug

### 3.8.1 Bug 演示

```cpp
#include <iostream>
#include <cmath>
#include <vector>

void shift(std::vector<std::pair<int, int>> &nums) {
    for (auto [num1, num2] : nums) {
        num1++;
        num2++;
    }
}
```

**问题所在**：虽然 `nums` 是通过引用传入的，但 `for` 循环中的 `auto [num1, num2]` 创建的是**拷贝**（copy），不是引用。所以 `num1++` 和 `num2++` 修改的是拷贝，`nums` 内部的实际值没有改变！

### 3.8.2 Bug 修复

在结构化绑定前加上 `&`：

```cpp
#include <iostream>
#include <cmath>
#include <vector>

void shift(std::vector<std::pair<int, int>> &nums) {
    for (auto& [num1, num2] : nums) {  // 注意 auto&
        num1++;
        num2++;
    }
}
```

### 3.8.3 另一种修复方式

使用传统的索引访问：

```cpp
void shift(std::vector<std::pair<int, int>> &nums) {
    for (size_t i = 0; i < nums.size(); i++) {
        nums[i].first++;
        nums[i].second++;
    }
}
```

---

## 3.9 左值与右值 (L-values vs R-values)

### 3.9.1 概念对比

| 属性 | 左值 (l-value) | 右值 (r-value) |
|------|---------------|---------------|
| 全称 | Locator Value | Read Value |
| 相对于等号的位置 | 左侧或右侧 | 右侧 |
| 内存 | 有内存地址 | 临时值（无内存地址） |
| 示例 | `int x = 10;` 中的 `x` | `int x = 10;` 中的 `10` |
| 示例 | `int y = x;` 中的 `x` | `int y = x;` 中的 `x`（作为右值被读取） |

### 3.9.2 左值与右值的难点

```cpp
#include <iostream>
#include <cmath>

void squareN(int& n) {  // n 必须是左值引用
    n = n * n;  // 不要用 std::pow 做整数平方（可能产生浮点误差）
}

int main() {
    int num = 5;
    squareN(num);   // ✅ OK：num 是左值
    squareN(5);     // ❌ 编译错误：5 是右值（临时值）！

    std::cout << num << std::endl;
    return 0;
}
```

**原因**：
1. 右值是临时的 —— 它们在赋值后立即消失
2. 不能通过引用传递右值，因为它们是临时的
3. `squareN(int& n)` 期望一个能修改的左值，而字面量 `5` 是一个右值

---

## 3.10 `const` 关键字

### 3.10.1 定义

> "const"：用于声明对象不可修改的限定符 -- cppreference.com

### 3.10.2 快速测试

```cpp
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> vec{ 1, 2, 3 };            // 普通 vector
    const std::vector<int> const_vec{ 1, 2, 3 }; // const vector
    std::vector<int>& ref_vec{ vec };            // 对 vec 的引用
    const std::vector<int>& const_ref{ vec };    // 对 vec 的 const 引用

    vec.push_back(3);          // ✅ OK！
    const_vec.push_back(3);    // ❌ 不可以，这是 const！
    ref_vec.push_back(3);      // ✅ OK，只是一个引用！
    const_ref.push_back(3);    // ❌ 这是 const，编译错误！

    return 0;
}
```

### 3.10.3 重要规则

**不能声明一个非 const 引用来引用 const 变量**：

```cpp
const std::vector<int> const_vec{ 1, 2, 3 };
std::vector<int>& bad_ref{ const_vec };  // ❌ 错误
```

**可以声明一个 const 引用来引用 const 变量**：

```cpp
const std::vector<int> const_vec{ 1, 2, 3 };
const std::vector<int>& good_ref{ const_vec };  // ✅ 正确
```

---

## 3.11 编译 C++ 程序

### 3.11.1 你需要知道的

- C++ 是编译型语言
- 有叫做**编译器**的计算机程序
- 流行的编译器包括 `clang` 和 `g++`
- 使用 `g++` 编译程序：

```bash
g++ -std=c++23 main.cpp -o main
```

各部分含义：

| 命令部分 | 含义 |
|---------|------|
| `g++` | 编译器命令 |
| `-std=c++23` | 指定 C++ 版本 |
| `main.cpp` | 源文件 |
| `-o main` | 指定可执行文件名称 |

### 3.11.2 其他编译方式

```bash
# 不带 -o 标志，默认生成 a.out
g++ -std=c++23 main.cpp
```

### 3.11.3 为什么要了解编译？

当编写 C++ 代码时，它需要被翻译成计算机能理解的形式。像 TensorFlow 这样的大项目包含 2000+ 个源文件 —— 编译和构建工业级 C++ 软件是一个复杂的话题（将在第 8 周深入探讨）。

---

## 3.12 本章回顾

- **使用统一初始化** — 它适用于所有类型和对象！
- **引用**是给变量起别名的方式
- 引用只能绑定到左值！
- **`const`** 是确保不能修改变量的方式

---

## 📚 补充知识点

### 初始化方式的完整对比

| 初始化方式 | 语法 | 窄化转换 | 使用场景 |
|-----------|------|---------|---------|
| 拷贝初始化 | `int x = 5;` | 允许 | 简单类型，传统代码 |
| 直接初始化 | `int x(5);` | 允许 | 构造函数调用 |
| 统一初始化 | `int x{5};` | **禁止** (安全) | 推荐作为默认方式 |
| 拷贝列表初始化 | `int x = {5};` | **禁止** | 与统一初始化类似 |

**推荐**：在现代 C++ 中，优先使用统一初始化 `{}`，因为它提供了最强的类型安全保障。

### 引用与指针的区别

| 特性 | 引用 (`&`) | 指针 (`*`) |
|------|-----------|-----------|
| 可为空 | 不能（必须绑定到有效对象） | 可以 (`nullptr`) |
| 可重新绑定 | 不能 | 可以 |
| 解引用 | 自动（语法糖） | 需要 `*` 或 `->` |
| 使用安全 | 更安全（不可为空） | 需要空指针检查 |
| 内存占用 | 不一定占用内存 | 一定占用内存 |

### 结构化绑定更多用法

结构化绑定不仅适用于 `std::tuple`，还可以用于：

```cpp
// std::pair
std::pair<int, double> p{42, 3.14};
auto [i, d] = p;

// 结构体
struct Point { double x, y; };
Point pt{1.0, 2.0};
auto [x, y] = pt;

// 数组
int arr[] = {1, 2, 3};
auto [a, b, c] = arr;

// std::map 遍历
std::map<std::string, int> m{...};
for (const auto& [key, value] : m) {
    // key 有类型 const std::string&
    // value 有类型 const int&
}
```

### 编译过程详解

C++ 的编译实际上包含多个阶段：

1. **预处理 (Preprocessing)**：处理 `#include`、`#define` 等预处理指令
2. **编译 (Compilation)**：将预处理后的源码编译为汇编代码
3. **汇编 (Assembly)**：将汇编代码转换为目标文件 (`.o`/`.obj`)
4. **链接 (Linking)**：将多个目标文件和库链接成最终可执行文件

```bash
# 分步查看
g++ -E main.cpp -o main.i      # 仅预处理
g++ -S main.i -o main.s        # 仅编译（生成汇编）
g++ -c main.s -o main.o        # 仅汇编（生成目标文件）
g++ main.o -o main             # 仅链接
```

### `const` 正确性最佳实践

1. **参数**：如果不修改传入的参数，声明为 `const T&`（常量引用）
2. **成员函数**：如果不修改对象状态，声明为 `const` 成员函数
3. **局部变量**：初始化后不被修改的变量声明为 `const`
4. **返回值**：如果返回的是类内部数据的引用且不应被外部修改，返回 `const T&`

```cpp
class Example {
    std::string name_;
public:
    // const 成员函数：不修改对象
    const std::string& getName() const { return name_; }
    void setName(const std::string& name) { name_ = name; }
};
```
