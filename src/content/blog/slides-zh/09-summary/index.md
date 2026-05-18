---
title: "多线程入门：pthreads与并发编程基础"
description: "多线程入门：pthreads与并发编程基础"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告
* 今日：完成多进程的讨论，启动多线程和并发的学习。
    * Assignment 3 于周一晚上截止。
    * Assignment 4 也于周一晚上发布，截止日期为八天后。
        * 可能会在周日晚发布作业说明和起始仓库，
          请留意网上更新。
    * B&O 阅读：第 12 章，跳过 12.2 节。（第 12 章是
      教材四个章节中的第四章。）
    * 我将花 10 到 15 分钟回顾虚拟内存（virtual memory）、页面（pages）、TLB（Translation Lookaside Buffer，快表）、
      调度器（schedulers）、上下文切换（context switches）、进程控制块（Process Control Block，PCB）以及就绪和阻塞 PCB 队列。我讨论
      所有这些是因为：
        * 我想让你理解每个进程如何能够像拥有全部内存一样运行，即使
          有数百个其他进程也以同样的方式运行。
        * 我想让你理解多个进程如何能够同时执行，即使
          你使用的机器只有很少的 CPU。
    * 在最后一小时左右，我将用 C 和 `pthreads` 教你少量的线程知识。
        * 教材中的示例引用了 `pthreads`，这就是为什么我想让
          你们在课堂上至少看到一点点它，然后再放弃它转向 C++。
        * B&O 第 12 章中教授的概念（你们将在
          接下来的两周内阅读）在所有语言中都是相关的，
          所以不要认为阅读材料没有帮助。
        * 我也会用 `pthreads` 展示最简单的并发问题，
          这样我稍后可以论证为什么 C++ 线程——也就是我们将花大部分时间学习的——
          让事情变得更容易（或者至少不那么困难，因为任何与
          线程相关的事情都非常具有挑战性）。
    * 一旦我完成三个小的 `pthreads` 示例，就开始 C++ 线程。
    * 请务必查看 `/usr/class/cs110/lecture-examples/autumn-2017/threads-c` 中的代码示例，
      以查看并实践接下来幻灯片中所有内容的工作版本。

# 什么是线程？
* 线程是单个进程内的一条独立的执行流。
    * 操作系统和/或编程语言允许进程将自身
      分割成两个或多个并发执行的函数。
    * 允许进程内并发（在多处理器和/或多核机器上
      甚至可以实现真正的并行）
        * 栈段被细分为多个微型栈。
        * 线程管理器在同时运行的线程之间进行时间分片和切换，
          其方式与内核调度器在进程之间切换的方式非常相似。
          （事实上，线程通常被称为**轻量级进程（lightweight processes）**）。
        * 主要区别：线程有独立的栈，但它们都共享访问
          相同的文本段（text）、数据段（data）和堆段（heap）。
            * 优点：更容易支持线程之间的通信，因为可访问的地址空间
              大体相同。
            * 优点：多个线程可以访问相同的全局数据和一个代码副本。
            * 优点：一个线程可以通过指针和引用与其他线程
              共享其栈空间。
            * 缺点：没有受保护的内存，因为地址空间是共享的。内存错误
              更常见，调试也更为困难。
            * 缺点：多个线程可以访问相同的全局数据和一个代码副本。
            * 缺点：一个线程可以通过指针和引用与其他线程
              共享其栈空间。

::: tip 重难点解析
**进程 vs 线程**：这是操作系统中最基础也最重要的对比之一。

| 维度 | 进程 | 线程 |
|------|------|------|
| 地址空间 | 独立（隔离） | 共享（除栈外） |
| 通信方式 | IPC（管道、信号、共享内存等） | 直接读写共享内存 |
| 创建开销 | 大（复制整个地址空间） | 小（只需分配栈） |
| 上下文切换 | 开销大（切换页表、刷新 TLB） | 开销小（同一进程内切换） |
| 隔离性 | 强（一个进程崩溃不影响其他） | 弱（一个线程崩溃可能导致整个进程崩溃） |
| 内核参与 | 每个进程独立 PCB | 多个线程共享一个 PCB（轻量级） |

