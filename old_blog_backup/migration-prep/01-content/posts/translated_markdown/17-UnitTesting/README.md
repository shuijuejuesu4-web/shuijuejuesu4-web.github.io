# 第17章：单元测试与 C++ 测试框架

> **授课教师**：Thomas Poimenidis, Rachel Fernandez
> **学期**：Stanford CS106L, Fall 2025（选修讲座）

---

## 目录

- [17.1 什么是单元测试？](#171-什么是单元测试)
- [17.2 C++ 测试框架概览](#172-c-测试框架概览)
- [17.3 GoogleTest 入门](#173-googletest-入门)
  - [17.3.1 TEST 宏](#1731-test-宏)
  - [17.3.2 TEST_F 宏与测试夹具](#1732-test_f-宏与测试夹具)
  - [17.3.3 TEST_P 宏与参数化测试](#1733-test_p-宏与参数化测试)
- [17.4 GoogleTest 工作原理](#174-googletest-工作原理)
- [📚 补充知识点](#-补充知识点)
- [🔧 常用API参考](#-常用api参考)

---

## 17.1 什么是单元测试？

### 17.1.1 定义

> "Unit testing is a test-driven development (TDD) method for evaluating software that pays special attention to an individual component or unit of code—the smallest increment possible" — IBM

单元测试是一种测试驱动开发（TDD）方法，专注于验证**最小的可独立测试的代码单元**（通常是单个类或函数）。

### 17.1.2 单元测试 vs. 集成/系统测试

| 维度 | 单元测试 (Unit Tests) | 集成/系统测试 (Integration/System Tests) |
|------|----------------------|----------------------------------------|
| **范围** | 小范围（通常单个类或方法） | 可大可小（集成了类/方法，甚至整个系统） |
| **隔离性** | 必须隔离被测单元（无外部依赖、网络、文件系统读写），依赖通常被 mock | 不一定要隔离，通常在应用程序/系统的上下文中测试 |
| **速度** | 通常非常快 | 更长、更广泛的测试 |
| **目的** | 验证单个单元的功能 | 在更大子系统上下文中验证单元/系统/应用程序的功能 |

### 17.1.3 如何编写单元测试？

1. **确定测试单元**（Identify the Unit）
2. **选择方法**（Select an Approach）
3. **建立环境**（Establish the Environment）—— 测试框架、测试数据等
4. **创建并使用测试用例**（Create and Use Test Cases）
5. **调试并解决问题**（Debug and Resolve Issues）—— 让测试帮助你发现问题

### 17.1.4 为什么要写单元测试？

1. **尽早发现 Bug**：单元测试快速且测试小单元，可以在开发中频繁运行
2. **变更的安全网**：已有单元测试可以即时测试代码的增量变更
3. **充当文档**：单元测试本身就可以作为文档，说明哪些功能已经过测试

---

## 17.2 C++ 测试框架概览

C++ 有多个测试框架可供选择：

| 框架 | 特点 |
|------|------|
| **GoogleTest** | Google 开发，最流行，功能全面 |
| **Catch2** | 单头文件，易于集成，现代化的语法 |
| **Boost.Test** | Boost 库的一部分，功能强大但较重量级 |
| **CppTest** | 较简单的轻量级框架 |

**本章重点：GoogleTest**

---

## 17.3 GoogleTest 入门

### 17.3.1 什么是 GoogleTest？

- Google 开发的 C++ 测试框架
- 提供测试夹具（test fixtures）、参数化测试等测试功能
- 底层实现：一组**宏**和**断言**，由预处理器插入代码
- **自动测试发现**：编译时自动注册测试，无需手动管理

### 17.3.2 TEST 宏

最基本的测试定义方式：

```cpp
#include <gtest/gtest.h>

// TEST(测试套件名, 测试名)
TEST(BankAccountTest, DepositIncreasesBalance) {
    BankAccount account("Alice", 100.0);
    account.deposit(50.0);
    EXPECT_EQ(account.balance(), 150.0);
}

TEST(BankAccountTest, WithdrawDecreasesBalance) {
    BankAccount account("Bob", 200.0);
    account.withdraw(80.0);
    EXPECT_EQ(account.balance(), 120.0);
}

TEST(BankAccountTest, WithdrawMoreThanBalanceThrows) {
    BankAccount account("Charlie", 100.0);
    EXPECT_THROW(account.withdraw(200.0), std::runtime_error);
}
```

### 17.3.3 TEST_F 宏与测试夹具

当多个测试需要共享相同的初始化代码时，使用**测试夹具（Test Fixture）**：

```cpp
// 定义测试夹具类（继承自 testing::Test）
class BankAccountTest : public ::testing::Test {
protected:
    void SetUp() override {
        // 每个 TEST_F 运行前调用
        account_ = std::make_unique<BankAccount>("TestUser", 100.0);
    }

    void TearDown() override {
        // 每个 TEST_F 运行后调用
        account_.reset();
    }

    std::unique_ptr<BankAccount> account_;
};

// 使用 TEST_F（第一个参数必须是夹具类名）
TEST_F(BankAccountTest, DepositWorks) {
    account_->deposit(50.0);
    EXPECT_EQ(account_->balance(), 150.0);
}

TEST_F(BankAccountTest, InitialBalanceIsCorrect) {
    EXPECT_EQ(account_->balance(), 100.0);
}
```

#### TEST vs TEST_F

| | TEST | TEST_F |
|------|------|--------|
| 第一个参数 | 测试套件名 | 测试夹具类名 |
| 夹具 | 没有 | 继承 `testing::Test`，自动构造/析构 |
| 共享状态 | 不支持 | 通过夹具类成员变量共享 |

**夹具的作用**：消除重复的初始化和清理工作。

### 17.3.4 TEST_P 宏与参数化测试

参数化测试让你用**不同参数运行同一个测试**：

```cpp
// 参数结构体
struct WithdrawTestParam {
    double initialBalance;
    double withdrawAmount;
    double expectedBalance;
};

// 参数化测试夹具
class WithdrawAccountTest
    : public ::testing::TestWithParam<WithdrawTestParam> {
protected:
    void SetUp() override {
        account_ = std::make_unique<BankAccount>("Test", GetParam().initialBalance);
    }
    std::unique_ptr<BankAccount> account_;
};

// 参数化测试
TEST_P(WithdrawAccountTest, ValidWithdraw) {
    account_->withdraw(GetParam().withdrawAmount);
    EXPECT_EQ(account_->balance(), GetParam().expectedBalance);
}

// 实例化参数化测试（提供多组参数）
INSTANTIATE_TEST_SUITE_P(
    WithdrawTests,           // 前缀名
    WithdrawAccountTest,     // 测试夹具
    ::testing::Values(       // 参数值
        WithdrawTestParam{100.0, 50.0, 50.0},
        WithdrawTestParam{200.0, 100.0, 100.0},
        WithdrawTestParam{500.0, 1.0,  499.0},
        WithdrawTestParam{50.0,  50.0, 0.0}
    )
);
```

每提供一组参数，就会自动生成一个独立的测试！这是一种极其优雅的大规模测试实例化方式。

> 提示：可以为参数结构体重载 `operator<<`，以获得更清晰的错误信息。

---

## 17.4 GoogleTest 工作原理

### 17.4.1 宏与预处理器

GoogleTest 在底层使用宏（如 `TEST`、`EXPECT_EQ`）通过预处理器插入测试代码：

- `TEST(test_suite, test_name)` → 展开为新的测试类
- `EXPECT_EQ(a, b)` → 值比较 + 失败时记录错误并继续
- `ASSERT_EQ(a, b)` → 值比较 + 失败时立即终止当前测试

### 17.4.2 自动测试发现

GoogleTest 在编译时自动注册所有测试。`RUN_ALL_TESTS()` 会执行所有已注册的测试，无需手动维护测试清单。

### 17.4.3 组合使用示例

```cpp
// 完整的测试文件示例
#include <gtest/gtest.h>
#include "bank_account.h"

// 基础测试
TEST(BasicTest, CreateAccount) {
    BankAccount a("Me", 0.0);
    EXPECT_EQ(a.getOwner(), "Me");
}

// 夹具测试
class BankAccountFixture : public ::testing::Test {
protected:
    void SetUp() override {
        acc = std::make_unique<BankAccount>("Test", 100.0);
    }
    std::unique_ptr<BankAccount> acc;
};

TEST_F(BankAccountFixture, Deposit) {
    acc->deposit(50);
    EXPECT_EQ(acc->balance(), 150);
}

TEST_F(BankAccountFixture, Withdraw) {
    acc->withdraw(30);
    EXPECT_EQ(acc->balance(), 70);
}

// 参数化测试
class WithdrawParamTest
    : public ::testing::TestWithParam<std::tuple<double, double, double>> {};

TEST_P(WithdrawParamTest, WithdrawVarious) {
    auto [initial, amount, expected] = GetParam();
    BankAccount acc("Test", initial);
    acc.withdraw(amount);
    EXPECT_DOUBLE_EQ(acc.balance(), expected);
}

INSTANTIATE_TEST_SUITE_P(
    Values,
    WithdrawParamTest,
    ::testing::Values(
        std::make_tuple(100.0, 50.0, 50.0),
        std::make_tuple(200.0, 100.0, 100.0)
    )
);

int main(int argc, char** argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

---

## 📚 补充知识点

### 测试驱动开发 (TDD) 的红-绿-重构循环

```
    红 (Red) → 绿 (Green) → 重构 (Refactor)
        ↑                                 |
        └────────────────────────────────┘
```

1. **红**：先写一个失败的测试（因为功能还不存在）
2. **绿**：写最少的代码让测试通过
3. **重构**：改进代码质量，同时保证测试仍然通过

### 常用测试断言

GoogleTest 提供两种断言风格：

| 风格 | 失败时... | 示例 |
|------|----------|------|
| `EXPECT_*` | 记录错误并**继续**执行 | `EXPECT_EQ(a, b);` |
| `ASSERT_*` | 记录错误并**立即终止**当前测试 | `ASSERT_NE(a, b);` |

### 测试框架对比

| 特性 | GoogleTest | Catch2 | Boost.Test |
|------|-----------|--------|------------|
| 集成复杂度 | 中等（需要编译安装） | 简单（单头文件） | 复杂 |
| 测试自动注册 | 是 | 是 | 是 |
| 参数化测试 | 是 (TEST_P) | 是 | 是 |
| Mock 支持 | Google Mock | 内置 | 无 |
| 社区规模 | 最大 | 大 | 中 |
| 文档质量 | 优秀 | 优秀 | 一般 |

### 编写可测试代码的技巧

1. **依赖注入**：通过构造函数或方法参数注入依赖，而非硬编码
2. **接口隔离**：使用抽象接口，方便 mock
3. **单一职责**：每个类/函数只做一件事
4. **避免全局状态**：全局变量让测试难以独立
5. **使用 mock 隔离外部依赖**（网络、数据库、文件系统）

```cpp
// 不好的做法：硬编码依赖
class Report {
    Database db_;  // 硬编码具体类，难以 mock
};

// 好的做法：接口注入
class Report {
    IDatabase& db_;  // 接口引用，测试时可以 mock
public:
    Report(IDatabase& db) : db_(db) {}
};
```

---

## 🔧 常用API参考

### GoogleTest 断言

#### 布尔断言

| 断言 | 说明 |
|------|------|
| `EXPECT_TRUE(condition);` | condition 为 true |
| `ASSERT_TRUE(condition);` | condition 为 true（失败时中止） |
| `EXPECT_FALSE(condition);` | condition 为 false |
| `ASSERT_FALSE(condition);` | condition 为 false（失败时中止） |

#### 比较断言

| 断言 | 说明 |
|------|------|
| `EXPECT_EQ(val1, val2);` | val1 == val2 |
| `EXPECT_NE(val1, val2);` | val1 != val2 |
| `EXPECT_LT(val1, val2);` | val1 < val2 |
| `EXPECT_LE(val1, val2);` | val1 <= val2 |
| `EXPECT_GT(val1, val2);` | val1 > val2 |
| `EXPECT_GE(val1, val2);` | val1 >= val2 |

#### 浮点断言

| 断言 | 说明 |
|------|------|
| `EXPECT_FLOAT_EQ(val1, val2);` | 两个 float 相等（4ULP 内） |
| `EXPECT_DOUBLE_EQ(val1, val2);` | 两个 double 相等（4ULP 内） |
| `EXPECT_NEAR(val1, val2, abs_error);` | 差值在绝对误差范围内 |

#### 字符串断言

| 断言 | 说明 |
|------|------|
| `EXPECT_STREQ(str1, str2);` | C 字符串相等 |
| `EXPECT_STRNE(str1, str2);` | C 字符串不等 |
| `EXPECT_STRCASEEQ(str1, str2);` | C 字符串相等（忽略大小写） |

#### 异常断言

| 断言 | 说明 |
|------|------|
| `EXPECT_THROW(statement, exception_type);` | 语句抛出指定类型的异常 |
| `ASSERT_THROW(statement, exception_type);` | 同上（失败时中止） |
| `EXPECT_NO_THROW(statement);` | 语句不抛出任何异常 |
| `EXPECT_ANY_THROW(statement);` | 语句抛出任何类型的异常 |

#### 浮点谓词（Predicate）

| 断言 | 说明 |
|------|------|
| `EXPECT_PRED1(pred, val1);` | 一元谓词 |
| `EXPECT_PRED2(pred, val1, val2);` | 二元谓词 |
| `EXPECT_PRED_FORMAT1(format, val1);` | 自定义格式化的一元谓词 |

#### 死亡测试

| 断言 | 说明 |
|------|------|
| `EXPECT_DEATH(statement, regex);` | 语句导致进程以匹配正则的错误退出 |
| `ASSERT_DEATH(statement, regex);` | 同上（失败时中止） |

### Google Mock (gmock) 常用宏

`#include <gmock/gmock.h>`

```cpp
// 创建 Mock 类
class MockDatabase : public IDatabase {
public:
    MOCK_METHOD(bool, connect, (const std::string& url), (override));
    MOCK_METHOD(std::vector<Record>, query, (const std::string& sql), (override));
};

// 设置期望
EXPECT_CALL(mockDb, connect("test_url"))
    .Times(1)
    .WillOnce(Return(true));

// 参数匹配
EXPECT_CALL(mock, method(testing::_, testing::Ge(5)))
    .WillRepeatedly(Return(42));
```

### GoogleTest 命令行选项

```bash
# 运行所有测试
./test_executable

# 只运行特定测试套件
./test_executable --gtest_filter=BankAccountTest.*

# 只运行特定测试
./test_executable --gtest_filter=BankAccountTest.DepositWorks

# 列出所有测试（不运行）
./test_executable --gtest_list_tests

# 重复运行测试（检测不稳定性）
./test_executable --gtest_repeat=10

# 随机顺序运行
./test_executable --gtest_shuffle

# 输出 XML 报告
./test_executable --gtest_output=xml:report.xml
```

### CMake 中集成 GoogleTest

```cmake
# 方式一：FetchContent（推荐）
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.14.0
)
FetchContent_MakeAvailable(googletest)

enable_testing()

add_executable(my_tests test_main.cpp)
target_link_libraries(my_tests PRIVATE gtest_main gmock_main)

include(GoogleTest)
gtest_discover_tests(my_tests)
```

### Catch2 快速参考（备选框架）

```cpp
#include <catch2/catch_all.hpp>

TEST_CASE("BankAccount operations", "[bank]") {
    BankAccount acc("Test", 100.0);

    SECTION("deposit increases balance") {
        acc.deposit(50.0);
        REQUIRE(acc.balance() == 150.0);
    }

    SECTION("withdraw decreases balance") {
        acc.withdraw(30.0);
        REQUIRE(acc.balance() == 70.0);
    }
}
```
