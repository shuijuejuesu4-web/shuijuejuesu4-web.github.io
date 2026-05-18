---
title: "信号量：哲学家就餐问题与同步原语"
description: "信号量：哲学家就餐问题与同步原语"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 议程

* 回顾 `condition_variable_any` 及其对周三课程最后 15 分钟我们完成的哲学家就餐问题解决方案的贡献。
    * 在我看来，条件变量是最难理解的多线程指令。
    * 我想尽我的一份力，确保你现在就理解条件变量，而不是等到后面的作业中需要时才去学。
* 介绍 `semaphore`（信号量）
    * 哲学家就餐问题的最新解决方案使用了 `mutex` 和 `condition_variable_any` 的服务。它们共同提供的东西可以最好地描述为一个具有原子递增、原子递减以及整数不能变为负数的附加限制的整数。任何尝试将 0 递减的操作都会提示该线程阻塞，直到某个其他线程将其递增。
    * 哲学家就餐问题特有的计数变量代表了一种在多个竞争线程之间共享的有限资源——本质上，是允许哲学家进餐的有限数量的许可。
    * 我们可以并且将会通过定义一个 `semaphore` 类来泛化计数变量的概念，该类封装一个整数，通过 `signal` 和 `wait` 方法提供原子递增和递减操作，并无限期阻塞尝试递减围绕 0 的 `semaphore` 的线程。
        * 从概念上讲，`semaphore` 允许我们建模共享资源——剩余的许可条数量、剩余的文件描述符数量、剩余的网络连接数量等——同时使我们免受使用 `condition_variable_any` 编程带来的复杂性。`condition_variable_any` 比用它们实现的 `semaphore` 更通用，但很大一部分同步需求可以用 `semaphore` 来表达，而在我看来 `semaphore` 更容易理解。
        * 许多现代语言都提供了对线程和同步的原生支持。
            * Java 尤其从一开始就支持线程和条件变量风格的锁定。它在 21 世纪初最终添加了 `Semaphore` 类。
            * 老实说，我不确定为什么 C++11 决定排除 `semaphore`，但我认为它比 `condition_variable_any` 更容易使用，所以我将引入它，以便我们可以在后续学习中假装它就是 C++ 语言的一部分。

::: tip 重难点解析
**信号量（Semaphore）**：信号量是由 Edsger Dijkstra 在 1965 年发明的同步原语，可以说是并发编程的"始祖级"工具。它本质上是一个受保护的整数计数器，支持两个原子操作：`wait`（P 操作，来自荷兰语 "proberen"，意为"尝试"）和 `signal`（V 操作，来自荷兰语 "verhogen"，意为"增加"）。信号量的优雅之处在于，它将复杂的同步逻辑抽象为简单的计数操作——就像用令牌控制进入房间的人数一样直观。CS111 中会详细讨论信号量在操作系统调度和资源管理中的应用。
:::

::: tip 重难点解析
**信号量的内部实现：原子计数器 + 等待队列**

回顾 CS110 提供的 semaphore 实现：

```cpp
void semaphore::wait() {
  lock_guard<mutex> lg(m);
  cv.wait(m, [this]{ return value > 0; });
  value--;
}

void semaphore::signal() {
  lock_guard<mutex> lg(m);
  value++;
  if (value == 1) cv.notify_all();
}
```

拆解其底层机制：

1. **原子性保障**：`m`（mutex）保证 value 的检查和修改是原子的。没有 mutex，两个线程同时读 value=1，都通过 >0 检查，然后都执行 value--，value 变成 -1（违反不变式）。

2. **阻塞语义**：当 value==0 时，`cv.wait` 将线程放入与该 condition_variable 关联的内核等待队列。线程状态变为 TASK_INTERRUPTIBLE（可以通过 `ps` 看到进程状态为 S），不消耗 CPU。

3. **唤醒语义**：`signal()` 中 `if (value == 1)` 是一个优化——只在 value 从 0 变为正数时才唤醒等待线程。如果 value 变为更大的数（如 5），已经有一个线程被唤醒在运行，该线程会递减 value，剩余值供下一个等待线程使用。

在真实操作系统中（如 Linux），信号量通常直接使用 futex 实现，不需要额外的 mutex 和 condition_variable 包装：

