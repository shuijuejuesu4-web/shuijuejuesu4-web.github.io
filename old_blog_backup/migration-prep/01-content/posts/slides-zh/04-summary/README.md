# 公告

* 作业 1 将于周三晚上截止。
* 助教办公时间！
    * 完整的办公时间矩阵已发布在[这里](http://web.stanford.edu/class/cs110/autumn-2017/calendar.html)。
    * 办公时间是提问有关讲座内容和作业说明的好地方，这些问题可能不容易在 Piazza 上解决。
* 本周阅读材料：
    * 仔细阅读 Bryant 和 O'Hallaron 的教材，第 2 章（其中大部分你们已经了解：`open`、`read`、`write` 等）和第 1 章（按此顺序）。在本周内慢慢仔细阅读这两章。
* 预计今天将完成文件系统、命名和分层的内容，如果稍早完成，我们将开始多进程和异常控制流。
     * 所有多进程和异常控制流示例可以在 `/usr/class/cs110/lecture-examples/autumn-2017/processes` 中找到。

# 多进程：第一个程序

* 新的系统调用：`fork`
    * 这是一个最简单的能创建其他进程的程序。它使用了一个名为 `fork` 的系统调用。
    * 代码在讲座示例文件夹中：`processes/basic-fork.c`。代码也在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/processes/basic-fork.c)。

    ```c
    static const int kForkFailed = 1;
    int main(int argc, char *argv[]) {
      printf("Greetings from process %d! (parent %d)\n", getpid(), getppid());
      pid_t pid = fork();
      exitIf(pid == -1, kForkFailed, stderr, "fork function failed.\n");
      printf("Bye-bye from process %d! (parent %d)\n", getpid(), getppid());
      return 0;
    }
    ```

* `fork` 被调用一次，但返回两次。
    * `fork` 知道如何克隆调用进程，创建其几乎完全相同的副本，并将其调度，就好像原始进程的第二个副本一直在运行一样。所有段（data、bss、init、stack、heap、text）都被忠实复制，所有打开的文件描述符也从第一个进程复制并捐赠给克隆进程。

::: tip 重难点解析
**fork 的"调用一次、返回两次"**：这是理解 Unix 进程模型最关键的概念。`fork()` 是在父进程中调用的，但它会返回两次：一次返回给父进程（返回值为子进程的 PID），一次返回给子进程（返回值为 0）。这意味着 `fork()` 调用之后的代码会被两个进程中的每一个执行。理解这一点，你就理解了为什么上述程序会打印一条问候但两条告别。

**写时复制（Copy-on-Write，COW）**：虽然概念上讲 `fork` 复制了父进程的整个地址空间，但现代操作系统实际上使用了一种优化——写时复制。`fork` 之后，父子进程的页表暂时指向相同的物理页，只有当某个进程尝试修改某一页时，内核才真正复制该页。这大大降低了 `fork` 的开销（特别是对于紧接着调用 `exec` 的场景——复制了地址空间但马上又要丢弃）。

**fork 后父子进程的独立性**：`fork` 之后，父子进程拥有独立的地址空间——一个进程修改全局变量不会影响另一个。但是，打开的文件描述符是共享的——父子进程指向内核中相同的"打开文件表"条目，共享文件偏移量。这也是为什么在 shell 实现中，父进程在 fork 后通常需要关闭某些管道端点的原因。
:::

::: tip 重难点解析
**fork 的内核实现深入——`copy_process` 的全过程**：

`fork` 之所以能"调用一次，返回两次"，根源在于内核实现。以下是简化但准确的 `fork` 内核路径：

1. **进入内核**：`syscall` 指令触发，CPU 切换到内核态，执行 `sys_fork`。

2. **`copy_process()` 核心步骤**：
   - `dup_task_struct()`：分配新 `task_struct`（进程描述符，大小约 2KB），复制父进程的 `thread_struct`（保存的寄存器状态）。
   - 复制 `mm_struct`（内存描述符）：复制页表，但**不复制物理页**——所有页表条目被标记为只读（设置 COW）。
   - 复制 `files_struct`（fd 表）：遍历每个 fd，增加 `struct file` 的 `f_count`（引用计数），父子共享同一 `struct file`，因此共享 `f_pos`。
   - 复制 `fs_struct`（cwd, umask）、`sighand_struct`（信号处理表）。
   - `alloc_pid()`：在 pid namespace 中分配新 PID。

