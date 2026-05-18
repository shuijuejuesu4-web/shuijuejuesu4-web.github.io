---
title: "线程安全：竞争条件与C++线程"
description: "线程安全：竞争条件与C++线程"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告
* 今日课程
    * 回顾上周三的第一个多线程示例，强调多进程和
      多线程之间的相似之处，指出不同之处。
    * 完成一个多线程程序的两个版本，其中主执行线程与
      它创建的每个子线程共享数据。
        * 版本 1 有意地被破坏了，以展示最明显的竞争条件（race condition）。
        * 版本 2 提供了一个非常简单的修复，这样竞争条件就不复存在了。
    * 说明我从 C 转向 C++ 的动机，因为 C++ 对线程的处理更加健壮且不易出错。
    * 完成两到三个 C++ 线程示例。

* 指定阅读
    * 你们现在正在阅读第 12.1 节和第 12.3 节到第 12.8 节的全部内容，
      跳过那些涉及网络和服务器的子节。
        * 第 12 章实际上是教材的第四章。
        * 代码示例是用 C 写的，但概念在很大程度上与语言无关。
    * 等我们学到网络部分时，我们会回过头来看其中一些被排除的章节。

# 公告
* 其他公告
    * Assignment 3 今晚截止，就在午夜前。
    * Assignment 4 也已发布，截止日期为下周二（明天起一周后）晚上。
    * 期中考试定于 11 月 3 日（星期五），在正常上课时间，地点在 CEMEX Auditorium。