```c
// 简化的内核级 sem_wait 实现
void sem_wait(sem_t *sem) {
    while (1) {
        int val = atomic_load(&sem->value);
        if (val > 0 && atomic_cas(&sem->value, val, val - 1))
            return;                          // 成功递减
        futex_wait(&sem->value, val);        // 阻塞直到值改变
    }
}
```

CS111 内核同步章节会详细讲解 futex 和原子操作在同步原语实现中的角色。
:::

# Semaphore API

* Semaphore 的 API 非常小。
    * 我们已经提到，递增和可能阻塞的递减分别称为 `signal` 和 `wait`。
    * 以下是我们自己的 `semaphore` 类的精简接口。

    ```cpp
    class semaphore {
     public:
      semaphore(int value = 0);
      void wait();
      void signal();

     private:
      int value;
      std::mutex m;
      std::condition_variable_any cv;

      semaphore(const semaphore& orig) = delete;
      const semaphore& operator=(const semaphore& rhs) const = delete;
    };
    ```

    * 你可以通过包含 `semaphore.h` 文件来使用 `semaphore`。
        * 所有的 `Makefile` 都已配置好，你可以包含它并链接到它的实现。你可以把它当作 C++ 内置类一样使用。
        * 你可以查看 `/usr/class/cs110/local/include/semaphore.h` 来确认它确实存在。

# Semaphore API（续）

* 设计决策
    * 按照设计，没有类似 `getValue` 的方法！
        * 有些 `semaphore` 设计提供了这样的方法，但我们没有。
        * 为什么省略它？在你调用它并根据它行动的这段时间里，某个其他线程很可能已经改变了它。并发指令本身不应该鼓励任何可能导致竞态条件或死锁的做法，所以我们的版本不提供。
    * 注意，三个私有数据成员类似于我们在上一版本中为限制同时抓取叉子的哲学家数量而引入的三个全局变量。
    * 我删除了拷贝构造函数和赋值运算符（使用 `delete` 关键字），因为 `mutex` 和 `condition_variable_any` 都不可拷贝构造、不可拷贝赋值，甚至不可移动。
        * 简而言之，这意味着你需要通过引用或地址来传递所有 `mutex`、`condition_variable_any` 和 `semaphore` 的实例。

::: tip 重难点解析
**为什么信号量不提供 getValue 方法？**：这是一个经典的并发设计原则——"检查-然后行动"（check-then-act）在并发环境中是不可靠的。假设信号量当前值为 1，你调用 `getValue()` 得到结果 1，然后决定执行某些操作。但在你检查值之后、执行操作之前，另一个线程可能已经调用了 `wait()` 将值变为 0。你的程序基于"信号量值为 1"这个过期信息做出了错误的决策。这个原则在操作系统设计中无处不在——任何可能被并发修改的状态，都不应该被"先检查再使用"。
:::

::: tip 重难点解析
**二元信号量（Binary Semaphore）vs 互斥锁（Mutex）**

初值为 1 的 semaphore（binary semaphore）表面行为类似 mutex，但有三个关键区别：

1. **所有权概念**：Mutex 有"所有权"——谁 lock 的谁必须 unlock。如果线程 A lock 了一个 mutex，线程 B 去 unlock 它是**未定义行为**（Pthreads 会返回 EPERM 错误）。Binary semaphore 没有所有权——线程 A 调用 `wait()`，线程 B 可以调用 `signal()`，完全合法。这使得 semaphore 适用于"生产者通知消费者"的场景，而 mutex 适用于"保护临界区"的场景。

2. **递归锁定**：Mutex 有递归版本（`recursive_mutex`），允许同一个线程多次 lock。Semaphore 没有这个概念——同一线程连续两次 `wait()` 会死锁（value 变为 0 后第二次 wait 阻塞）。

3. **优先级继承**：POSIX mutex 支持优先级继承协议（PTHREAD_PRIO_INHERIT），用于解决优先级反转问题（高优先级线程被中优先级线程阻塞，因为低优先级线程持有锁）。Semaphore 不提供此特性。

简单选择规则：保护临界区用 mutex，线程间信号/计数用 semaphore。CS111 会讨论优先级反转问题及操作系统的解决方案。
:::

# Semaphore 实现