3. **设置返回值——"返回两次"的真相**：内核在子进程的 `task_struct` 中手动修改保存的寄存器上下文：
   ```c
   // x86-64: 返回值通过 RAX 寄存器传递
   child->thread.sp = 分配的新内核栈;
   child->thread.regs.rax = 0;  // 子进程看到 fork() 返回 0
   // 父进程的 rax 由正常路径设置为子进程 PID
   ```

4. **唤醒子进程**：`wake_up_new_task()` 将子进程放入 runqueue，调度器择机执行。子进程从 `fork()` 返回点之后的第一条指令开始运行。

**资源继承表**：
| 资源 | 机制 | 共享/独立 |
|------|------|----------|
| 地址空间 (mm) | COW 页表复制 | 逻辑独立，物理按需复制 |
| fd 表 (files) | 原子增加 f_count | 共享同一 struct file |
| 信号处理 (sighand) | 深复制 + 引用计数 | 独立副本 |
| PID | 新分配 | 独立 |
| PGID/SID | 继承 | 共享 |
:::

::: tip 重难点解析
**COW（Copy-on-Write）的虚拟内存实现——按需复制的核心机制**：

COW 不是简单的"只读标记"，而是硬件（MMU）和软件（内核 page fault handler）协同工作的一套精妙机制：

1. **`fork` 时刻**：内核复制父进程的页表（页表本身被复制，不是共享），但新旧页表中的所有用户空间 PTE（Page Table Entry）都指向相同的物理页帧。关键操作：将每个 PTE 的写权限位清除（在 x86-64 上，清除 PTE 的 `_PAGE_BIT_RW`），即使该页原本是可写的。同时设置 PTE 的 `_PAGE_BIT_COW`（软件定义位，硬件不可见）。

2. **首次写入触发 page fault**：当父进程或子进程尝试写某个 COW 页时，MMU 检测到写权限缺失，触发 **#PF（Page Fault）异常**。CPU 将故障地址保存在 `CR2` 寄存器中并转入内核的 page fault handler。

3. **内核处理 page fault**：
   ```c
   // 简化的内核 page fault handler 逻辑:
   if (fault on a present page with write bit cleared) {
       if (page is COW) {
           if (page refcount == 1) {
               // 只有一个进程在用这个页 → 无需复制
               just set writable bit;
           } else {
               new_page = alloc_page();          // 分配新物理页
               copy_page(new_page, old_page);    // 复制内容
               update faulting process's PTE to point to new_page (set writable);
               decrement old_page refcount;
               // 另一个进程的 PTE 保持指向 old_page, 恢复其写权限
           }
       }
   }
   ```
   关键优化：如果物理页的引用计数已降为 1（另一个进程已通过 COW 或 `exec` 放弃了对该页的引用），则无需复制——直接恢复写权限即可。

4. **`fork + exec` 的零复制路径**：子进程 `exec` 时调用 `exec_mmap()`，创建一个全新的 `mm_struct`，并释放旧的 `mm_struct`。对于每个 COW 共享的物理页，refcount 递减。父进程中尚未被写入的页（refcount 从 2 降到 1）可以直接恢复写权限——没有任何物理页被实际复制。这就是为什么即使父进程占用了 2GB 内存，`fork + exec` 也能在微秒级完成。
:::

    * 唯一的区别：`fork` 在新进程（**子进程**）中的返回值为 0，而 fork 在生成进程（**父进程**）中的返回值为子进程的进程 ID。返回值可以用于将两个进程中的每一个导向不同的方向（尽管在这个入门示例中，我们并没有这样做）。
    * 因此，上述程序的输出实际上是两个进程的输出。我们应该预期会看到一条问候但两条告别。
    * 关键观察：每条告别信息由两个不同的进程插入控制台。每条行执行的顺序，原则上，是无法预测的。系统的调度程序决定子进程还是父进程先打印它的告别信息（虽然在多核机器上，子进程和父进程可能正并行运行）。

::: warning 注意事项
**fork 之后的执行顺序是不确定的**：父子进程在 `fork` 之后的相对执行顺序完全由操作系统的调度程序决定。特别是在多核系统上，两者可能真正并行运行。永远不要假设父进程先于子进程执行，反之亦然。如果你需要特定的执行顺序（例如父进程等待子进程完成），必须使用 `waitpid` 等同步机制。
:::

# 爆炸式的 `fork` 图

