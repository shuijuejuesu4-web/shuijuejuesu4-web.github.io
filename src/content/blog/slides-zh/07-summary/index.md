---
title: "多进程进阶：进程控制与作业调度"
description: "多进程进阶：进程控制与作业调度"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告
* Assignment 2 和 Assignment 3
    * Assignment 2 今晚 11:59 截止。
        * 我们预计从周六早上开始批改 Assignment 2 的提交（在硬截止时间过后），
          并在下周六之前发出成绩报告。
        * 一般来说，我们承诺在硬截止时间后 7 天内返回成绩报告。
    * Assignment 3 将于今晚稍后发布，截止日期为下周一的 10 月 23 日。
        * Assignment 3 是本学期最长的作业，这就是为什么你有这么长的时间来完成它。
          请尽早开始。

* 阅读材料：
    * 完成 B&O 第 2 章（完整教材的第 10 章）的阅读，
      这样你可以确认自己已经掌握了其中的大部分内容，
      因为这些内容我在前两周的课程中已经涵盖了很大一部分。
    * 完成 B&O 第 1 章（完整教材的第 8 章）的阅读，
      重点关注第 5 节，该节涵盖了进程组（process groups）、信号（signals）
      和信号处理器（signal handlers），这三者都将对你的
      Assignment 3 提交有所帮助。