"线程是轻量级进程"的比喻很贴切：线程在内核中的表示比进程简单得多，因为它们共享了大部分资源。但代价是隔离性下降——一个线程的野指针可以破坏另一个线程的数据。CS111 中会深入讨论内核级线程 vs 用户级线程的调度模型。
:::

# ANSI C 不提供对线程的原生支持。
* 但 POSIX 线程（即 `pthreads`）随所有标准
  UNIX 和 Linux 的 `gcc` 安装一起提供。
    * 主要的 `pthreads` 数据类型是 `pthread_t`，它是
      一种不透明类型（opaque type），帮助管理一个 C 函数如何在其自己的
      执行线程中运行。
    * 我们关心的唯一 `pthreads` 函数（在转向
      C++ 线程之前）是 `pthread_create` 和 `pthread_join`。
    * 以下是一个非常小的示例程序（在线版本在[此处](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-c/introverts.c)）：

    ```c
    #include <stdio.h>    // provides printf, which is thread-safe
    #include <pthread.h>  // provides pthread_t type, thread functions

    static void *recharge(void *args) {
      printf("I recharge by spending time alone.\n"); // printf is thread-safe
      return NULL;
    }

    static const size_t kNumIntroverts = 6;
    int main(int argc, char *argv[]) {
      printf("Let's hear from %zu introverts.\n", kNumIntroverts);
      pthread_t introverts[kNumIntroverts];
      for (size_t i = 0; i < kNumIntroverts; i++)
        pthread_create(&introverts[i], NULL, recharge, NULL);
      for (size_t i = 0; i < kNumIntroverts; i++)
        pthread_join(introverts[i], NULL);
      printf("Everyone's recharged!\n");
      return 0;
    }
    ```
    * 上述程序声明了一个包含六个 `pthread_t` *句柄*的数组。
    * 程序然后（通过 `pthread_create`）初始化每个 `pthread_t`，
      将 `recharge` 函数安装为每个线程在
      执行时应遵循的例程。
        * 所有线程函数必须接受一个 `void` `*` 参数
          并返回一个同样的指针。这就是 C 语言中的泛型编程。
        * `pthread_create` 的第二个参数允许设置线程属性（线程优先级等）。
          我们将始终传入 `NULL` 以接受默认值。
        * 第四个参数在线程启动后被原封不动地传递给线程例程。
          在这个例子中，没有参数，所以我们选择传入 `NULL`。
        * 每个 `recharge` 线程在它所属于的 `pthread_t`
          被完全配置的那一刻就有资格获得处理器时间。
    * 六个线程争夺线程管理器的注意力，我们几乎无法控制
      线程管理器做出的选择。
        * 第 0 个线程*大概*会首先获得处理器时间。
        * "大概"和"保证"是两个不同的词。
        * 对于线程管理器可能采用的调度排列，我们没有官方的说明。
    * `pthread_join` 对于线程来说，就像 `waitpid` 对于进程一样。
      主线程阻塞直到参数线程退出。（关键区别：等待的线程
      不必是产生被阻塞线程的线程）。

::: tip 重难点解析
**`pthread_create` 函数签名**：
```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```
四个参数的含义：
1. `thread`：输出参数，成功创建后存储线程 ID
2. `attr`：线程属性（栈大小、调度策略等），通常传 NULL 使用默认值
3. `start_routine`：线程入口函数指针，签名必须为 `void* (*)(void*)`
4. `arg`：传递给入口函数的参数，可以是任意指针（通过 `void*` 消除类型）