* 理解工作流程，小心 `fork` 炸弹炸毁你的系统。
    * 虽然你很少有理由以这种方式使用 fork（你也不应该，否则你会开始给你正在使用的共享系统带来负担和/或达到并发进程数量的限制），但追踪一个短程序，其中被 fork 出的进程自身也调用 `fork`，是很有启发性的。
    * 代码在讲座示例文件夹中：`processes/fork-puzzle.c`。代码也在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/processes/fork-puzzle.c)。
    * 看看这个：

    ```c
    static const char const *kTrail = "abcd";
    static const int kForkFail = 1;
    int main(int argc, char *argv[]) {
      size_t trailLength = strlen(kTrail);
      for (size_t i = 0; i < trailLength; i++) {
        printf("%c\n", kTrail[i]);
        pid_t pid = fork();
        exitIf(pid == -1, kForkFail, stderr, "Call to fork failed.");
      }
      return 0;
    }
    ```

    * 相当明显的：一个 a 被即将成为曾祖父的进程打印。
    * 不太明显的：第一个子进程和父进程各自从 fork 返回并继续在镜像进程中运行，每个都有自己的一份全局字符串 "abcd" 的副本，并且每个都在循环中推进到 i++ 行，将 0 提升为 1。现在应该清楚了，两个 b 将被打印。
    * 需要回答的关键问题：
        * 会打印多少个 c？
        * 会打印多少个 d？
        * 两个 b 是否一定一个接一个地打印？

::: tip 重难点解析
**fork 谜题解析——指数级进程增长**：这个程序是理解 `fork` 语义的经典练习题。让我们逐步追踪：

- 初始进程（P0）：打印 `a`，然后 fork 出 P1。P0 和 P1 都从循环继续。
- P0：i=1，打印 `b`，fork 出 P2。P0 i=2；P2 i=2。
- P1：i=1，打印 `b`，fork 出 P3。P1 i=2；P3 i=2。
- 现在有 4 个进程（P0、P1、P2、P3），每个 i=2，各自打印 `c` 并 fork。
- 结果是：4 个 `c`、8 个 `d`。

**一般规律**：循环 n 次 `fork`，最终产生的进程数为 2^n（包括最初的那个）。每次迭代中，当前的每个进程都 fork 出一个新进程，进程数量翻倍。这也是为什么"fork 炸弹"（无限循环 fork）能迅速耗尽系统资源。

**关于输出顺序**：`a` 首先出现，然后是两个 `b`，但这两个 `b` 的顺序是不确定的。同理，四个 `c` 之间的相对顺序也是不确定的。唯一能保证的是 `a` 在 `b` 之前，`b` 在 `c` 之前，因为同一进程内代码是顺序执行的——但不同进程之间的顺序完全由调度器决定。
:::

# 在父进程和子进程之间同步执行

* `waitpid` 指示一个进程阻塞直到另一个进程退出。
    * 第一个参数指定等待集合，目前就是需要在 `waitpid` 返回之前完成的子进程的 ID。（还有其他可能性，但我们稍后会探讨）。
    * 第二个参数提供一个整数的地址，子进程终止状态信息可以放入其中（或者如果我们不关心获取该信息，可以传入 `NULL`）。
    * 第三个参数是我们稍后将学习的一组位标志。目前，我们只用 0 作为所需的参数值，这意味着 `waitpid` 应该仅在子进程退出时返回。

::: tip 重难点解析
**waitpid 参数详解**：
- `pid > 0`：等待特定的子进程。
- `pid == -1`：等待任意子进程（哪个先退出就回收哪个）。
- `pid == 0`：等待与调用进程同进程组的任意子进程。
- `pid < -1`：等待进程组 ID 等于 `|pid|` 的任意子进程。