* 实现非常简短且非常紧凑。
    * 以下是构造函数的实现：

    ```cpp
    semaphore::semaphore(int value) : value(value), {}
    ```

    * `m` 和 `cv` 通过零参数构造。
    * 数据成员按照它们在 `class` 定义中出现的顺序构造（不一定按它们在初始化列表中出现的顺序）。
    * 以下是 `wait` 和 `signal` 的实现，它们看起来很像我们之前对 `waitForPermission` 和 `grantPermission` 的实现：

    ```cpp
    void semaphore::wait() {
      lock_guard<mutex> lg(m);
      cv.wait(m, [this]{ return value > 0; });
      value--;
    }

    void semaphore::signal() {
      lock_guard<mutex> lg(m);
      value++;
      if (value == 1) cv.notify_all();
    }
    ```

    * 注意，`this` 需要被我们传递给 `cv.wait` 的即时谓词函数捕获。我们需要访问 `value` 数据成员，捕获周围对象的地址可以做到这一点。
        * `[&value]` 在 `g++` 中可以工作，但它不符合 C++11 规范，不一定能在其他编译器中工作。

::: tip 重难点解析
**Lamda 表达式中的捕获**：在 `[this]{ return value > 0; }` 中，`[this]` 捕获了当前对象的指针，使得 lambda 可以访问成员变量 `value`。这是一种安全的 C++11 标准写法。相比之下，`[&value]` 虽然在某些编译器（如 `g++`）中能工作，但它不符合 C++ 标准——因为 `value` 不是局部变量，而 lambda 的默认捕获规则对成员变量有特殊要求。这个细节提醒我们：编写可移植的 C++ 并发代码时，要注意编译器之间的差异。
:::

# 哲学家就餐问题的最终版本