# 重温内向者（Introverts）示例
* C++ 现在将线程和同步指令作为语言的一部分提供。
    * 以下是我们之前见过的 `introverts` 示例——这次是用 C++ 写的！
      （此程序的完整版本在[这里](http://cs110.stanford.edu/autumn-2017/examples/threads-cpp/introverts.cc)）。

    ```cpp
    #include <iostream>       // for cout, endl
    #include <thread>         // for C++ thread support
    #include "ostreamlock.h"  // for CS110 iomanipulators (oslock, osunlock) used to lock down streams
    using namespace std;

    static void recharge() {
      cout << oslock << "I recharge by spending time alone." << endl << osunlock;
    }

    static const size_t kNumIntroverts = 6;
    int main(int argc, char *argv[]) {
      cout << "Let's hear from " << kNumIntroverts << " introverts." << endl;      
      thread introverts[kNumIntroverts]; // declare array of empty thread handles
      for (thread& introvert: introverts)
        introvert = thread(recharge);    // move anonymous threads into empty handles
      for (thread& introvert: introverts)
        introvert.join();    
      cout << "Everyone's recharged!" << endl;
      return 0;
    }    
    ```

# 重温内向者（Introverts）示例
* C++ 将线程和同步指令作为语言的一部分提供。
    * 特性：
        * 我们声明了一个空的（即不可 join 的）`thread` 句柄数组，就像我们
          在 C 版本中所做的那样。（`thread` 是一个相对新加入 C++ 的类，而
          `thread` 文档页面的"终极参考"在
          [这里](http://en.cppreference.com/w/cpp/thread/thread)。）
        * 我们将 `recharge` 函数安装到临时线程中，
          然后（通过 `operator=(thread&& other)`）将其移动到一个在此之前
          一直为空的 `thread` 句柄中。
            * 这是一种新形式的 `operator=`，它将右侧
              `thread` 的内容完全移植到左侧的 `thread`
              中，使右侧的 `thread` 变成完全被掏空的状态，就好像它是通过零参数构造函数
              创建的一样（即一个空的、不可 join 的 `thread` 句柄）。
            * 这是一个重要的澄清，因为传统的 `operator=`
              会产生同一个 `thread` 的第二个工作拷贝，而我们不想要那样。我们
              想要的是将原数组中的一个 `thread` 初始化为一个
              *现在运行代码*的线程。这就是我们实现它的方法。
        * `join` 方法，毫不意外地，等同于我们之前学过的 `pthread_join`
          函数。
        * 线程例程的原型——在这个例子中是 `recharge`——可以是
          任何形式（尽管返回类型总是被忽略，所以应为 `void`，除非
          同一个函数还在非 `thread` 上下文中使用，并且在那里返回值是有用的）。
        * 重要点：`operator<<`，与 `printf` 不同，**不是**线程安全的。
            * 更糟糕的是：一系列菊花链式的 `operator<<` 调用绝对不
              保证作为单个厚重的原子事务来执行。
            * 简单的解决方案：我构造了流操纵器 `oslock` 和
              `osunlock` 来锁定并随后释放对 `ostream` 的访问，
              这样两个独立的线程永远不会同时尝试向一个流中插入字符数据。
              本质上，`oslock` 和 `osunlock`
              构建了我们的第一个*临界区（critical region）*，即一段必须在
              任何其他线程进入该区域之前完整执行的代码块，
              否则我们会看到同步问题的证据。

::: tip 重难点解析
**C++ 移动语义与 `thread` 句柄**：`thread` 对象是不可复制的（copy constructor 被删除），但它是可移动的（move constructor 和 move assignment 被定义）。这意味着你不能写 `thread t2 = t1;`（编译错误），但可以写 `t2 = std::move(t1);`。代码中 `introvert = thread(recharge)` 的模式创建了一个临时 `thread` 对象（右值），然后通过移动赋值将其"转移"到数组元素中。移动后，临时对象变为空状态（不可 join），而数组元素持有了实际运行的线程。这种设计防止了多个 `thread` 对象同时管理同一个底层线程，避免了双重 join 或双重 detach 的 bug。

**`printf` 线程安全 vs `cout` 非线程安全**：这是 C 和 C++ 的一个重要差异。`printf` 在大多数实现中是线程安全的——因为 C 标准要求 `printf` 对 `FILE*` 流加锁。但 C++ 的 `cout` 标准并未要求每个 `operator<<` 调用是原子的。这意味着 `cout << a << b;` 可能被另一个线程的 `cout << c << d;` 插入，产生类似 `acbd` 的交错输出。`oslock`/`osunlock` 通过互斥锁（mutex）解决了这个问题，确保整个输出链作为原子操作执行。这也是本课程后续深入讲解 mutex 的引子。
:::

::: tip 重难点解析
**`std::thread` 的内部实现 — 它本质上包装了 `pthread_t`**

在 Linux 上，`std::thread` 不是魔法——它内部封装了一个 `pthread_t`，并通过 RAII（Resource Acquisition Is Initialization）管理线程的生命周期。

```cpp
// std::thread 的简化内部结构（libstdc++ 实现）
class thread {
    pthread_t _M_id;  // 底层 pthread 句柄
    // ... 其他实现细节 ...
public:
    // 构造函数：创建线程
    template<typename _Callable, typename... _Args>
    explicit thread(_Callable&& __f, _Args&&... __args) {
        // 包装参数，调用 pthread_create
        // pthread_create(&_M_id, NULL, wrapper_func, args_pack);
    }

    // 析构函数：RAII 的关键
    ~thread() {
        if (joinable()) {
            std::terminate();  // 毁灭性设计！见下文
        }
    }
};
```

**`std::thread` 的关键设计决策**：

1. **为何析构时调用 `std::terminate()` 而非自动 join/detach？**
   如果析构时线程仍在运行且可 join，调用 `join()` 会阻塞——在析构函数中阻塞是危险的（可能死锁、掩盖 bug）。调用 `detach()` 会让线程在对象销毁后继续运行——可能导致悬垂引用（dangling reference）。C++ 标准委员会选择了"立即终止"作为最安全的默认行为，强制程序员显式决定 join 还是 detach。

2. **与 `pthread_t` 的关键区别 — RAII**：
   `pthread_t` 是纯 C 类型，没有任何资源管理——你需要手动 `pthread_join` 或 `pthread_detach`。`std::thread` 通过 RAII 确保线程资源被正确清理：析构时如果线程未 join 也未 detach，程序立即 `terminate()`，强制你编写正确的资源管理代码。

3. **`native_handle()` 方法**：可以获取底层的 `pthread_t` 句柄用于那些 C++ 未封装的 POSIX 操作：
   ```cpp
   std::thread t(func);
   pthread_t handle = t.native_handle();  // 获取底层 pthread_t
   pthread_setaffinity_np(handle, ...);   // 设置 CPU 亲和性
   ```
:::

::: tip 重难点解析
**移动语义与线程所有权的深层原理**

`std::thread` 的移动语义体现了**独占所有权（unique ownership）** 的资源管理模式，与 `std::unique_ptr` 的设计理念相同。

**为什么拷贝构造被删除？**

一个 `pthread_t` 标识内核中的一条执行流。如果允许拷贝：
```cpp
thread t1(func);
thread t2 = t1;  // 如果允许：两个对象管理同一个内核线程
// t1.join();    // 第一个 join 成功
// t2.join();    // 第二个 join 会怎样？双重 join 是未定义行为！
```
这会违反"每个内核线程有唯一的管理者"这一约束。

**移动构造的实现（简化）**：
```cpp
thread(thread&& other) noexcept {
    _M_id = other._M_id;       // 窃取底层 pthread_t 句柄
    other._M_id = pthread_t(); // 将源对象置为空（默认构造的 pthread_t）
}
// 移动后：
// - *this 持有原线程的所有权
// - other 变为"空线程"（not joinable）
```

**为什么移动是 noexcept？**

移动操作标记为 `noexcept` 很重要——如果移动可能抛出异常，`std::vector<thread>` 在重新分配内存时无法安全地将线程元素移动到新缓冲区，因为失败的移动会导致线程句柄丢失。`noexcept` 保证了移动总是成功的，使 `thread` 可以被安全地存储在 STL 容器中。
:::

# 线程例程可以以类型安全的方式进行配置
* 线程例程可以接受任意数量的参数。
    * 可变参数列表——等同于 C 语言中的省略号——通过一种新的
      C++ 特性（称为[可变参数模板（variadic templates）](http://www.cplusplus.com/articles/EhvU7k9E/)）得到支持。
    * 这意味着我们在如何设计线程例程的原型方面有相当大的灵活性。
        * 毫无疑问，这比 `pthreads` 提供的不安全的 `void` `*`
          古怪做法要好。
    * 以下是一个稍微复杂一点的示例，其中 `greet` 线程被配置为
      以不同的次数说 hello。（以下程序的在线版本可以在
      [这里](http://cs110.stanford.edu/autumn-2017/examples/threads-cpp/greeters.cc)找到）。

    ```cpp
    static void greet(size_t id) {
      for (size_t i = 0; i < id; i++) {
        cout << oslock << "Greeter #" << id << " says 'Hello!'" << endl << osunlock;
        struct timespec ts = {
          0, random() % 1000000000
        };
        nanosleep(&ts, NULL);
      }    
      cout << oslock << "Greeter #" << id << " has issued all of his hellos, "
           << "so he goes home!" << endl << osunlock;
    }

    static const size_t kNumGreeters = 6;
    int main(int argc, char *argv[]) {
      srandom(time(NULL));
      cout << "Welcome to Greetland!" << endl;    
      thread greeters[kNumGreeters];
      for (size_t i = 0; i < kNumGreeters; i++)
        greeters[i] = thread(greet, i + 1);
      for (thread& greeter: greeters)
        greeter.join();
      cout << "Everyone's all greeted out!" << endl;
      return 0;
    }
    ```

::: tip 重难点解析
**可变参数模板与类型安全**：`std::thread` 的构造函数使用了可变参数模板（variadic templates），这意味着你可以写 `thread(greet, 42, "hello", 3.14)`，编译器会在编译期检查参数类型是否与 `greet` 的函数签名匹配。这与 pthreads 的 `void*` 参数传递形成鲜明对比——在 pthreads 中，你必须将所有参数打包成一个 struct，传递其指针，然后在函数内部解包。这种方式不仅繁琐，而且完全没有类型检查，极易出错。C++ 的线程 API 在类型安全上是一个巨大的进步。

**`nanosleep` 的作用**：程序中使用 `nanosleep` 来模拟每个线程"说话"之间的随机延迟。这个延迟是关键的教学手段——它增加了线程执行顺序的不确定性，使得每次运行程序时输出的顺序都可能不同。如果没有这个随机延迟，在单核 CPU 上线程可能碰巧按创建顺序执行，掩盖了并发的不确定性。
:::

# 售票代理和机票销售
* 多个线程通常被创建出来，以便它们能够细分并共同解决一个更大的问题。
    * 考虑这样一个场景：在 United Airlines 有 10 个售票代理接听电话，共同
      销售 1000 张机票（查看完整程序请
      点击[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-cpp/tickets.cc)）：

    ```cpp
    static const unsigned int kBaseIDNumber = 101;
    static const unsigned int kNumAgents = 10;
    static const unsigned int kNumTickets = 1000;
    static mutex ticketsLock;
    static unsigned int remainingTickets = kNumTickets;

    static void ticketAgent(size_t id) {
      while (true) {
        ticketsLock.lock();
        if (remainingTickets == 0) break;
        remainingTickets--;
        cout << oslock << "Agent #" << id << " sold a ticket! (" << remainingTickets 
             << " more to be sold)." << endl << osunlock;
        ticketsLock.unlock();
        if (shouldTakeBreak()) 
          takeBreak();
      }
      ticketsLock.unlock();
      cout << oslock << "Agent #" << id << " notices all tickets are sold, and goes home!" 
           << endl << osunlock;
    }

    int main(int argc, const char *argv[]) {
      thread agents[kNumAgents];
      for (size_t i = 0; i < kNumAgents; i++)
        agents[i] = thread(ticketAgent, kBaseIDNumber + i);
      for (thread& agent: agents)
        agent.join();
      cout << "End of Business Day!" << endl;
    }
    ```

    * 是的，我们使用了全局变量，但我们的解释是：多个线程都
      需要访问一个共享资源。`static` 全局变量在
      多线程程序中并不罕见（尽管更大的程序可能会
      将它们打包在专用的库或模块中）。
    * 在这个程序中有一个重要的临界区（critical region），我们使用一个 `mutex`
      对象来标记它的开始和结束。
        * 我们将首先在不使用 `mutex` 的情况下编写这个程序，以说明
          其中的问题是什么。
        * 然后我们将插入 `mutex`，以精确理解它如何解决这些问题。

::: tip 重难点解析
**互斥锁（Mutex）与临界区**：这是本课程最重要的概念之一。

**临界区（Critical Region）**：指访问共享资源（如全局变量 `remainingTickets`）的代码段。临界区必须被保护，确保同一时刻只有一个线程能够执行其中的代码，否则会出现数据竞争（data race）。

**没有 mutex 会怎样？** 假设 10 个线程同时执行 `remainingTickets--`。这个看似简单的操作实际上包括三个步骤：(1) 从内存加载值到寄存器；(2) 寄存器减 1；(3) 把结果存回内存。如果两个线程同时执行这些步骤，可能出现：
- 线程 A 加载 remainingTickets = 100
- 线程 B 加载 remainingTickets = 100
- 线程 A 减为 99 并存入
- 线程 B 减为 99 并存入
- 结果：两次售出操作只减少了一张票！

这就是经典的"丢失更新"（lost update）问题。mutex 的 `lock()`/`unlock()` 确保了这些步骤作为原子操作执行——一个线程在锁内时，其他线程必须等待。

**`break` 前的 unlock 陷阱**：注意代码中 `if (remainingTickets == 0) break;` 在 `lock()` 之后但在 `unlock()` 之前。如果直接 break，锁不会被释放，其他线程将永远阻塞。这就是为什么 `break` 之后（循环外）有一个额外的 `ticketsLock.unlock()` ——确保锁在任何退出路径上都能被释放。C++ 提供了 `lock_guard` 和 `unique_lock` 来自动管理锁的释放，避免这种手动 unlock 的易错模式，后续课程会涉及。

**CS111 衔接**：互斥锁是解决临界区问题的最基本工具，但 CS111 会进一步探讨其局限性——包括死锁（deadlock，多个锁的循环等待）、优先级反转（priority inversion）、以及更高级的同步原语如条件变量（condition variables）和信号量（semaphores）来解决生产者-消费者等问题。
:::

::: tip 重难点解析
**`std::mutex` 的内部实现 — 它包装了 `pthread_mutex_t`**

在 Linux 上，`std::mutex` 并非从零实现——它内部封装了一个 `pthread_mutex_t`，利用 futex 机制实现高效同步。`std::mutex` 的增值在于提供了 C++ 的类型安全和 RAII 包装器（`lock_guard` 和 `unique_lock`）。

**`std::lock_guard` — 最简单的 RAII 锁包装器**：

```cpp
template <typename Mutex>
class lock_guard {
    Mutex& _M_mutex;
public:
    explicit lock_guard(Mutex& m) : _M_mutex(m) { _M_mutex.lock(); }
    ~lock_guard() { _M_mutex.unlock(); }
    // 禁止拷贝和移动
    lock_guard(const lock_guard&) = delete;
    lock_guard& operator=(const lock_guard&) = delete;
};
```

使用方式：
```cpp
std::mutex m;
void safe_function() {
    std::lock_guard<std::mutex> lg(m);  // 构造时 lock
    // ... 临界区代码 ...
}  // 析构时自动 unlock，即使发生异常也能正确释放
```

**`std::unique_lock` — 更灵活的 RAII 锁包装器**：

`unique_lock` 比 `lock_guard` 功能更强，代价是稍大的内存占用和轻微的性能开销。关键区别：

| 特性 | `lock_guard` | `unique_lock` |
|------|-------------|---------------|
| 自动 lock/unlock | ✓ | ✓ |
| 手动 unlock（提前释放） | ✗ | ✓ `unlock()` |
| 延迟锁定（defer lock） | ✗ | ✓ `std::defer_lock` |
| 尝试锁定（try lock） | ✗ | ✓ `std::try_to_lock` |
| 可移动（movable） | ✗ | ✓ |
| 配合条件变量使用 | ✗ | ✓ 必须使用 |
| 内存开销 | 最小（仅一个引用） | 额外存储一个 `bool` 标记所有权 |

```cpp
std::mutex m;
std::condition_variable cv;
bool ready = false;

// unique_lock 与条件变量的配合（lock_guard 无法做到）
void consumer() {
    std::unique_lock<std::mutex> lk(m);  // 锁定
    cv.wait(lk, []{ return ready; });     // wait 内部会 unlock 并睡眠，被唤醒后重新 lock
    // ... 继续处理（此时锁已被 wait 重新获取）
}  // 析构时 unlock

// unique_lock 的移动语义
std::unique_lock<std::mutex> acquire_lock(std::mutex& m) {
    std::unique_lock<std::mutex> lk(m);  // 锁定
    return lk;  // 移动返回，转移锁的所有权给调用者
}
```

**经验法则**：优先使用 `lock_guard`（更简单、更轻量）。只有当需要与条件变量配合、需要手动提前 unlock、或需要转移锁的所有权时，才使用 `unique_lock`。
:::

::: tip 重难点解析
**C++ 内存序（Memory Order）基础 — 为什么 mutex 提供 acquire-release 语义**

在多线程编程中，不仅要防止竞争条件，还要确保修改对其它线程**可见**。CPU 和编译器可能为了性能而重排指令——这在线程间会导致反直觉的行为。

**问题场景**：
```cpp
// 线程 A 写入，线程 B 读取
int data = 0;
bool flag = false;

// 线程 A
data = 42;
flag = true;  // 编译器或 CPU 可能先执行这行！

// 线程 B
while (!flag);
printf("%d\n", data);  // 可能打印 0！（因为 data=42 还没对 B 可见）
```

**mutex 的 acquire-release 语义**：

当线程 A 执行 `mutex.unlock()` 时，它会执行一个 **release** 操作——将所有在此锁保护下的写操作"发布"到一个全局可见的状态。

当线程 B 执行 `mutex.lock()` 时，它会执行一个 **acquire** 操作——"获取"所有之前的 release 操作发布的修改。

```cpp
int data = 0;
std::mutex m;

// 线程 A
{
    std::lock_guard<std::mutex> lk(m);
    data = 42;
}  // unlock → release：确保 data=42 对后续 acquire 可见

// 线程 B
{
    std::lock_guard<std::mutex> lk(m);  // lock → acquire：保证看到 data=42
    printf("%d\n", data);  // 一定输出 42
}
```

**`std::memory_order` 枚举**（`std::atomic` 使用）：

| 内存序 | 含义 |
|--------|------|
| `memory_order_relaxed` | 无同步保证，仅保证原子性 |
| `memory_order_acquire` | 后续读写不会被重排到此操作之前 |
| `memory_order_release` | 之前的读写不会被重排到此操作之后 |
| `memory_order_acq_rel` | 同时具有 acquire 和 release 语义 |
| `memory_order_seq_cst` | 顺序一致性（默认，最严格，所有线程看到相同的操作顺序） |

在 CS110 中，你主要使用 mutex 进行同步（它隐式提供了 acquire-release 语义），不需要直接使用 `memory_order`。但在 CS111 和更高级的并发编程中，lock-free 数据结构需要你显式控制内存序以实现更高性能。
:::