第三个参数的常用选项：
- `0`：默认行为，阻塞直到子进程退出。
- `WNOHANG`：非阻塞——如果没有子进程退出，立即返回 0 而不是阻塞。
- `WUNTRACED`：如果子进程被停止（如收到 SIGSTOP），也返回。
:::

    * 返回值是退出的子进程的 ID，如果 `waitpid` 被调用但没有子进程可等待，则返回 -1。
    * 下一个示例的代码在讲座示例文件夹中：`processes/parent-child.c`。代码也在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/processes/parent-child.c)。
    * 下面是该示例：

    ```c
    static const int kForkFailure = 1;
    int main(int argc, char *argv[]) {
      printf("I'm unique and just get printed once.\n");
      pid_t pid = fork(); // returns 0 within child, returns pid of child within fork
      exitIf(pid == -1, kForkFailure, stderr, "Call to fork failed... aborting.\n");
      bool parent = pid != 0;
      if ((random() % 2 == 0) == parent) sleep(1); // force exactly one of the two to sleep
      if (parent) waitpid(pid, NULL, 0); // parent shouldn't exit until it knows its child has finished
      printf("I get printed twice (this one is being printed from the %s).\n", parent  ? "parent" : "child");
      return 0;
    }
    ```

    * 上面的示例使用抛硬币的方式来诱使两个进程中的一个睡眠一秒钟，这通常足够另一个进程先打印它需要打印的内容所需的时间。
    * 父线程选择等待子线程退出，然后才允许自己退出。这类似于父母不能睡觉直到知道孩子已经睡了，并且它象征着我们本季度将要大量看到的同步指令类型。

::: tip 重难点解析
**僵尸进程（Zombie Process）与资源回收**：如果父进程没有调用 `waitpid` 就退出了（或者子进程退出时父进程仍在运行但没有 `wait`），子进程会变成僵尸进程——它已经终止，但在进程表中仍占据一个条目，等待父进程回收其退出状态。如果父进程先于子进程退出，子进程变成"孤儿进程"，由 init 进程（PID 1）收养，init 会定期调用 `wait` 来回收它们。

**为什么 `waitpid` 很重要？** 这不仅仅是获得子进程退出状态的问题——如果不回收，僵尸进程会一直占用进程表条目（操作系统对进程数量有上限），最终导致系统无法创建新进程。这是一种资源泄漏，类似于忘记 `free` 导致的内存泄漏。CS111 课程对进程调度和生命周期管理有更深入的讨论。
:::

    * 请理解，最后的 `printf` 被执行了两次。然而，子进程总是第一个执行它，因为父进程在 `waitpid` 调用中被阻塞，直到子进程执行完**所有**内容。

# 父进程和子进程通常会分道扬镳：