* 在我看来，使用 `semaphore` 改善了叙事。
    * 剥离暴露的 `int`、`mutex` 和 `condition_variable_any`，替换为单个 `semaphore`。
    * 不再需要单独的 `waitForPermission` 和 `grantPermission` 函数。
    * 最后一次，充满感情地（代码在[这里](http://cs110.stanford.edu/autumn-2017/examples/threads-cpp/dining-philosophers-with-semaphore.cc)）：

    ```cpp
    static mutex forks[kNumForks];
    static semaphore numAllowed(kNumForks - 1);

    static void eat(unsigned int id) {
      unsigned int left = id;
      unsigned int right = (id + 1) % kNumForks;
      numAllowed.wait(); // atomic -- that blocks on attempt to decrement 0
      forks[left].lock();
      forks[right].lock();
      cout << oslock << id << " starts eating om nom nom nom." << endl << osunlock;
      sleep_for(getEatDuration());
      cout << oslock << id << " all done eating." << endl << osunlock;
      numAllowed.signal(); // atomic ++, never blocks, possibly unblocks other waiting threads
      forks[left].unlock();
      forks[right].unlock();
    }
    ```

    * 一些临别评论：
        * 与 `signal` 和 `wait` 相伴的事务性 `++` 和 `--` 很容易理解。
        * 当 `semaphore` 值为 0 时，`wait` 带来的线程让出更难理解。鉴于 `semaphore` 代表一个共享的、有限的资源，阻塞并等待直到该资源变得可用，几乎总是正确的做法。
        * 请确保你理解这种方法相对于我们最初用于避免死锁威胁的忙等待方法的诸多优点。
        * 你能想到任何情况下忙等待（也称为自旋锁，spin locking）可能是正确的方法吗？

::: tip 重难点解析
**从 mutex+CV 到 semaphore 的抽象层次提升**：回顾整个演进过程——我们从一个存在死锁风险的原始版本，迭代到了使用 semaphore 的干净版本。这个过程揭示了软件工程中的一个重要原则：选择合适的抽象层次。`semaphore` 为"有限的共享资源池"这个概念提供了恰到好处的抽象，既隐藏了底层 `mutex` 和 `condition_variable_any` 的复杂性，又足够简单易用。在 CS111 中你会看到，操作系统内核大量使用信号量来管理各种有限的系统资源。
:::

::: warning 注意事项
**自旋锁的使用场景**：忙等待（自旋锁）并非总是错误的。当临界区非常短（仅需几条指令）、且线程冲突概率很低时，自旋等待比阻塞-唤醒的上下文切换开销更小。Linux 内核中大量使用了自旋锁，这在 CS111 的调度和内核同步部分会详细讨论。但在用户态应用程序中，通常应优先使用阻塞机制，除非你经过性能分析确定自旋等待更优。
:::

# 经典的读者/写者示例

* 线程会合（Thread Rendezvous）
    * `semaphore::wait()` 和 `semaphore::signal()` 可以被利用来提供一种**不同**形式的线程通信：**会合**（rendezvous）。
    * 以下是我们的第一个示例（完整程序在[这里](http://cs110.stanford.edu/autumn-2017/examples/threads-cpp/reader-writer.cc)）：

    ```cpp
    static const unsigned int kNumBuffers = 30;
    static const unsigned int kNumCycles = 4;

    static char buffer[kNumBuffers];
    static semaphore emptyBuffers(kNumBuffers);
    static semaphore fullBuffers(0);

    static void writer() {
      cout << oslock << "Writer: ready to write." << endl << osunlock;
      for (unsigned int i = 0; i < kNumCycles * kNumBuffers; i++) {
        char ch = prepareData();
        emptyBuffers.wait();   // don't try to write to a slot unless you know it's empty
        buffer[i % kNumBuffers] = ch;
        fullBuffers.signal();  // signal reader there's more stuff to read
        cout << oslock << "Writer: published data packet with character '"
             << ch << "'." << endl << osunlock;
      }
    }

    static void reader() {
      cout << oslock << "\t\tReader: ready to read." << endl << osunlock;
      for (unsigned int i = 0; i < kNumCycles * kNumBuffers; i++) {
        fullBuffers.wait();    // don't try to read from a slot unless you know it's full
        char ch = buffer[i % kNumBuffers];
        emptyBuffers.signal(); // signal writer there's a slot that can receive data
        processData(ch);
        cout << oslock << "\t\tReader: consumed data packet "
             << "with character '" << ch << "'." << endl << osunlock;
      }
    }

    int main(int argc, const char *argv[]) {
      thread w(writer);
      thread r(reader);
      w.join();
      r.join();
      return 0;
    }
    ```

    * 将 `writer` 线程视为**服务**数据到网络连接的线程，将 `reader` 线程视为**消费**数据的线程。
    * 使用两个 `semaphore` 来同步这两个线程，以确保：
        * `reader` 永远不会领先于 `writer`，并且
        * `writer` 永远不会领先 `reader` 太多以至于覆盖尚未被消费的数据。

::: tip 重难点解析
**生产者-消费者模式（Producer-Consumer）**：读者/写者问题是经典的生产者-消费者问题的一个特例。核心挑战在于：(1) 生产者不能向已满的缓冲区写入（否则会覆盖未消费的数据），(2) 消费者不能从空的缓冲区读取（否则会读到无效数据）。两个信号量的巧妙之处在于：`emptyBuffers` 初始值为缓冲区大小（表示所有槽位都是空的），`fullBuffers` 初始值为 0（表示没有数据可读）。写者消耗"空槽位"并产生"满槽位"，读者则相反——两者通过信号量的计数值实现了完美的速度匹配。这种双信号量实现也被称为"有界缓冲区"（bounded buffer）模式，是 CS111 文件系统和管道实现的重要基础。
:::

::: tip 重难点解析
**有界缓冲区（Bounded Buffer）的深入分析**

上述双信号量看似完美，但有一个重要前提：**只有一个 reader 线程和一个 writer 线程**。如果引入多个 reader 和多个 writer，代码会暴露竞态条件：

```cpp
// 多个 writer 线程同时执行：
emptyBuffers.wait();                     // 多个 writer 都可能通过
buffer[i % kNumBuffers] = ch;            // 两个 writer 可能写入同一槽位！
fullBuffers.signal();
```

修复方案是引入额外的 mutex 保护对 buffer 和索引的访问：
```cpp
static mutex bufferLock;                 // 保护 buffer 和下标
static unsigned int writeIndex = 0;
// writer:
emptyBuffers.wait();                     // 先获取资源许可
lock_guard<mutex> lg(bufferLock);        // 再获取缓冲区互斥访问
buffer[writeIndex % kNumBuffers] = ch;
writeIndex++;
fullBuffers.signal();
```

注意锁的获取顺序至关重要：先 `wait`（可能阻塞），后 `lock`（只短暂持有）。如果顺序反过来（先 lock 再 wait），writer 在持有锁的情况下阻塞，导致所有其他 writer 和 reader 都无法访问缓冲区——死锁风险。

这揭示了一个通用设计原则：**将资源许可（semaphore）和互斥访问（mutex）分离**。resource semaphore 控制"是否可以操作"，mutex 控制"操作的互斥性"。
:::

::: tip 重难点解析
**信号量的扩展应用场景**

除了哲学家问题和生产者-消费者问题，信号量还有以下典型使用模式：

1. **连接池限制**：`semaphore connections(10)`。线程获取数据库连接前 `wait()`，归还后 `signal()`。当所有 10 个连接都在使用时，新请求线程阻塞等待归还。

2. **速率限制（Rate Limiting）**：配合定时器使用。一个后台线程每秒 `signal(N)` 次，工作线程每次处理请求前 `wait()`——限制为每秒最多 N 个请求。

3. **线程会合（Rendezvous）**：两个线程需要在某个执行点"碰头"后再继续。线程 A 到达后 `signal()` 并 `wait()`，线程 B 到达后 `signal()` 并 `wait()`。保证两者都到达后才能继续——类似屏障（barrier）的二元版本。

4. **屏障同步（Barrier）**：N 个线程都必须到达某个点后才能继续。初始化 semaphore 为 0，每个到达的线程对计数器加 1，第 N 个到达的线程 `signal()` 所有等待的线程。

5. **读写锁**：信号量和 mutex 的组合可以实现读者-写者锁（RW lock），这是下一节的扩展内容。

这些模式在 CS111 的操作系统内核（I/O 调度、进程同步、内存管理）中都有实际应用。
:::

::: tip 重难点解析
**读者-写者锁（Reader-Writer Lock）与饥饿问题**

读者-写者问题的核心是：多个读者可以同时读取共享数据（读不修改状态，因此安全），但写者需要独占访问。最简单的实现：

```cpp
static mutex rwLock;
static int numReaders = 0;

// reader:
lock_guard<mutex> lg(rwLock);
numReaders++;
if (numReaders == 1) { /* 第一个读者：阻止写者 */ }
// ... 读取 ...
numReaders--;
if (numReaders == 0) { /* 最后一个读者：允许写者 */ }

// writer:
lock_guard<mutex> lg(rwLock);
// ... 写入 ...
```

**读者饥饿（Reader Starvation）**：如果读者不断到来，numReaders 永远不归零，写者永远无法获取锁。解决方案是**写者优先**：新读者到来时，如果有写者在等待，读者阻塞，让写者先执行。代价是降低读者并发度。

**写者饥饿（Writer Starvation）**：反之，如果没有控制，读者的持续存在会使写者永远等待。就是上面的简单实现的问题。

pthread 提供了标准的 rwlock API：
```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
pthread_rwlock_rdlock(&rwlock);   // 读者获取共享锁
pthread_rwlock_wrlock(&rwlock);   // 写者获取独占锁
pthread_rwlock_unlock(&rwlock);   // 释放（读者或写者都用这个）
```

Linux 实现默认采用写者优先策略以避免写者饥饿。CS111 内核同步章节会讨论 rwlock 在内核中的广泛应用（如保护内核数据结构、文件系统的 inode 缓存等）。
:::

# 实现 `myth-buster`

* `myth-buster` 顺序版本的核心
    * 顺序地连接到所有约 30 台 `myth` 机器，向每台查询由 CS110 学生运行的进程总数。
    * 网络细节被抽象化并封装在一个库例程中，具有以下原型：

    ```cpp
    int getNumProcesses(unsigned short num, const unordered_set<string>& sunetIDs);
    ```

    * `num` 是 myth 机器编号（例如 14 代表 `myth14`），而 `sunetIDs` 是一个哈希集，包含根据我们的 `/usr/class/cs110/repos/assign3/` 目录当前所有选修 CS110 的学生的 SUNet ID。
    * 以下是 `compileCS110ProcessCountMap` 的顺序（且非常慢的）实现，它非常暴力且很像 CS106B 的风格。（它假设 `sunetIDs` 已经配置了所有 CS110 学生的 SUNet ID 集合，并进一步假设 `counts` 引用了一个初始为空的映射）。
    * 完整程序在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-cpp/myth-buster-sequential.cc)。

    ```cpp
    static unsigned short kMinMythMachine = 1;
    static unsigned short kMaxMythMachine = 32;
    static void compileCS110ProcessCountMap(const unordered_set<string>& sunetIDs,
                                            map<unsigned short, unsigned short>& counts) {
      for (unsigned short num = kMinMythMachine; num <= kMaxMythMachine; num++) {
        int numProcesses = getNumProcesses(num, sunetIDs);
        if (numProcesses >= 0) { // -1 expresses networking failure
          counts[num] = numProcesses;
          cout << "myth" << num << " has this many CS110-student processes: "
               << numProcesses << endl;
        }
      }
    }
    ```

    * 每次对 `getNumProcesses` 的调用都很慢，而累积大约 30 次顺序调用则痛苦地慢。
    * 运行顺序版本大约需要 40 秒，尽管其中 99% 的时间都花在等待网络连接的建立上。

::: tip 重难点解析
**I/O 密集型 vs CPU 密集型任务**：这个示例完美地展示了 I/O 密集型任务的特性。`getNumProcesses` 函数每次调用的大部分时间都在等待网络连接建立（I/O 等待），而不是进行计算。顺序执行的 30 次调用中，CPU 几乎完全空闲，时间都浪费在等待上。这就是多线程能大显身手的地方——当一个线程阻塞在 I/O 上时，另一个线程可以利用 CPU 继续发起新的连接请求。CS111 的调度算法部分会详细讨论 I/O 密集型和 CPU 密集型任务对调度策略的不同影响。
:::

# 引入线程

* 通过引入线程，我们将网络等待时间重叠起来。
    * 该程序的多线程版本在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-cpp/myth-buster-concurrent.cc)。
    * 将共享数据结构和同步指令移动到全局空间

    ```cpp
    static unordered_set<string> sunetIDs;
    static map<unsigned short, unsigned short> processCountMap;
    static mutex processCountMapLock;
    ```

    * 将 `thread` 包裹在顺序代码的核心周围，并使用 `semaphore` 将活跃工作的线程数量限制在一个合理的小数目（但又不能太小以至于程序又变成顺序的），这样线程管理器就不会被过多的线程所压垮。
    * 注意，我们使用了 `signal` 方法的重载版本，它接受 `on_thread_exit` 标签作为其唯一参数。调用这个第二个版本不会立即发送信号给 `semaphore`，而是安排在整个线程例程退出后、线程正在被销毁时发送信号。

    ```cpp
    static void countCS110Processes(unsigned short num, semaphore& s) {
      int numProcesses = getNumProcesses(num, sunetIDs);
      if (numProcesses >= 0) {
        processCountMapLock.lock();
        processCountMap[num] = numProcesses;
        processCountMapLock.unlock();
        cout << oslock << "myth" << num << " has this many CS110-student processes: "
             << numProcesses << endl << osunlock;
      }

      s.signal(on_thread_exit);
    }

    static unsigned short kMinMythMachine = 1;
    static unsigned short kMaxMythMachine = 32;
    static int kMaxNumThreads = 8; // number of threads beyond main thread that are permitted to exist at any one moment
    static void compileCS110ProcessCountMap() {
      vector<thread> threads;
      semaphore numAllowed(kMaxNumThreads);
      for (unsigned short num = kMinMythMachine; num <= kMaxMythMachine; num++) {
        numAllowed.wait();
        threads.push_back(thread(countCS110Processes, num, ref(numAllowed)));
      }

      for (thread& t: threads) t.join();
    }
    ```

::: tip 重难点解析
**线程池与并发度控制**：本示例中的 semaphore `numAllowed` 实现了一种简单的"线程池"效果——通过将信号量初始化为最大允许线程数（8），在创建每个线程前先执行 `wait()`（获取许可），在 thread 结束时通过 `on_thread_exit` 自动执行 `signal()`（归还许可）。这种模式也称为"信号量作为资源计数器"——它优雅地限制了并发度，防止因创建过多线程而导致系统资源耗尽。CS111 中会讨论更正式的线程池模式和调度队列实现。

注意这里的一个重要细节：`on_thread_exit` 确保了信号量的 `signal()` 在线程完全退出后才执行，而不是在线程函数返回前的某个中间时刻。如果不这样做，主线程可能在子线程还持有某些资源（如栈空间）时就误以为许可已归还，导致资源竞争。
:::