**`pthread_join` vs `waitpid` 的关键区别**：`waitpid` 只能由父进程等待其子进程；而 `pthread_join` 允许同一进程内的任何线程等待任何其他线程——没有"父子"关系的约束。这为线程间的同步协作提供了更大的灵活性。

**线程调度的不确定性**：六个线程被创建后，哪个先执行完全取决于操作系统的线程调度器。即使你在循环中按 0、1、2...的顺序创建线程，也不能保证线程 0 第一个执行。这是一种"非确定性"行为——同一程序多次运行可能产生不同的输出顺序。这是并发编程的核心挑战之一。
:::

::: tip 重难点解析
**`pthread_create` 的内核实现 — clone() 系统调用与资源共享**

`pthread_create` 不是通过 `fork` 实现的。在 Linux 上，`pthread_create` 底层调用 `clone()` 系统调用——这是比 `fork` 更通用的创建执行流的方式。

**`fork` vs `clone` 的本质区别**：

`fork` 创建一个**独立的进程**：新的地址空间（通过 CoW），独立的文件描述符表，独立的信号处理设置。

`clone` 允许调用者**精确控制哪些资源共享，哪些复制**，通过标志位掩码：

```c
// pthread_create 底层调用 clone 的等价代码（简化）
int clone_flags = CLONE_VM          // 共享虚拟地址空间 (mm_struct)
                | CLONE_FILES       // 共享文件描述符表
                | CLONE_FS          // 共享文件系统信息（cwd, umask）
                | CLONE_SIGHAND     // 共享信号处理器表
                | CLONE_THREAD      // 放入同一线程组
                | CLONE_SYSVSEM     // 共享 SysV 信号量撤销值
                | CLONE_SETTLS      // 设置线程局部存储（TLS）
                | CLONE_PARENT_SETTID
                | CLONE_CHILD_CLEARTID;
clone(thread_routine, child_stack, clone_flags, arg);
```

**关键标志的含义**：

| 标志 | 含义 | 共享的数据结构 |
|------|------|---------------|
| `CLONE_VM` | 共享虚拟内存 | `mm_struct`（页表、地址空间） |
| `CLONE_FILES` | 共享文件描述符 | `files_struct`（fd 表） |
| `CLONE_SIGHAND` | 共享信号处理器 | `sighand_struct`（处理器表、信号掩码） |
| `CLONE_THREAD` | 加入同一线程组 | 同一 tgid（thread group id = pid） |

**内核创建线程的具体步骤**：
1. 分配新的 `task_struct`（内核中的"任务"描述符）
2. 分配内核栈（通常 2 页 = 8KB 或 16KB，取决于配置）
3. **不分配**新的 `mm_struct` —— 而是将新线程的 `mm` 指针指向与创建者相同的 `mm_struct`，并递增引用计数（因为 `CLONE_VM`）
4. **不复制**文件描述符表 —— 同样共享（因为 `CLONE_FILES`）
5. 将新 `task_struct` 加入调度器的就绪队列

这意味着线程的创建成本远低于进程：无需复制页表（即使 CoW 也需要设置页表结构），无需复制 fd 表。这就是为什么"线程是轻量级进程"的说法成立。
:::

# 竞争条件（Race Conditions）
* 以下是一个稍微复杂一点的程序！
    * 外向者（Extroverts）轮到了！
    * 下面内容的在线版本在
      [此处](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-c/confused-extroverts.c)。

    ```c
    static const char *kExtroverts[] = {
      "Albert Chon",
      "John Carlo Buenaflor",
      "Jessica Guo",
      "Lucas Ege",
      "Sona Allahverdiyeva",
      "Yun Zhang",
      "Tagalong Introvert Jerry Cain"
    };
    static const size_t kNumExtroverts = sizeof(kExtroverts)/sizeof(kExtroverts[0]) - 1;

    static void *recharge(void *args) {
      const char *name = kExtroverts[*(size_t *)args];
      printf("Hey, I'm %s.  Empowered to meet you.\n", name);
      return NULL;
    }

    int main() {
      printf("Let's hear from %zu extroverts.\n", kNumExtroverts);
      pthread_t extroverts[kNumExtroverts];
      for (size_t i = 0; i < kNumExtroverts; i++)
        pthread_create(&extroverts[i], NULL, recharge, &i);  
      for (size_t j = 0; j < kNumExtroverts; j++)
        pthread_join(extroverts[j], NULL);
      printf("Everyone's recharged!\n");
      return 0;
    }
    ```