* 议程？继续搭乘这趟多进程列车！
    * 上次我刚刚准备开始讲
      [这个](http://web.stanford.edu/class/cs110/autumn-2017/lectures/06-execvp-subprocess.html#(1))，
      但还没讲到。我们对 `subprocess` 函数的讨论依赖于
      管道（pipe）的概念、`pipe` 和 `dup2`
      系统调用，以及如何利用它们在多个进程之间实现更复杂的通信。
    * 一旦我们完成了 `subprocess` 的内容，我们将开始讨论信号和信号处理器。

# 信号简介
* 信号（Signals）
    * 信号是一条小消息，用于通知进程发生了某种类型的事件。
      信号通常由内核发送，但也可以从其他进程发送。
    * 信号处理器（signal handler）是一个函数，它在接收到
      并处理某个信号时执行。
    * 即使你以前没有用过这个名字，你已经对某些信号很熟悉了：
        * 如果你从未在 C 编程中解引用过 `NULL`
          指针，那你就不算真正用 C 编程过。当这种情况发生时，内核会发送一个
          `SIGSEGV` 类型的信号，通俗地称为段错误（segmentation fault，或 SEGmentation
          Violation，简称为 `SIGSEGV`）。除非你安装了一个自定义的信号处理器
          以不同方式处理该信号，否则 `SIGSEGV` 会终止
          程序并生成一个核心转储（core dump）。
        * 每当进程发生整数除零（在某些情况下也包括
          浮点除零）时，内核会发出并发送一个
          `SIGFPE` 信号给违规进程。（默认情况下，
          程序会终止，显示 `SIGFPE` 消息并生成核心转储）。
        * 当你输入 ctrl-c 时，内核会向前台进程发送一个 `SIGINT`
          信号（默认情况下，程序被终止）。
        * 当你输入 ctrl-z 时，内核会向前台进程发送一个 `SIGTSTP`
          信号（默认情况下，进程被挂起，直到后续的 `SIGCONT` 信号指示
          它恢复执行）。
        * 每当一个子进程退出（无论是正常退出还是异常退出），内核
          可能会向父进程发送一个 `SIGCHLD` 信号。
          默认情况下，该信号被忽略（事实上，默认情况下内核甚至
          不会发送它，除非父进程注册了对接收这些信号感兴趣的意图）。
          这种特殊类型的信号对于允许 fork 出的子进程在后台运行
          而父进程继续做自己的工作而不在 `waitpid` 调用上阻塞至关重要。然而，
          父进程仍然有义务回收子进程僵尸进程（zombie），因此
          父进程通常会注册代码，在子进程
          终止时被调用。这样做会促使内核开始发送
          `SIGCHLD` 信号，以便注册的 `SIGCHLD` 处理器
          可以通过 `waitpid` 回收僵尸进程。

::: tip 重难点解析
**信号与中断的类比**：信号可以理解为操作系统向进程发送的"软件中断"。就像硬件中断会打断 CPU 当前执行的指令流一样，信号的到达会暂停进程的正常执行，转去执行信号处理器函数。关键区别在于：信号由内核或其它进程触发，而非硬件。在 CS111（操作系统原理）中，你还会学习到信号在线程环境下的复杂行为——因为每个线程可以拥有独立的信号掩码（signal mask）。

**僵尸进程（Zombie Process）**：当子进程先于父进程退出时，它会进入"僵尸"状态——进程已经终止，但其退出状态和进程表项仍然保留在内核中，等待父进程通过 `waitpid()` 来回收。如果父进程不回收，僵尸进程会一直占用系统资源。这就是为什么 `SIGCHLD` 信号处理器如此重要——即使父进程忙于其他工作，也能在子进程退出时被异步通知、及时回收。
:::

::: tip 重难点解析
**信号在内核中的实现机制 — 从产生到递送的全过程**

理解信号的内核实现是掌握信号编程的关键。整个流程涉及三个核心数据结构：

**1. 待处理信号位图（Pending Signal Bitmap）**

每个进程的 `task_struct` 中维护一个 `pending` 位图（`sigset_t` 类型），每个位对应一种信号。当信号产生时（如子进程退出产生 SIGCHLD），内核执行：
```c
// 内核级伪代码
task_struct->pending.signal |= (1UL << (sig - 1));  // 设置对应的 pending bit
```
注意：pending 是**位图**而非**计数器**。这就解释了之前五胞胎示例中"信号合并"的现象——多个同类型信号到达时，pending bit 只能记录"有"还是"没有"，无法区分到底来了 1 个还是 5 个。

**2. 阻塞信号掩码（Blocked Signal Mask）**

每个进程还有一个 `blocked` 掩码（同样是 `sigset_t`），记录哪些信号当前被阻塞。`sigprocmask(SIG_BLOCK, &set, NULL)` 就是在修改这个掩码。

**3. 信号递送时机 — 内核态返回用户态**

信号的递送（delivery）——即触发信号处理器——发生在**内核态返回用户态**的时刻。具体地，在每次系统调用返回、中断返回、或上下文切换恢复进程执行时，内核检查：
```c
// 内核级伪代码
sigset_t ready = task->pending & ~task->blocked;  // 待处理且未被阻塞的信号
if (ready) {
    int sig = ffs(ready) - 1;  // 找最低位的有效信号
    handle_signal(sig, task);  // 递送信号
}
```
关键点：**信号处理器不在信号产生的时刻立即执行**，而是在进程下次从内核态返回用户态时被"递送"。如果进程正在内核中执行系统调用，信号会被暂时挂起。

**`sa_mask` 的作用**：信号处理器执行期间，引发该处理器的信号会自动加入 blocked 掩码（由内核自动完成），防止同一信号的嵌套递送。此外，`sigaction` 允许通过 `sa_mask` 字段指定额外的信号在处理器执行期间被阻塞。这是通过**临时修改** blocked 掩码实现的——处理器退出前内核恢复原掩码。
:::

# 第一个信号处理器示例
* 源代码可以安装一个信号处理器来*捕获*并以不同于默认方式的方式处理某种
  类型的信号。
    * 以下是一个精心编写的示例，说明如何实现和
      安装一个 `SIGCHLD` 处理器（分两张幻灯片展示）。前提是
      爸爸带他的五个孩子出去玩。五个孩子各自
      玩耍的时间不同。当所有五个孩子都
      玩累了，六个人（五个孩子和爸爸）一起回家。
    * 请理解，父进程模拟的是爸爸，子进程
      模拟的是他的孩子。（代码可以在
      [此处](http://cs110.stanford.edu/autumn-2017/examples/processes/five-children.c)找到。）

    ```c
    static const size_t kNumChildren = 5;
    static size_t numChildrenDonePlaying = 0;

    static void reapChild(int sig) {
      exitIf(waitpid(-1, NULL, 0) == -1, kWaitFailed,
             stderr, "waitpid failed within reapChild sighandler.\n");
      numChildrenDonePlaying++;
      sleep(1); // represents time spent doing other useful work
    }

    int main(int argc, char *argv[]) {
      printf("Let my five children play while I take a nap.\n");
      exitIf(signal(SIGCHLD, reapChild) == SIG_ERR, kSignalFailed,
             stderr, "Failed to install SIGCHLD handler.\n");
      for (size_t kid = 1; kid <= 5; kid++) {
        pid_t pid = fork();
        exitIf(pid == -1, kForkFailed, stderr, "Child #%zu doesn't want to play.\n", kid);
        if (pid == 0) {
          sleep(3 * kid); // sleep emulates "play" time
          printf("Child #%zu tired... returns to dad.\n", kid);
          return 0;
        }
      }
    ```

# 第一个信号处理器示例（续）
* 以下是上一张幻灯片中程序的下半部分：
    * 注意使用了 `snooze` 函数，这是我
      自己实现的一个不可中断的 `sleep`：

    ```c
      while (numChildrenDonePlaying < kNumChildren) {
        printf("At least one child still playing, so dad nods off.\n");
        snooze(5); // implementation in /usr/class/cs110/local/include/sleep-utils.h
        printf("Dad wakes up! ");
      }

      printf("All children accounted for.  Good job, dad!\n");
      return 0;
    }
    ```

    * 上述代码经过精心设计，使每个子进程
      大约间隔 3 秒退出。`reapChild` 当然会
      捕获并处理内核在每次 fork 出的子进程退出时发送的
      每个 `SIGCHLD` 信号。`reapChild`
      被设计为大约需要一秒钟才能执行完成，所以每次
      `reapChild` 在下个子进程退出前大约两秒完成。
    * `signal` 函数不允许与信号处理器共享状态，
      因此我们别无选择，只能使用全局变量。

::: tip 重难点解析
**`signal` 函数的限制**：`signal()` 是最简单的信号处理器注册方式，但它有重要局限：(1) 信号处理器只能访问全局变量，不能通过参数传递状态；(2) 不同 UNIX 系统的 `signal()` 行为有细微差异（如 BSD vs System V 的"一次性"还是"持久"语义）。现代编程中更推荐使用 `sigaction()` 来替代 `signal()`，它提供了更精确的控制和可移植性。

**`snooze` 与 `sleep` 的区别**：标准的 `sleep()` 函数在收到信号后会被打断并提前返回，而 `snooze()` 是老师的自定义实现，确保即使收到信号也能睡满指定时间。这种区别在多进程同步场景下至关重要——如果用 `sleep()`，信号处理器执行完后父进程会被唤醒，程序逻辑可能被意外打乱。
:::

::: tip 重难点解析
**`sigaction` vs `signal` — 现代信号处理器注册方式**

`signal()` 是历史遗留的简化 API，存在两大问题：(1) 不同 UNIX 变体行为不一致（BSD 的 `signal()` 是持久的，System V 是一次性的，`SIG_IGN` 对 `SIGCHLD` 的行为也不同）；(2) 无法精细控制信号处理行为。`sigaction` 通过结构体提供了精确控制：

```c
struct sigaction {
    void     (*sa_handler)(int);      // 信号处理器函数（SIG_DFL, SIG_IGN, 或自定义函数）
    void     (*sa_sigaction)(int, siginfo_t *, void *);  // 替代处理器（使用 SA_SIGINFO 时）
    sigset_t   sa_mask;               // 处理器执行期间额外阻塞的信号集合
    int        sa_flags;              // 标志位控制行为
    void     (*sa_restorer)(void);    // 已废弃，不使用
};
```

**关键标志位**：

- **`SA_RESTART`**：自动重启被信号中断的慢速系统调用（如 `read`、`write`、`waitpid`）。如果不设置此标志，信号处理器返回后这些系统调用会返回 -1 并设置 `errno = EINTR`，需要手动处理。设置 `SA_RESTART` 后内核会自动重启它们，简化编程。

```c
// 禁止 SA_RESTART：被信号中断后必须手动重启
struct sigaction sa;
sa.sa_handler = my_handler;
sa.sa_flags = 0;  // 不设置 SA_RESTART
sigaction(SIGINT, &sa, NULL);
// 如果 read() 在等待键盘输入时被 SIGINT 中断：
// read() 返回 -1, errno = EINTR，你需要自己再次调用 read()
```

- **`SA_NOCLDSTOP`**：仅对 `SIGCHLD` 有效。设置后，当子进程被暂停（stopped via SIGTSTP）时，父进程**不会**收到 SIGCHLD 通知。只有子进程真正终止时才通知。在 shell 的作业控制实现中极其重要——你通常只关心子进程是否终止，不关心它是否被 Ctrl+Z 暂停。

- **`SA_SIGINFO`**：使用 `sa_sigaction` 而非 `sa_handler` 作为处理器。`sa_sigaction` 接收额外的 `siginfo_t` 参数，提供关于信号的丰富信息（发送者 pid、uid、导致故障的内存地址等）。

```c
void handler(int sig, siginfo_t *info, void *context) {
    printf("Signal %d from pid %d (uid %d)\n",
           sig, info->si_pid, info->si_uid);
    // info->si_addr 对于 SIGSEGV 包含导致段错误的地址
}
```
:::

::: tip 重难点解析
**信号处理器调用的底层机制 — 内核如何"劫持"用户栈**

这是一个操作系统实现细节，理解它能帮你避免信号编程中的许多陷阱。

当内核决定向进程递送信号时，它并不简单地在当前指令位置调用信号处理器——那会破坏用户程序的正常执行流程。相反，内核执行以下步骤：

1. **保存上下文**：内核在用户栈上创建一个 `sigframe` 结构体，保存当前进程的寄存器状态（包括被中断时的程序计数器 PC、栈指针 SP、通用寄存器等）

2. **设置返回地址**：内核在用户栈上压入一个特殊的返回地址——指向 `sigreturn` 系统调用的代码片段（通常位于 vdso 或 vsyscall 页）

3. **修改用户栈**：内核修改用户栈指针和指令指针，使得：
   - 栈指针指向准备好的栈帧（含返回地址和 sigframe）
   - 指令指针指向信号处理器的第一条指令

4. **返回用户态**：内核执行 `iret`（x86）或 `eret`（ARM）指令，从内核态返回用户态，但从信号处理器开始执行而非原来的程序

5. **处理器执行完毕后**：当处理器执行 `return` 或函数结束时，返回地址将控制流引向 `sigreturn`。`sigreturn` 系统调用让内核从之前保存的 sigframe 中恢复所有寄存器，让原程序从被中断的位置无缝继续执行。

```
原程序:  ...  instr_N  ...
                      ↑
         信号到达 → 内核保存状态 → 执行处理器
                                    ↓
                                 处理器 return
                                    ↓
                                 sigreturn → 恢复状态
                                    ↓
原程序:  ...  instr_N+1 ...  (无缝继续)
```

这就是为什么信号处理器可以被视为一种"软件中断"——它与硬件中断一样修改了执行流，但整个过程由内核协调，通过操作用户栈来实现。
:::

    * 以下是上述程序的可预测输出。

    ```sh
    myth22> ./five-children 
    Let my five children play while I take a nap.
    At least one child still playing, so dad nods off.
    Child #1 tired... returns to dad.
    Child #2 tired... returns to dad.
    Dad wakes up! At least one child still playing, so dad nods off.
    Child #3 tired... returns to dad.
    Child #4 tired... returns to dad.
    Dad wakes up! At least one child still playing, so dad nods off.
    Child #5 tired... returns to dad.
    Dad wakes up! All children accounted for.  Good job, dad!
    myth22>
    ```

# 主题变奏：五胞胎
* 考虑下一个程序：
    * 下一个程序与前一个大致相同，只是每个子进程
      需要*相同的时间来完成*（程序分两张幻灯片展示）
    * 此示例的代码可以在
      [此处](http://cs110.stanford.edu/autumn-2017/examples/processes/indistinguishable-pentuplets.c)找到。

    ```c
    static const size_t kNumChildren = 5;
    static size_t numChildrenDonePlaying = 0;

    static void reapChild(int sig) {
      exitIf(waitpid(-1, NULL, 0) == -1, kWaitFailed,
             stderr, "waitpid failed within reapChild sighandler.\n");
      numChildrenDonePlaying++;
      sleep(1); // represents time that useful work might be done
    }

    int main(int argc, char *argv[]) {
      printf("Let my five children play while I take a nap.\n");
      exitIf(signal(SIGCHLD, reapChild) == SIG_ERR, kSignalFailed,
             stderr, "Failed to install SIGCHLD handler.\n");
      for (size_t kid = 1; kid <= 5; kid++) {
        pid_t pid = fork();
        exitIf(pid == -1, kForkFailed, stderr, "Child #%zu doesn't want to play.\n", kid);
        if (pid == 0) {
          sleep(3); // all kids play together for three seconds
          printf("Kid #%zu done playing... runs back to dad.\n", kid);
          return 0;
        }
      }
    ```

# 主题变奏：五胞胎（续）
* 以下是程序的第二部分：
    * （这部分与之前完全相同）。

    ```c
      while (numChildrenDonePlaying < kNumChildren) {
        printf("At least one child still playing, so dad nods off.\n");
        snooze(5);
        printf("Dad wakes up! ");
      }

      printf("All children accounted for.  Good job, dad!\n");
      return 0;
    }
    ```
    * 这里的主要区别在于所有子进程几乎
      同时退出。虽然内核确实会发出五个 `SIGCHLD` 信号，
      但并非所有这些信号都会触发一次专门的 `reapChild` 执行。
    * 不相信我？看看这个第二个版本的可复现测试运行
      （其中所有五个孩子几乎同时完成玩耍）。

    ```sh
    myth22> ./indistinguishable-pentuplets
    Let my five children play while I take a nap.
    At least one child still playing, so dad nods off.
    Kid #1 done playing... runs back to dad.
    Kid #3 done playing... runs back to dad.
    Kid #4 done playing... runs back to dad.
    Kid #2 done playing... runs back to dad.
    Kid #5 done playing... runs back to dad.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    Dad wakes up! At least one child still playing, so dad nods off.
    <ctrl-c>
    myth22>
    ```

# 主题变奏（续）
* 以下是发生的事情：
    * 其中一个子进程在其他四个之前完成，内核
      代表它向父进程发送了一个 `SIGCHLD` 信号。
    * 该 `SIGCHLD` 信号被捕获，`reapChild`
      执行（大约需要一秒钟）来处理它。
    * 在这一秒钟内，五个子进程中的第二个退出，相应的
      `SIGCHLD` 信号被记录但被阻塞，
      直到第一次 `reapChild` 调用退出。第三、第四和
      第五个子进程在第一次 `reapChild` 仍在运行时全部退出，但它们
      的 `SIGCHLD` 信号被丢弃了，因为内核维护的不是
      待处理 `SIGCHLD` 信号的**计数**，而是一个单独的位（bit），
      用来记录**是否有一个或多个**信号已经到达。
    * 当第一次 `reapChild` 调用退出时，待处理
      `SIGCHLD` 信号的阻塞被解除。内核很快检测到
      高位的 `SIGCHLD` 位，但不知道实际上有多少个 `SIGCHLD` 信号
      被触发了。`reapChild` 被调用来处理
      所有未处理的 `SIGCHLD` 信号，因此 `reapChild`
      只多执行了一次。
    * 总结：`numChildrenDonePlaying` 全局变量只
      被递增了两次，而父进程（模拟爸爸的进程）
      反复地每次小睡五秒钟，直到永远。

::: tip 重难点解析
**信号的不可靠性与信号合并（Signal Coalescing）**：这是本讲最核心的概念之一。在 UNIX 的经典信号模型中，待处理信号不是按"计数"而是按"位"（pending bit）来记录的——对于每种信号类型，内核只保留一个比特位来表示"是否有该类型的信号待处理"。因此，如果同一种信号在短时间内多次到达（如上面的例子中 5 个子进程几乎同时退出），后续到达的信号可能会被"合并"丢失。这意味着信号处理器被调用的次数可能少于信号实际发生的次数。这就是为什么信号处理被认为是"不可靠"的，也是为什么在下一张幻灯片中，`reapChild` 必须使用 `WNOHANG` 循环来确保回收所有僵尸子进程。在 CS111 中你会学到，实时信号（real-time signals，`SIGRTMIN` 到 `SIGRTMAX`）支持排队（queueing），从而解决了信号合并问题。
:::

::: tip 重难点解析
**可重入性（Reentrancy）与异步信号安全（Async-Signal-Safety）**

信号处理器可以在程序执行的**任意时刻**被调用——包括程序正在执行 `malloc`、`printf` 或其他库函数的过程中。如果信号处理器也调用这些函数，就会发生**重入（reentry）**：同一个函数在执行完之前被再次调用，导致内部数据结构损坏。

**可重入（Reentrant）函数的特征**：
- 不修改全局状态（或只修改，但使用原子操作）
- 不调用不可重入的函数
- 不持有锁（或者说，如果持有锁，另外的调用会死锁）

**POSIX 异步信号安全函数列表（最常用的）**：

| 函数 | 说明 |
|------|------|
| `write()` | 写入文件描述符（信号处理器中安全的输出方式） |
| `read()` | 从文件描述符读取 |
| `waitpid()` | 等待/回收子进程 |
| `_exit()` / `_Exit()` | 立即终止进程（不是 `exit()`，后者会刷新缓冲区） |
| `kill()` | 发送信号 |
| `signal()` / `sigaction()` | 安装信号处理器 |
| `sigprocmask()` | 修改信号掩码 |
| `getpid()` / `getppid()` | 获取进程 ID |
| `close()` | 关闭文件描述符 |
| `dup2()` | 复制文件描述符 |
| `stat()`, `open()`, `fcntl()` | 文件操作（部分安全） |

**绝对不要在信号处理器中使用的函数**：`printf`、`fprintf`、`malloc`、`free`、`exit`（非 `_exit`）、任何可能分配内存或持有锁的 C++ 标准库函数。

```c
// 错误示例：信号处理器中使用 printf
void bad_handler(int sig) {
    printf("Received signal %d\n", sig);  // 不安全！可能死锁
}

// 正确示例：使用 write 替代
void good_handler(int sig) {
    char msg[64];
    int len = snprintf(msg, sizeof(msg), "Received signal %d\n", sig);
    write(STDOUT_FILENO, msg, len);  // write 是异步信号安全的
}
```

**为什么 `printf` 不是异步信号安全的？** `printf` 内部使用 `malloc` 分配缓冲区，而 `malloc` 维护一个全局空闲链表。如果主程序正在 `malloc` 中操作该链表（且持有内部锁），信号处理器又调用 `printf` → `malloc`，就会导致死锁或数据损坏。这就是为什么在信号处理器中，只能用 `write` 直接输出，或用 `_exit` 立即退出。
:::

# 解决方案...
* ...很简单，前提是你理解了待处理信号是按布尔值（bool）而非计数（count）来记录的。
    * `reapChild` 必须被实现为考虑到
      许多 `SIGCHLD` 信号被触发的可能性，而不仅仅是
      一个（例如，在之前的 `reapChild`
      调用期间，也就是在 `SIGCHLD` 信号被接收但被阻塞的窗口期内，
      有许多子进程完成了）。
    * 每次调用 `waitpid` 应该包含第三个参数 `WNOHANG`，这是一个标志，
      指示 waitpid 不要阻塞在尚未退出的子进程上。当使用
      `WNOHANG` 时，我们需要在两种不同的返回值上中断：
        * 0，表示还有其他子进程，但它们都还没有
          转变为僵尸状态，以及
        * -1，表示其之前的含义：没有任何子进程（这
          由 `errno` 被设置为
          `ECHILD` 来确认）。

::: tip 重难点解析
**`WNOHANG` 标志的作用**：`waitpid(pid, status, WNOHANG)` 的本质是"非阻塞等待"——如果有子进程已经退出，立即回收并返回其 pid；如果没有已退出的子进程，立即返回 0 而不是阻塞等待。这允许信号处理器用 `while (true)` 循环反复调用 `waitpid(-1, NULL, WNOHANG)` 来一次性回收所有已退出的子进程，无论到底有多少个 `SIGCHLD` 信号被合并丢失了。这是应对信号合并问题的标准范式。
:::

# 解决方案
* 解决方案使用了 `WNOHANG`。
    * 以下是 `reapChild` 的**正确**实现
      （查看[此处](http://cs110.stanford.edu/spring-2017/examples/processes/pentuplets.c)
      获取代码）。

    ```c
    static void reapChild(int sig) {
      pid_t pid;
      while (true) {
        pid = waitpid(-1, NULL, WNOHANG);
        if (pid <= 0) break;
        numChildrenDonePlaying++;
      }
      exitUnless(pid == 0 || errno == ECHILD, kWaitFailed,
                 stderr, "waitpid failed within reapChild sighandler.\n");
      sleep(1); // represents time that useful work might be done     
    }
    ```

    * 以下是修复后程序的输出：

    ```sh
    myth22> ./pentuplets
    Let my five children play while I take a nap.
    At least one child still playing, so dad nods off.
    Kid #1 done playing... runs back to dad.
    Kid #2 done playing... runs back to dad.
    Kid #4 done playing... runs back to dad.
    Kid #3 done playing... runs back to dad.
    Kid #5 done playing... runs back to dad.
    Dad wakes up! All children accounted for.  Good job, dad!
    myth22>    
    ```

# 同步问题简介
* 同步（Synchronization）、多进程（multi-processing）、并行（parallelism）和并发（concurrency）
    * 所有这些都是本课程的核心主题，也是极其强大的特性。
    * 非常难以理解且难以做到正确，各种并发问题
      和竞争条件（race conditions）可能会出现，除非你非常小心地编写代码。
    * 考虑以下程序，它向你将在 Assignment 4 中完成的 shell
      的最终形态致敬（整个程序的代码可以在
      [此处](http://cs110.stanford.edu/autumn-2017/examples/processes/job-list-synchronization.c)找到）：

    ```c
    static void reapChild(int sig) {
      pid_t pid;
      while (true) {
        pid = waitpid(-1, NULL, WNOHANG);
        if (pid <= 0) break;
        printf("Job %d removed from job list.\n", pid);
      }
      exitUnless(pid == 0 || errno == ECHILD, kWaitFailed,
                 stderr, "waitpid failed within reapChild sighandler.\n");
    }

    int main(int argc, char *argv[]) {
      exitIf(signal(SIGCHLD, reapChild) == SIG_ERR, kSignalFailed,
             stderr, "signal function failed.\n");
      for (size_t i = 0; i < 3; i++) {
        pid_t pid = fork();
        exitIf(pid == -1, kForkFailed,
               stderr, "fork function failed.\n");
        if (pid == 0) {
          char *listArguments[] = {"date", NULL};
          exitIf(execvp(listArguments[0], listArguments) == -1,
                 kExecFailed, stderr, "execvp function failed.\n");
        }
        snooze(1); // represents meaningful time spent
        printf("Job %d added to job list.\n", pid);
      }

      return 0;
    }

    ```

# 初学者的第一个并发问题！
* 看看上一个程序的测试运行：
    * 这有多错？!!

    ```sh
    myth22> ./job-list-synchronization
    Wed Oct 11 9:45:41 PDT 2017
    Job 26874 removed from job list.
    Job 26874 added to job list.
    Wed Oct 11 9:45:42 PDT 2017
    Job 26875 removed from job list.
    Job 26875 added to job list.
    Wed Oct 11 9:45:43 PDT 2017
    Job 26876 removed from job list.
    Job 26876 added to job list.
    myth22>
    ```

    * 这里最大的问题是，每个子进程（通过 `fork`/`execvp` 对快速
      将日期输出到控制台）在父进程完成其一秒小睡之前就退出，
      并促使内核向父进程发射一个
      `SIGCHLD` 信号。`reapChild` 函数完全执行
      （并将作业从某个作业列表中"移除"），然后父进程才推进到
      将同一作业"添加"到同一作业列表的那一点。
    * 这太糟糕了！欢迎来到在共享数据结构上操作的并发上下文的世界。
    * 解决方案：
        * 通俗地说，我们需要以编程方式阻塞
          `SIGCHLD` 信号的处理，直到父进程成功
          将进程添加到作业列表中。
        * 形式上，我们需要使用 `sigset_t` 掩码来临时阻塞
          `SIGCHLD` 信号的处理，直到父进程
          执行完将作业添加到作业列表的那段代码。

::: tip 重难点解析
**竞争条件（Race Condition）**：这是一个经典的竞争条件案例。子进程快速执行 `date` 命令后退出，内核立即向父进程发送 `SIGCHLD`。由于信号是异步的，它可能在父进程执行到 `printf("Job %d added to job list.\n", pid)` 之前就触发 `reapChild`，导致"先移除后添加"的荒谬输出。这就是为什么在多进程/多线程编程中需要同步机制——当多个执行流并发访问共享数据时，其相对执行顺序是不可预测的，任何依赖于特定顺序的假设都可能导致 bug。

**信号与主程序流的"竞速"**：信号处理器的执行可以被看作是"中断"了主程序的正常流程。在上面的输出中可以清晰看到：`Job 26874 removed from job list.` 出现在 `Job 26874 added to job list.` 之前，说明信号处理器在主程序的 `printf` 之前就执行了。这种"执行顺序的不确定性"是并发编程的核心挑战。
:::

# 初学者的第一个并发解决方案
* 这个并发问题是不可接受的。
    * 我们必须采取步骤，以编程方式暂停
      `SIGCHLD` 信号的处理，直到父进程准备好处理它们。
    * 引入 `sigset_t` 类型以及一组可用于
      临时阻塞一种或多种信号类型接收的函数。
    * 以下是可工作的 `main` 函数（整个程序的代码可以在
      [此处](http://cs110.stanford.edu/autumn-2017/examples/processes/job-list-synchronization-improved.c)找到）：

    ```c
    int main(int argc, char *argv[]) {
      exitIf(signal(SIGCHLD, reapChild) == SIG_ERR, kSignalFailed,
             stderr, "signal function failed.\n");
      sigset_t mask;
      sigemptyset(&mask);
      sigaddset(&mask, SIGCHLD);
      for (size_t i = 0; i < 3; i++) {
        sigprocmask(SIG_BLOCK, &mask, NULL);
        pid_t pid = fork();
        exitIf(pid == -1, kForkFailed,
               stderr, "fork function failed.\n");
        if (pid == 0) {
          sigprocmask(SIG_UNBLOCK, &mask, NULL);
          char *listArguments[] = {"date", NULL};
          exitIf(execvp(listArguments[0], listArguments) == -1,
                 kExecFailed, stderr, "execvp function failed.\n");
        }

        snooze(1);
        printf("Job %d added to job list.\n", pid);
        sigprocmask(SIG_UNBLOCK, &mask, NULL); // begin handling SIGCHLD signals again
      }    
      return 0;
    }
    ```

    * `sigset_t` 变量管理要通过 `sigprocmask`
      调用来阻塞和解除阻塞的信号类型集合。
        * 注意对 `sigprocmask(SIG_BLOCK, &mask, NULL);` 的调用，它通知内核
          调用进程在 fork 出的子进程被添加到作业列表之前不打算处理任何 `SIGCHLD`
          事件。还请注意，对 `sigprocmask(SIG_UNBLOCK, &mask, NULL);` 的调用出现在子进程
          pid 已被添加到作业列表之后（通过 `printf` 来表示）。
        * 事实证明，fork 出的进程继承了阻塞信号向量，因此它也需要
          通过自己的 `sigprocmask(SIG_UNBLOCK, &mask, NULL);` 调用来解除阻塞。
          在这个示例中这不重要，但如果 fork 出的子进程自己
          又 fork 出了自己的子进程，那就会很重要。

::: tip 重难点解析
**sigset_t 与信号掩码（Signal Mask）**：`sigset_t` 是 POSIX 定义的信号集合类型，配合以下函数使用：
- `sigemptyset(&mask)` — 清空集合（初始为空集）
- `sigaddset(&mask, SIGCHLD)` — 将 SIGCHLD 加入集合
- `sigprocmask(SIG_BLOCK, &mask, NULL)` — 阻塞集合中的信号（信号不会被丢弃，而是挂起等待解除阻塞）
- `sigprocmask(SIG_UNBLOCK, &mask, NULL)` — 解除阻塞（此时挂起的信号会被递送）

关键点：**被阻塞的信号不会丢失**（这点与之前讲到的信号合并不同）。阻塞只是延迟递送，一旦解除阻塞，挂起的信号就会被处理。这为实现"临界区"（critical section）提供了一种简单机制——在临界区内阻塞信号，退出临界区时解除阻塞。

**子进程继承信号掩码**：注意子进程中也调用了 `sigprocmask(SIG_UNBLOCK, ...)`。这是因为 `fork` 会完整复制父进程的信号掩码。如果子进程不解除对 `SIGCHLD` 的阻塞，而子进程自身又有子进程的话，就会导致信号被永久阻塞。这是一个容易遗漏的细节。
:::

::: tip 重难点解析
**`sigset_t` 的位图实现与信号操作原语**

`sigset_t` 不是一个魔法类型——它本质上就是一个**位图（bitmap）**。在 Linux 内核中，它被定义为一个 unsigned long 数组：

```c
// Linux 内核中的 sigset_t 定义（简化）
typedef struct {
    unsigned long sig[_NSIG_WORDS];  // _NSIG_WORDS 确保能容纳所有信号
} sigset_t;
```

每个比特位对应一个信号：位 0 对应信号 1 (SIGHUP)，位 1 对应信号 2 (SIGINT)，以此类推。操作函数本质上就是位运算：

```c
// sigaddset(&set, SIGCHLD) 的位运算等价：
set->sig[SIGCHLD / (8*sizeof(long))] |= (1UL << (SIGCHLD % (8*sizeof(long))));

// sigdelset(&set, SIGCHLD) 的等价：
set->sig[SIGCHLD / (8*sizeof(long))] &= ~(1UL << (SIGCHLD % (8*sizeof(long))));

// sigismember(&set, SIGCHLD) 的等价：
return (set->sig[SIGCHLD / (8*sizeof(long))] >> (SIGCHLD % (8*sizeof(long)))) & 1;
```

**为什么 SIGKILL 和 SIGSTOP 不可被阻塞、捕获或忽略？**

这是内核强制执行的安全机制，硬编码在 `sigprocmask` 和 `sigaction` 的实现中。如果进程可以忽略 `SIGKILL`，那么恶意或出错的进程就可能永远无法被终止，系统管理员将失去对系统的控制。同样，如果 `SIGSTOP` 可以被忽略，进程就无法被作业控制暂停（Ctrl+Z 将失效）。

```c
// 内核在 sigprocmask 中的检查（简化伪代码）
int sigprocmask(int how, sigset_t *set, sigset_t *oldset) {
    if (how == SIG_BLOCK) {
        sigdelset(set, SIGKILL);  // 强制清除：绝不允许阻塞 SIGKILL
        sigdelset(set, SIGSTOP);  // 强制清除：绝不允许阻塞 SIGSTOP
    }
    // ... 然后才真正更新进程的 blocked 掩码
}
```

值得注意的是，`SIGKILL` 甚至不遵循"内核态返回用户态时递送"的常规路径——内核收到 SIGKILL 后会尽可能快地终止目标进程，绕过所有安全检查直接回收资源。SIGKILL 和 SIGSTOP 的这种"不可阻挡"特性是 UNIX 安全模型的基石。
:::

    ```sh
    myth22> ./job-list-synchronization
    Wed Oct 11 9:49:23 PDT 2017
    Job 29619 added to job list.
    Job 29619 removed from job list.
    Wed Oct 11 9:49:24 PDT 2017
    Job 29620 added to job list.
    Job 29620 removed from job list.
    Wed Oct 11 9:49:25 PDT 2017
    Job 29621 added to job list.
    Job 29621 removed from job list.
    myth22>
    ```

# 进程间通信
* 进程可以使用 `kill` 系统调用通过信号向其他进程发送消息。
    * 原型：

    ```c
    int kill(pid_t pid, int signum);
    ```

    * 类似于 `/bin/kill` shell 命令。
        * 不幸的是它的命名，因为 kill 暗示着 `SIGKILL`，而后者暗示着死亡。
        * 之所以这样命名，是因为在早期的 UNIX 实现中，大多数信号的默认操作
          只是终止目标进程（由 `pid` 标识）。
    * 如果调用失败（通常是因为调用进程没有权限
      向目标进程发送信号），返回 -1，如果一切正常则返回 0。
    * `pid` 参数被重载以提供多种信号发送策略：
        * 当 `pid` 为正数时，目标是具有该 `pid` 的进程。
        * 当 `pid` 为小于 -1 的负数时，目标是进程组
          `abs(pid)` 中的所有进程。
        * `pid` 也可以是 0 或 -1，但我们现在不需要关心这些（不过如果你好奇，
          它们在 B&O 教材中有记录）。

::: tip 重难点解析
**`kill` 的命名误导**：`kill()` 系统调用的名字容易让人误解。它并不一定意味着"杀死进程"——它的作用仅仅是向目标进程发送一个信号。目标进程收到信号后如何响应，取决于该信号的默认行为以及目标进程是否安装了自定义信号处理器。例如，`kill(pid, SIGUSR1)` 发送的是用户自定义信号，目标进程可能用它来触发特定的业务逻辑，而不是终止。在 CS111 中你会看到，`kill` 也用于线程间通信的某些场景。

**进程组与负 pid 参数**：`kill` 支持向整个进程组广播信号（通过负的 pid 值）。进程组是 shell 作业控制的基础——当你按下 ctrl-c 时，`SIGINT` 会发送给前台进程组中的所有进程，而不仅仅是单个进程。这对于管理管道中的多个进程（如 `ls | sort`）至关重要。
:::

::: tip 重难点解析
**`kill(pid, 0)` — 一个极其实用的"进程存活检查"惯用法**

`kill()` 的签名中有一个鲜为人知但极其有用的特性：当 `signum` 参数为 0 时，`kill` **不发送任何信号**，但它会进行所有的错误检查：

```c
// kill(pid, 0) 的惯用法：检查进程是否存在
if (kill(pid, 0) == 0) {
    // 进程存在，且我们有权限向它发送信号
} else if (errno == ESRCH) {
    // 进程不存在（或已经成为僵尸且被回收）
} else if (errno == EPERM) {
    // 进程存在，但我们没有权限向它发送信号
    // （这本身也证明进程存在——只不过属于其他用户）
}
```

这个惯用法常常被称为"信号 0"技巧（null signal trick），在很多场景中非常实用：

- **守护进程健康检查**：daemon 可以用它检查子进程是否仍在运行
- **shell 作业管理**：在你自己的 stsh 中，可以用它检查作业表中的 pid 是否仍有效
- **锁文件清理**：如果 pid 文件中的进程已不存在，清除过期锁文件

**对比 `/proc` 文件系统方法**：另一种检查进程存在的方式是 `stat("/proc/<pid>", &buf)`，但 `kill(pid, 0)` 更轻量——它是一个系统调用，不需要遍历文件系统。

**`kill()` 的完整 pid 参数语义**：
| pid 值 | 含义 |
|--------|------|
| `> 0` | 向特定进程 pid 发送信号 |
| `== 0` | 向与调用者同进程组的所有进程发送信号 |
| `== -1` | 向所有有权限发送的进程发送信号（除 init 和自身） |
| `< -1` | 向进程组 `abs(pid)` 中的所有进程发送信号 |
:::

# Kill 谜题
* 以下是一个简短的谜题，以确保你理解了关键概念：
    * 我们首先来完成一个小谜题，以确认你们都理解了进程的
      工作流程，并理解了 `kill` 如何触发各种信号处理器的
      执行。（这个示例中省略了错误检查，因为它太小了，我们
      假设，为简单起见，没有任何会出错的地方）。
    * 把它们放在一起。以下
      [程序](http://cs110.stanford.edu/spring-2017/examples/processes/kill-puzzle.c)的可能输出（注意是复数！）是什么？

    ```c
    static pid_t pid;
    static int counter = 0;

    static void parentHandler(int unused) {       static void childHandler(int unused) {
      counter++;                                    counter += 3;
      printf("counter = %d\n", counter);            printf("counter = %d\n", counter);
      kill(pid, SIGUSR1);                         }
    }


    int main(int argc, char *argv[]) {
      signal(SIGUSR1, parentHandler);
      if ((pid = fork()) == 0) {
        signal(SIGUSR1, childHandler);
        kill(getppid(), SIGUSR1);
        return 0;
      }

      waitpid(pid, NULL, 0);
      counter += 7;
      printf("counter = %d\n", counter);
      return 0;
    }
    ```

# Kill 谜题（续）
* 预期输出是什么？
    * 确保你理解了为什么上一张幻灯片中的程序能够产生两种
      不同的输出，这取决于处理器的调度。（提示：子进程可能在
      `parentHandler` 有机会杀死它之前就已经退出了！）

    ```sh
    myth22> ./kill-puzzle
    counter = 1
    counter = 8
    myth22> ./kill-puzzle
    counter = 1
    counter = 3
    counter = 8
    myth22>
    ```

::: tip 重难点解析
**程序输出的不确定性**：这个谜题展示了并发程序的核心特性——同一程序可能产生不同输出。两种可能的执行路径：

**路径 A（输出 counter = 1, counter = 8）**：
1. 子进程调用 `kill(getppid(), SIGUSR1)` 向父进程发送信号
2. 父进程的 `parentHandler` 执行：`counter` 变为 1，打印 "counter = 1"
3. `parentHandler` 内调用 `kill(pid, SIGUSR1)` 向子进程发送信号
4. 但此时子进程已经执行完 `return 0` 退出了（时间窗口极小），信号无人处理
5. 父进程 `waitpid` 回收子进程后继续执行 `counter += 7`，变为 8，打印 "counter = 8"

**路径 B（输出 counter = 1, counter = 3, counter = 8）**：
1-2 同上
3. `parentHandler` 内调用 `kill(pid, SIGUSR1)` 时子进程尚未退出
4. 子进程的 `childHandler` 执行：`counter` 变为 `1 + 3 = 4`（注意全局变量是共享的！），打印 "counter = 4"...

等等，输出是 `counter = 3`，让我重新分析。实际上，在 `fork` 之后，子进程拥有父进程地址空间的独立副本，所以全局变量 `counter` 在父子进程之间是独立的。子进程中的 `counter` 初始值也是 0（因为 `fork` 发生在 `counter` 被修改之前）。所以子进程 `childHandler` 执行 `counter += 3` 后，子进程中的 `counter` 是 3，打印 "counter = 3"。然后子进程退出。父进程中的 `counter` 不受子进程修改影响，仍然是 1。父进程继续 `counter += 7` → 8。

这个谜题精妙地展示了：(1) `fork` 后进程地址空间是独立的——子进程对全局变量的修改不影响父进程；(2) 信号处理的异步性导致的不确定执行顺序；(3) `kill` 发送信号给已经退出的进程时，信号会被忽略。
:::