* 下面的程序是 `fork` 在实践中如何使用的一个步骤：
    * 看看这个（代码在讲座示例文件夹 `processes/separate.c` 中，也可以在[这里](http://cs110.stanford.edu/autumn-2017/examples/processes/separate.c)查看）

    ```c
    int main(int argc, char *argv[]) {
      printf("Before.\n");
      pid_t pid = fork();
      exitIf(pid == -1, kForkFailed, stderr, "Fork function failed.\n");
      printf("After.\n");
      if (pid == 0) {
        printf("I'm the child, and the parent will wait up for me.\n");
        return 110; // contrived exit status
      } else {
        int status;
        exitUnless(waitpid(pid, &status, 0) == pid, kWaitFailed, stderr, 
                   "Parent's wait for child process with pid %d failed.\n", pid);
        if (WIFEXITED(status)) {
          printf("Child exited with status %d.\n", WEXITSTATUS(status));
        } else {
          printf("Child terminated abnormally.\n");
        }
        return 0;
      }
    }
    ```

    * 上面的示例将子进程导向一个方向，父进程导向另一个方向。

::: tip 重难点解析
**fork 后的典型模式——分道扬镳**：这是一个经典的 `fork` 使用模式：父进程和子进程通过 `fork` 的返回值区分身份，然后各自执行完全不同的代码路径。具体来说：
- 子进程（`pid == 0`）执行 `if` 分支，做自己的事情后退出并返回状态码 110。
- 父进程（`pid != 0`）执行 `else` 分支，通过 `waitpid` 阻塞等待子进程完成，然后提取退出状态。

这种"fork 后分叉"的模式是 shell 实现的基础——父进程（shell）fork 一个子进程来执行用户输入的命令，自己则等待命令完成后再显示下一个提示符。

**退出状态解析宏**：
- `WIFEXITED(status)`：检查子进程是否正常退出（通过 `exit` 或 `return`）。
- `WEXITSTATUS(status)`：提取子进程 `exit` 参数的低 8 位。
- `WIFSIGNALED(status)`：检查子进程是否因信号而终止。
- `WTERMSIG(status)`：提取终止子进程的信号编号。
- `WIFSTOPPED(status)`：检查子进程是否被停止。
- `WSTOPSIG(status)`：提取使子进程停止的信号编号。
:::

::: tip 重难点解析
**`waitpid` 状态码的位级编码——你必须掌握的核心细节**：

`waitpid` 的第二个参数 `int *status` 看似是一个简单的整数，但实际上它用不同的位段编码了多种信息。理解这个位布局是正确使用 `waitpid` 的关键，也是期中/期末考试的必考内容。

**完整的 32 位 status 位布局**（Linux/x86-64）：

```
Bit:  31...16  |  15  14  13  12  11  10   9   8  |   7  |   6...0
      未使用    |  exit status (exit() 的参数)        | core |  termination signal
               |  (仅在正常退出时有效)                | dump |  (仅在异常终止时有效)

详细分解:
  bits 15..8: 子进程的退出码 (0-255), 仅当 WIFEXITED() 为真时有意义
  bit  7:     core dump 标志, 仅当 WIFSIGNALED() 为真时有意义  
  bits 6..0:  终止信号编号 (1-31), 仅当 WIFSIGNALED() 为真时有意义
```

**宏实现原理**（来自 glibc 的实际定义）：

```c
// 判断是否正常退出: 检查终止信号部分是否为 0
#define WIFEXITED(status)   (((status) & 0x7f) == 0)
//  如果低 7 位全是 0, 则不是被信号杀死的 → 正常退出

// 提取退出码: 取 bits 8-15 
#define WEXITSTATUS(status) (((status) >> 8) & 0xff)
//  右移 8 位再掩码低 8 位 → 得到 exit(n) 中的 n (0-255)

// 判断是否被信号杀死: 检查低 7 位是否为非零信号编号
// (核心: 进程正常退出时低 7 位为 0; 被信号杀死时为信号编号)
#define WIFSIGNALED(status) (((status) & 0x7f) != 0 && ((status) & 0x7f) != 0x7f)
//  注意: 0x7f = 127 = 被停止(stopped), 不是被杀死

// 提取终止信号: 低 7 位
#define WTERMSIG(status)    ((status) & 0x7f)

// 检查 core dump: bit 7
#define WCOREDUMP(status)   ((status) & 0x80)

// 判断是否被停止 (用于 WUNTRACED)
#define WIFSTOPPED(status)  (((status) & 0xff) == 0x7f)
//  整个低 8 位为 0x7f 表示进程被停止(如收到 SIGSTOP)

// 提取停止信号: bits 8-15
#define WSTOPSIG(status)    WEXITSTATUS(status)
//  与被停止的进程一样, 停止信号存储在 bits 8-15
```

**构建 status 值**：子进程调用 `exit(110)` 时，内核将 status 构建为：
```c
// 内核中的逻辑 (简化):
status = (110 << 8);  // 退出码放入 bits 8-15, 低 7 位保持为 0
// status = 0x6E00 = 28160

// 验证:
WIFEXITED(28160)  → ((28160 & 0x7f) == 0) → true
WEXITSTATUS(28160) → ((28160 >> 8) & 0xff) → 110 ✓
```

**被信号杀死时的 status 构建**：子进程收到 SIGSEGV (信号 11) 并产生 core dump：
```c
status = 11;          // 信号编号放在低 7 位: 0x000B
status |= (1 << 7);   // core dump 标志:  0x008B
// status = 0x008B = 139

WIFSIGNALED(139) → ((139 & 0x7f) != 0) → true
WTERMSIG(139)    → (139 & 0x7f) → 11 (SIGSEGV) ✓
WCOREDUMP(139)   → (139 & 0x80) → true ✓
WIFEXITED(139)   → ((139 & 0x7f) == 0) → false ✓
```

**记忆技巧**：想象 status 的低 16 位 = `0xTTSC`，其中 `TT` = 退出码 (当 `S=0x00`) 或停止信号 (当 `S=0x7F`)，`S` = 信号编号或 `0x7F`，`C` = core dump 标志。考试时画一张位域图比背宏更容易得分。
:::

    * 父进程正确地等待子进程完成，并且这个示例（与上一个幻灯片中的最后一个示例不同）实际上通过断言 `waitpid` 的返回值与退出的子进程的进程 ID 匹配来正确地执行。
    * 父进程还从 `waitpid` 调用中提取了关于子进程的退出状态信息，并使用 `WIFEXITED` 宏检查了这个 `status` 参数的一些高位以确认进程正常退出，还使用了 `WEXITSTATUS` 宏从参数中提取低八位来生成子进程的返回值。
    * 查看 `waitpid` 的手册页以获取关于所有不同宏的详细信息。（你的教材也非常好地涵盖了所有这些内容）。