# 竞争条件（续）
* 以下是一些（明显出错的）示例运行：

    ```sh
    poohbear@myth12:$ ./confused-extroverts 
    Let's hear from 6 extroverts.
    Hey, I'm John Carlo Buenaflor.  Empowered to meet you.
    Hey, I'm Jessica Guo.  Empowered to meet you.
    Hey, I'm Sona Allahverdiyeva.  Empowered to meet you.
    Hey, I'm Sona Allahverdiyeva.  Empowered to meet you.
    Hey, I'm Yun Zhang.  Empowered to meet you.
    Hey, I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
    Everyone's recharged!
    poohbear@myth12:$ ./confused-extroverts 
    Let's hear from 6 extroverts.
    Hey, I'm Sona Allahverdiyeva.  Empowered to meet you.
    Hey, I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
    Hey, I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
    Hey, I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
    Hey, I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
    Hey, I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
    Everyone's recharged!
    poohbear@myth12:$
    ```

# 竞争条件（续）
* 显然有哪里出错了，但问题是什么？
    * 注意这次 `recharge` 引用了其传入参数，并且
      `pthread_create` 通过其第四个参数接受外部循环
      索引变量的地址。`pthread_create` 的第四个
      参数始终被原封不动地传递给线程例程作为其唯一参数。
    * 问题：主线程推进 `i`，而没有考虑到
      `i` 的地址已经被共享给了六个子线程中的每一个。
    * 乍一看，很容易假设 `pthread_create` 捕获的
      不仅是 `i` 的地址，还包括 `i` 的值本身。
      这个假设是不正确的，它只复制了地址，仅此而已。
    * `i` 的地址（即使它超出了作用域）是常量，但
      其内容与六个 `recharge` 线程的执行并行演变。
      `*(size_t *)args` 取的是 `i` 在被求值时恰好是任何值的快照。
    * 这就是所谓的*竞争条件（race condition）*的一个简单例子。

::: tip 重难点解析
**竞争条件的本质**：这个例子非常经典，值得仔细分析。

问题出在 `pthread_create(&extroverts[i], NULL, recharge, &i)`。这里传递的是循环变量 `i` 的**地址**（`&i`），而不是 `i` 的**值**。关键时间线：

1. 主线程在循环中快速创建 6 个线程，每次将 `&i` 传给 `pthread_create`
2. 主线程递增 `i`（0→1→2→3→4→5→6），循环结束
3. 子线程**在之后的某个时刻**才被调度执行，它们各自执行 `*(size_t *)args` 来读取 `i` 的当前值
4. 此时 `i` 可能已经是 6（循环结束后的值），也可能在读取瞬间恰好是某个中间值

这就解释了为什么输出中出现了重复的名字（如 "Sona Allahverdiyeva" 出现两次，或 "Jerry Cain" 出现五次）——多个线程读取到了相同的或超出范围的 `i` 值。

**核心教训**：
- 不要在线程间共享可变数据的地址，除非你确切知道自己在做什么
- `pthread_create` 传递的是指针，而非值的快照
- 线程的调度时间是不可预测的，这就是"竞争"的含义——主线程修改 `i` 和子线程读取 `i` 之间存在竞速
- 这个 bug 的棘手之处在于：它有时能"正常工作"（如果子线程在 `i` 改变之前就读到了正确的值），这种间歇性 bug 最难调试
:::

::: tip 重难点解析
**竞争条件的汇编级分析 — 为什么 `count++` 不是原子的**

即使是最简单的 `count++` 操作，在汇编级别也是多条指令。以 x86-64 为例：

```c
// C 代码
static int counter = 0;
counter++;  // 看似"一行代码"，实际上是三条指令
```

编译后的汇编（x86-64）：
```asm
mov    eax, DWORD PTR [counter]   ; 步骤 1: 从内存加载 counter 到寄存器 eax
add    eax, 1                     ; 步骤 2: 寄存器值 +1
mov    DWORD PTR [counter], eax   ; 步骤 3: 将寄存器值写回内存
```

**两个线程同时执行 `counter++` 的灾难性交织**：

假设 counter 初始值为 0，线程 A 和线程 B 各执行一次 `counter++`，期望结果为 2。

```
时间 →
线程A: mov eax, [counter]     ---- eax_A = 0
线程B:           mov eax, [counter]  ---- eax_B = 0  (读到的也是 0！)
线程A: add eax, 1              ---- eax_A = 1
线程B:           add eax, 1           ---- eax_B = 1
线程A: mov [counter], eax      ---- counter = 1
线程B:           mov [counter], eax   ---- counter = 1  (覆盖！期望是 2)
```

两个线程各自完成了 `counter++`，但最终 counter = 1 而非 2。经典**丢失更新（Lost Update）**问题。

**更糟糕的情况**（假设 counter 是 64 位在 32 位平台上）：

```asm
; 32 位平台上的 64 位 counter++ 甚至涉及两次内存读写！
mov    eax, [counter]      ; 加载低 32 位
mov    edx, [counter+4]    ; 加载高 32 位
add    eax, 1
adc    edx, 0              ; 处理进位
mov    [counter], eax      ; 写回低 32 位
mov    [counter+4], edx    ; 写回高 32 位
```

两个线程可能在两次写入之间交错，导致 counter 的高 32 位和低 32 位来自不同操作——产生完全错误的值（"撕裂读/写" tear）。
:::

::: tip 重难点解析
**CPU 缓存层次与内存可见性 — 为什么 `volatile` 不够**

当多线程共享变量时，不仅需要考虑指令交织，还需要考虑 CPU 缓存一致性。

**CPU 缓存层次**：

```
CPU 核心 0                         CPU 核心 1
┌──────────────┐                 ┌──────────────┐
│  L1 Cache    │  私有 (32KB)    │  L1 Cache    │
│  L2 Cache    │  私有 (256KB)   │  L2 Cache    │
└──────┬───────┘                 └──────┬───────┘
       └──────────┬────────────────────┘
                  │
          ┌───────┴────────┐
          │  L3 Cache (共享) │  (8-32MB, 所有核心共享)
          └───────┬────────┘
                  │
          ┌───────┴────────┐
          │    主存 (RAM)    │
          └────────────────┘
```

当线程 A 在核心 0 上修改 `counter`，新值可能仅在核心 0 的 L1 缓存中，而核心 1 的 L1 缓存中仍是旧值。这就是**缓存一致性问题**。

**MESI 协议简介**：

现代 CPU 通过 MESI（Modified, Exclusive, Shared, Invalid）缓存一致性协议解决该问题：
- **Modified**：该缓存行仅存于此核心，已被修改，与主存不一致
- **Exclusive**：该缓存行仅存于此核心，与主存一致
- **Shared**：该缓存行可能存在于多个核心，与主存一致
- **Invalid**：该缓存行无效，读取时需从其他核心或主存获取

当一个核心写入 Shared 状态的缓存行时，协议会先使其他核心的副本 **Invalid**（发送 RFO 消息），然后执行写入。

**为什么 `volatile` 不够**：

`volatile` 只保证：
- 不优化掉看起来"多余"的读写（避免寄存器缓存变量）
- 不重排 volatile 变量之间的访问顺序

`volatile` **不保证**：
- 原子性：`counter++` 仍然是三条指令
- 互斥：多个线程仍可同时访问

```c
volatile int counter = 0;  // volatile 不能修复竞争条件！
// counter++ 仍然可以产生丢失更新
// 正确做法：使用 mutex 或 atomic 类型
```

C++ 的 `std::atomic<T>` 才是正确的线程间共享变量方案——它同时保证了原子性和内存顺序。
:::

# 防止竞争条件
* 幸运的是，这里的修复很直接。
    * 只需传入 `const char *` 即可。
    * 下面内容的在线版本在
      [此处](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-c/extroverts.c)。

    ```c
    static const char *kExtroverts[] = {
      "Albert Chon",
      "John Carlo Buenaflor",
      "Jessica Guo",
      "Lucas Ege",
      "Sona Allahverdiyeva",
      "Yun Zhang",
      "Tagalong Introvert Jerry Cain"
    };
    static const size_t kNumExtroverts = sizeof(kExtroverts)/sizeof(kExtroverts[0]) - 1;

    static void *recharge(void *args) {
      const char *name = args;
      printf("Hey, I'm %s.  Empowered to meet you.\n", name);
      return NULL;
    }

    int main() {
      printf("Let's hear from %zu extroverts.\n", kNumExtroverts);
      pthread_t extroverts[kNumExtroverts];
      for (size_t i = 0; i < kNumExtroverts; i++)
        pthread_create(&extroverts[i], NULL, recharge, (void *) kExtroverts[i]);
      for (size_t i = 0; i < kNumExtroverts; i++)
        pthread_join(extroverts[i], NULL);
      printf("Everyone's recharged!\n");
      return 0;
    }
    ```

# 防止竞争条件（续）
* 危机解除！
    * 每次安装 `recharge` 时共享的是不同的地址。
      相关 C 字符串的快照在线程被创建之前（而非在线程执行期间）就已经获取。
    * 竞争条件通常非常复杂，避免它们并不总是如此简单。
    * 以下是一些测试运行，只是为了让你看到它已经修复（并且输出在每次运行时有所不同）。

    ```sh
    poohbear@myth12:$ ./extroverts 
    Let's hear from 6 extroverts.
    Hey, I'm Albert Chon.  Empowered to meet you.
    Hey, I'm Jessica Guo.  Empowered to meet you.
    Hey, I'm Sona Allahverdiyeva.  Empowered to meet you.
    Hey, I'm Yun Zhang.  Empowered to meet you.
    Hey, I'm Lucas Ege.  Empowered to meet you.
    Hey, I'm John Carlo Buenaflor.  Empowered to meet you.
    Everyone's recharged!
    poohbear@myth12:$ ./extroverts 
    Let's hear from 6 extroverts.
    Hey, I'm Albert Chon.  Empowered to meet you.
    Hey, I'm John Carlo Buenaflor.  Empowered to meet you.
    Hey, I'm Jessica Guo.  Empowered to meet you.
    Hey, I'm Lucas Ege.  Empowered to meet you.
    Hey, I'm Sona Allahverdiyeva.  Empowered to meet you.
    Hey, I'm Yun Zhang.  Empowered to meet you.
    Everyone's recharged!
    poohbear@myth12:$
    ```

::: tip 重难点解析
**修复原理**：修复的关键区别在于：

- **有 bug 的版本**：`pthread_create(&extroverts[i], NULL, recharge, &i)` — 传递 `i` 的地址，所有线程共享同一个地址，内容随时间变化。
- **修复后的版本**：`pthread_create(&extroverts[i], NULL, recharge, (void *) kExtroverts[i])` — 每次传递的是数组中第 `i` 个字符串常量的地址。这个字符串常量在程序的整个生命周期内不变，存储在不同的内存位置。每个线程获得一个不同的、稳定的指针。

**为什么输出顺序仍然不确定？**：注意修复后的两次运行输出顺序不同（第一次 "Albert Chon" 开头，第二次 "Albert Chon" 也开头但后续顺序不同）。这体现了线程调度的非确定性——即使数据是正确的，线程执行的**相对顺序**仍然不确定。这就是为什么在后续课程中我们需要引入互斥锁（mutex）和条件变量（condition variable）来协调线程的执行顺序。

**CS111 衔接**：竞争条件是并发编程中最根本的问题之一。在 CS111 中，你将学习更复杂的同步原语（互斥锁、信号量、条件变量、读写锁）以及死锁（deadlock）的检测和预防。CS110 为你打下基础，CS111 将这些问题引向更深的层次。
:::

::: tip 重难点解析
**`pthread_mutex_lock` 的内核实现 — futex（Fast Userspace Mutex）机制**

互斥锁（mutex）的实现基于 Linux 的 futex（Fast Userspace muTEX）机制，这是一种巧妙的设计，使**无竞争情况下的加锁/解锁完全在用户空间完成，无需系统调用**。

**futex 的核心思想**：

锁的状态存储在一个用户空间的 32 位整数中（`atomic int`），所有线程共享该变量。`pthread_mutex_t` 内部包含这个整数。

**无竞争路径（Fast Path）**：

```c
// pthread_mutex_lock 的无竞争路径（简化伪代码）
void pthread_mutex_lock(pthread_mutex_t *mutex) {
    // 尝试用原子 compare-and-swap 从 0（未锁定）改为 1（已锁定）
    int expected = 0;
    if (atomic_compare_exchange_strong(&mutex->__lock, &expected, 1)) {
        // 成功！锁获取成功，零系统调用
        return;
    }
    // 失败了，说明有竞争，走慢路径
    lock_slow_path(mutex);
}
```

`atomic_compare_exchange_strong` 是一条硬件原子指令（x86: `LOCK CMPXCHG`），在现代 CPU 上仅需几十个周期。如果锁没有被其他线程持有，整个过程**不需要进入内核**——这就是 futex 中 "Fast" 的含义。

**有竞争路径（Slow Path）**：

```c
// 慢路径：需要系统调用
void lock_slow_path(pthread_mutex_t *mutex) {
    while (true) {
        int old = atomic_exchange(&mutex->__lock, 2);
        if (old == 0) return;  // 恰好这瞬间锁空闲了
        // 锁被别人持有，通过 futex 等待
        futex_wait(&mutex->__lock, 2, NULL);  // futex(FUTEX_WAIT)
    }
}

// pthread_mutex_unlock 的简化逻辑
void pthread_mutex_unlock(pthread_mutex_t *mutex) {
    int old = atomic_fetch_sub(&mutex->__lock, 1);
    if (old != 1) {
        // 有等待者，需要唤醒
        mutex->__lock = 0;
        futex_wake(&mutex->__lock, 1);  // futex(FUTEX_WAKE)
    }
}
```

**futex 的优势**：

| 竞争情况 | 行为 | 系统调用开销 |
|----------|------|-------------|
| 无竞争 | 用户空间 CAS 操作 | 0 次系统调用 |
| 轻度竞争 | 短暂自旋 + futex_wait | 1 次 futex(FUTEX_WAIT) |
| 重度竞争 | 直接 futex_wait | 1 次 futex(FUTEX_WAIT) |

关键是：**日常使用中大多数锁都是无竞争的**（线程访问锁的时间窗口很少重叠）。futex 使这一常见情况的性能极致优化——无需上下文切换到内核态。这就是为什么 pthread mutex 比 System V 信号量（semaphore）等传统同步机制快得多的原因。

**futex 与进程间同步**：

futex 通常用于线程间同步（共享地址空间），但通过 `FUTEX_PRIVATE_FLAG` 和共享内存映射，也可以用于进程间同步。`PTHREAD_PROCESS_SHARED` 属性的互斥锁就是基于此实现。
:::
