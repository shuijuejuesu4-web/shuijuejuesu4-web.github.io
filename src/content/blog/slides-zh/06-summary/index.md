---
title: "进程间通信：信号与信号处理器"
description: "进程间通信：信号与信号处理器"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告
* Assignment 1 的成绩报告已于今早通过邮件发送。
    * 中位数成绩为满分 56 分（满分 56 分）。
    * 平均代码风格评分介于 'solid' 和 'minor-problems' 之间。
    * 在 CS110 中，功能正确性的权重是代码风格的五倍。
        * 我关心软件工程和代码清晰度，但是...
        * 我更关心能正常工作的代码。
        * 代码风格成绩转换规则如下：
            * `exceptional`    -> 105%
            * `solid`          -> 95%
            * `minor-problems` -> 80%
            * `major-problems` -> 60%
* Assignment 2 的截止日期是本周三晚上 11:59。
* Assignment 3 将于周三发布，截止日期为 11 天后。
* 阅读材料：
    * 完成 B&O 第 2 章（完整教材的第 10 章）的阅读，这样你可以确认自己已经掌握了其中的大部分内容，
      因为这些内容我在前两周的课程中已经涵盖了很大一部分。
    * 完成 B&O 第 1 章（完整教材的第 8 章）的阅读，重点关注第 5 节，该节涵盖了
      进程组（process groups）、信号（signals）和信号处理器（signal handlers），
      这三者都与你接下来的几个作业相关。

# 今日内容
* 今天的话题都非常精彩。
    * 我们需要完成上周三留下的几个课堂示例。特别地，我们想要实现一组微型 shell
      来说明 `fork`、`waitpid` 和 `execvp` 的工作原理。
    * 我想介绍管道（pipe）的概念、`pipe` 和 `dup2`
      函数，以及如何利用它们在不同进程之间实现更复杂的通信。
    * 今天晚些时候或周三第一件事，我们将开始讨论信号和信号处理器。

# `simplesh` 的核心实现，1.0 版
* 这是你能想象到的最好的 `execvp` 示例：一个微型
  shell，与你从学习 UNIX 以来一直在使用的那种 shell 非常相似。
    * 依赖于 `fork`、`waitpid`，
      和 `execvp`。
    * 第一个版本以读-求值-打印循环（read-eval-print loop，REPL）的方式运行，
      对我们在键盘上输入的许多命令做出响应，通过 fork 出子进程来执行。
        * 每个子进程最初都是 shell 的一个深度克隆。
        * 每个子进程接着用我们指定的新进程映像替换自己的进程映像
          （例如 `ls`、`cp`、
          我们自己写的 CS110 `search`（我们在开课第二天写的），
          甚至 `emacs`）。
        * 末尾的 `&` 符号（例如 `emacs` `&`）
          表示该命令在后台执行，不会阻塞对终端的访问。
    * `simplesh` 的实现将在接下来的三张幻灯片中展示。
      当辅助函数不依赖于 CS110 概念时，我省略了它们的实现（但会在课堂上描述）。

::: tip 重难点解析
**fork-execvp-waitpid 模式**：这是 UNIX 进程管理的核心设计模式。`fork()` 创建一个子进程（几乎是父进程的完整拷贝），然后子进程通过 `execvp()` 加载并执行一个新的程序映像，而父进程则通过 `waitpid()` 等待子进程结束。这种"创建-替换-等待"的三步曲是所有 shell 程序背后的基本原理。理解这个模式是掌握进程管理的关键。
:::

# `simplesh` 实现的核心
* `main` 的实现：
    * 以下是上半部分（完整实现见[此处](http://cs110.stanford.edu/autumn-2017/examples/processes/simplesh.c)）

    ```c
    int main(int argc, char *argv[]) {
      while (true) {
        char command[kMaxCommandLength + 1];
        readCommand(command, sizeof(command) - 1);
        if (feof(stdin)) break;
        char *arguments[kMaxArgumentCount + 1];
        int count = parseCommandLine(command, arguments, 
                                     sizeof(arguments)/sizeof(arguments[0]));
        if (count == 0) continue;
        bool builtin = handleBuiltin(arguments);
        if (builtin) continue; // it's been handled, move on
        bool isBackgroundProcess = strcmp(arguments[count - 1], "&") == 0;
        if (isBackgroundProcess) arguments[--count] = NULL; // overwrite "&"
        pid_t pid = forkProcess();
    ```

# `simplesh` 实现的核心（续）
* `main` 的实现：
    * 以下是下半部分

    ```c
        if (pid == 0) {
          if (execvp(arguments[0], arguments) < 0) {
            printf("%s: Command not found\n", arguments[0]);
            exit(0);
          }
        }

        if (!isBackgroundProcess) {
          waitForChildProcess(pid);
        } else {
          printf("%d %s\n", pid, command);
        }
      }

      printf("\n");
      return 0;
    }
    ```

::: tip 重难点解析
**前台与后台进程的区别**：在上面的代码中，`isBackgroundProcess` 标志决定父进程的行为。对于前台进程（无 `&`），父进程调用 `waitForChildProcess()` 阻塞等待子进程完成；对于后台进程（有 `&`），父进程仅打印子进程的 pid 后继续循环，不等待。这正是 shell 中 `&` 符号的实现原理——它告诉 shell 不要等待该命令完成，立即返回提示符。注意，后台进程的输出仍会打印到终端，可能与 shell 的提示符混在一起。
:::

# `simplesh` 实现的核心（续）
* 辅助函数
    * 以下是一些确实依赖于 CS110 内容的辅助函数：

    ```c
    static bool handleBuiltin(char *arguments[]) {
      if (strcasecmp(arguments[0], "quit") == 0) exit(0);
      return strcmp(arguments[0], "&") == 0;
    }

    static pid_t forkProcess() {
      pid_t pid = fork();
      exitIf(pid == -1, kForkFailed, stderr, "fork function failed.\n");
      return pid;
    }

    static void waitForChildProcess(pid_t pid) {
      exitUnless(waitpid(pid, NULL, 0) == pid, kWaitFailed,
                 stderr, "Error waiting in foreground for process %d to exit", pid);
    }
    ```

::: tip 重难点解析
**内置命令 vs 外部命令**：`handleBuiltin` 展示了 shell 中"内置命令"的处理方式。像 `quit` 这样的命令不需要 fork 新进程，而是直接在 shell 进程内部处理（这里直接调用 `exit(0)` 退出整个 shell）。这与 `ls`、`cp` 等外部命令形成对比——外部命令需要通过 `fork()` + `execvp()` 在新进程中执行。常见的 shell 内置命令还包括 `cd`（改变当前工作目录）、`export`（设置环境变量）等——这些命令必须内置于 shell 中，因为它们需要修改 shell 进程自身的状态。
:::

::: tip 重难点解析
**为什么 `cd` 必须是内置命令？**

这是一个经典的面试题。`cd` 改变的是当前进程的**工作目录**（cwd, current working directory），该信息存储在内核的进程控制块（PCB）中。如果你把 `cd` 实现为外部命令（即 fork + exec）：

```c
// 假设 /bin/cd 是一个外部命令（这个假设是错误的）
pid_t pid = fork();
if (pid == 0) {
    execvp("/bin/cd", args);  // 子进程修改了自己的 cwd
}
waitpid(pid, NULL, 0);
// 父进程（shell）的 cwd 完全没有变化！
```

子进程在 `execvp` 后修改的是**它自己**的工作目录，而 `fork` 创建的子进程拥有独立的 PCB。子进程退出后，父进程（shell）的工作目录纹丝不动。这就是为什么 `cd` 必须由 shell 直接调用 `chdir()` 系统调用来实现——它必须修改 shell 自身进程的状态。

**PATH 解析机制**：

当你在 shell 中输入 `ls`（不带路径），shell 如何找到 `/bin/ls`？流程如下：

1. shell 读取环境变量 `PATH`（例如 `/usr/local/bin:/usr/bin:/bin`）
2. 对 PATH 中的每个目录，拼接 `<目录>/ls`，然后用 `stat()` 或 `access()` 检查该文件是否存在且可执行
3. 找到第一个匹配项后，将其完整路径传给 `execvp`
4. 实际上，`execvp` 内置了这个逻辑——`execvp("ls", args)` 会自动在 PATH 中搜索，无需你手动拼接。而 `execv` 没有这个功能，它要求你提供完整路径。

这就是为什么课堂上使用 `execvp` 而不是 `execv` —— `execvp` 末尾的 `p` 表示 "PATH search"。
:::

# 实现 `subprocess`
* 介绍 `pipe`：
    * `pipe` 系统调用接收一个未初始化的
      包含两个整数的数组（我们称这个数组为 `fds`），
      并用两个文件描述符填充它，使得所有*写入*
      `fds[1]` 的数据都可以从 `fds[0]` 中*读取*。

    ```c
    int pipe(int fds[]); // fds array should be of length 2, return -1 on error, 0 otherwise
    ```

    * `pipe` 对于允许父进程与 fork 出的子进程进行通信特别有用。
      （回想一下，`fork` 克隆了调用者的虚拟地址空间，**并且**复制了所有打开的文件描述符）。
    * 使用 `pipe`、`fork`、`dup2`、
      `execvp`、`close` 和 `waitpid`，
      我们可以实现 `subprocess` 函数，它依赖于如下
      记录定义，并按以下原型实现：

    ```c
    typedef struct {
      pid_t pid;
      int infd;
    } subprocess_t;
    subprocess_t subprocess(const char *command);
    ```

    * 由 `subprocess` 创建的子进程通过调用
      `"/bin/sh -c command"` 来执行给定的命令（保证是一个 `'\0'` 结尾的 C 字符串）。
      与其等待 `command` 完成，实现会返回一个 `subprocess_t`，
      其中包含 `command` 进程的 pid 和一个文件描述符。具体来说，任意数据可以
      被发布到返回值中的 `infd`，并理解
      这些数据将被 `command` 的标准输入所读取。

::: tip 重难点解析
**管道（pipe）的本质**：管道是 UNIX 中最基本的进程间通信（IPC）机制。它本质上是一个内核维护的环形缓冲区，提供两个文件描述符——一个只读端（`fds[0]`）和一个只写端（`fds[1]`）。管道是单向的：数据只能从写端流向读端。管道的关键特性是：它在 `fork` 之后仍然有效，因为 `fork` 会复制文件描述符表。这意味着父进程和子进程可以通过管道通信——典型场景是父进程向管道写入数据，子进程从管道读取（或反之）。管道也是 shell 中 `|` 运算符（如 `ls | sort`）的底层实现机制。
:::

# 实现 `subprocess`（续）
* 示例客户端应用程序及其输出
    * 以下客户端程序和测试运行精确地展示了
      `subprocess` 应该如何工作：

    ```c
    int main(int argc, char *argv[]) {
      subprocess_t sp = subprocess("/usr/bin/sort");
      const char *words[] = {
        "felicity", "umbrage", "susurration", "halcyon",
        "pulchritude", "ablution", "somnolent", "indefatigable"
      };      
      for (size_t i = 0; i < sizeof(words)/sizeof(words[0]); i++) { 
        dprintf(sp.infd, "%s\n", words[i]);
      }
      close(sp.infd); // effectively sends cntl-D to child process
      int status;
      pid_t pid = waitpid(sp.pid, &status, 0);
      return pid == sp.pid && WIFEXITED(status) ? WEXITSTATUS(status) : -1;
    }
    ```

    * 如果 `subprocess` 实现正确，上述程序的输出
      应该如下所示：

    ```sh
    myth22> ./subprocess-test
    ablution
    felicity
    halcyon
    indefatigable
    pulchritude
    somnolent
    susurration
    umbrage
    ```

::: tip 重难点解析
**关闭写端的作用**：上面代码中的 `close(sp.infd)` 是一个关键步骤。它关闭了父进程持有的管道写端，这实际上向子进程（`sort`）发送了一个 EOF（文件结束）信号。如果不关闭写端，`sort` 会永远阻塞等待更多输入——因为它不知道父进程是否还会写入更多数据。这是管道编程中的一个常见陷阱：必须在不使用时关闭管道的对应端，否则会导致死锁或无限等待。
:::

# 实现 `subprocess`（续）
* 实现在这里（注意：代码很密集）：
    * 以下是 `subprocess` 的实现（我省略了错误
      检查，以便实现的核心逻辑更加清晰）：

    ```c
    subprocess_t subprocess(const char *command) {
      int fds[2];
      pipe(fds);
      subprocess_t process = { fork(), fds[1] };
      if (process.pid == 0) {
        close(fds[1]);
        dup2(fds[0], STDIN_FILENO);
        close(fds[0]);
        char *argv[] = {"/bin/sh", "-c", (char *) command, NULL};
        execvp(argv[0], argv);
      }
      close(fds[0]);
      return process;
    }
    ```

    * 注意，管道的写端被嵌入到了
      `subprocess_t` 中。这样，父进程就知道
      应该往哪里发布文本，使其流到管道的另一端，
      跨越父进程/子进程的边界。
    * 进一步注意，子进程使用 `dup2` 将管道的读端
      与自己的标准输入重新关联。
    * 一旦完成了 `dup2` 的重新关联，
      子进程就可以关闭其管道副本的两端。
    * 父进程不需要管道的读端，因此
      它也可以关闭自己的 `fds[0]` 副本。
    * 完整实现在[此处](http://cs110.stanford.edu/autumn-2017/examples/processes/subprocess.c)。

::: tip 重难点解析
**dup2 与 I/O 重定向**：`dup2(oldfd, newfd)` 是理解 UNIX I/O 重定向的核心函数。它的作用是将 `newfd` 指向的文件描述符表项"覆盖"为 `oldfd` 的副本——也就是说，之后对 `newfd` 的读写操作实际上会作用在 `oldfd` 所指向的文件/管道上。在上面的代码中，`dup2(fds[0], STDIN_FILENO)` 让文件描述符 0（标准输入）指向了管道的读端。因此，当子进程后续通过 `execvp` 执行 `sort` 命令时，`sort` 从标准输入读取的数据实际上来自管道——这就是 I/O 重定向的底层机制。

**文件描述符泄露与关闭**：注意子进程中 `close(fds[1])` 和父进程中 `close(fds[0])` 的重要性。`fork` 后，父子进程各自拥有管道两端文件描述符的副本。每个进程应关闭自己不需要的那一端，以避免文件描述符泄露和潜在的阻塞问题。例如，如果父进程不关闭 `fds[0]`（读端），那么即使父进程本身不读取，管道的读端仍有一个引用计数未归零，当子进程退出后某些边缘情况下可能导致资源无法释放。
:::

::: tip 重难点解析
**`dup2` 的内核实现与原子性**

理解 `dup2` 的内核行为对于理解 I/O 重定向至关重要。每个进程在内核中维护一个**文件描述符表**（fd table），其索引即为 fd 编号（0、1、2...）。每个 fd 表项指向**打开文件表**（open file table）中的一个条目，该条目记录了文件读写偏移量、访问模式、引用计数等信息。

`dup2(oldfd, newfd)` 在内核中执行以下步骤：
1. 如果 `newfd` 已经打开，先对其执行 `close(newfd)` ——释放旧引用，减少对应打开文件表项的引用计数
2. 将 fd 表中 `newfd` 位置的指针复制为 `oldfd` 位置指针的副本
3. **递增**该打开文件表项的引用计数（refcount）

```c
// dup2 的内核级伪代码（简化）
int dup2(int oldfd, int newfd) {
    if (oldfd == newfd) return newfd;
    if (fd_table[newfd] != NULL) {
        close(newfd);  // 步骤 1：关闭 newfd 的旧映射
    }
    fd_table[newfd] = fd_table[oldfd];     // 步骤 2：复制指针
    fd_table[newfd]->refcount++;           // 步骤 3：递增引用计数
    return newfd;
}
```

**为什么 `dup2` 是原子的而 `close` + `dup` 不是？**

这是一个关键的并发正确性问题。如果你手动执行：
```c
close(1);
dup(fds[1]);  // 两步操作，中间有"间隙"
```
在 `close(1)` 和 `dup(fds[1])` 之间，如果信号处理器被触发，它可能会在 fd 1 处于"空档"时执行 I/O 操作——本来要写入 stdout 的数据可能被写入另一个恰好分配到 fd 1 的文件中，造成数据损坏。而 `dup2(fds[1], 1)` 是一个**原子系统调用**——内核保证整个 close + dup 过程不可被信号中断。这就是为什么在信号处理器或任何需处理并发的程序中，都应该使用 `dup2` 而非 `close` + `dup` 的组合。

**引用计数与 EOF**：打开文件表项的引用计数决定了何时发送 EOF。以管道为例：管道的读端只有在**所有**写端 fd 都被关闭（refcount = 0）时才会返回 EOF。这解释了为什么 fork 后关闭不需要的管道端如此重要——如果任何一个进程持有写端 fd，读端就永远不会看到 EOF。
:::

::: tip 重难点解析
**管道（pipe）的内核实现深度解析**

管道不仅是"两个 fd 之间的通道"——它在内核中有着精确的数据结构和语义。

**内核管道缓冲区**：

管道在内核中的实现是一个**环形缓冲区**（circular buffer），默认大小为 64KB（Linux，可通过 `/proc/sys/fs/pipe-max-size` 查看系统上限）。可以通过 `fcntl` 调整缓冲区大小：

```c
int pipe_fds[2];
pipe(pipe_fds);
int pipe_size = fcntl(pipe_fds[0], F_GETPIPE_SZ);  // 查询大小，通常是 65536
fcntl(pipe_fds[0], F_SETPIPE_SZ, 1048576);          // 设置为 1MB
```

**原子写入保证（PIPE_BUF）**：

这是管道最重要的语义之一：当写入数据量 **≤ PIPE_BUF**（在 Linux 上为 4096 字节）时，`write()` 操作是**原子的**。这意味着如果进程 A 写入 256 字节，进程 B 写入 512 字节，读取端永远不会看到这两个写入的数据交错——要么完整读到 256 字节，要么完整读到 512 字节。

但如果写入超过 PIPE_BUF（例如写入 8192 字节），内核可能将一次 write 拆分为多次，而来自不同写者的数据可能在管道中交错。在 shell 管道中这通常不是问题，因为大多数单行输出远小于 PIPE_BUF。

**SIGPIPE 信号**：

当你向一个读端已经被关闭的管道写入数据时，内核向写进程发送 `SIGPIPE` 信号。默认行为是终止进程（这解释了为什么 `yes | head -n 1` 中 `yes` 会自动终止——`head` 读完一行就退出并关闭了管道的读端）。

```c
// 控制 SIGPIPE 行为
signal(SIGPIPE, SIG_IGN);  // 忽略 SIGPIPE
// 此后向断开的管道 write() 不会终止进程，
// 而是返回 -1 并设置 errno = EPIPE
```
:::

::: tip 重难点解析
**文件描述符泄露导致死锁的经典场景**

以下代码展示了一个典型的管道死锁 bug——忘记关闭不需要的管道端会导致读者永远等待 EOF：

```c
// BUGGY：父进程忘记关闭写端，导致死锁
int fds[2];
pipe(fds);
pid_t pid = fork();
if (pid == 0) {
    // 子进程：读取者
    close(fds[1]);  // 子进程关闭写端 ✓
    char buf[256];
    ssize_t n;
    while ((n = read(fds[0], buf, sizeof(buf))) > 0) {
        write(STDOUT_FILENO, buf, n);
    }
    // read() 返回 0 表示 EOF —— 但 EOF 永远不会到来！
    close(fds[0]);
    exit(0);
} else {
    // 父进程：写入者
    close(fds[0]);  // 父进程关闭读端 ✓
    write(fds[1], "hello", 5);
    // BUG：父进程忘记 close(fds[1])！
    // 父进程仍持有写端 → 写端引用计数 > 0 → 子进程 read() 永不返回 0
    waitpid(pid, NULL, 0);  // 永远不会执行到这里 — 死锁！
}
```

**引用计数视角分析**：
- 初始：写端 refcount = 2（父子各一），读端 refcount = 2
- 子进程 `close(fds[1])`：写端 refcount = 1（只剩父进程持有）
- 父进程写入后必须 `close(fds[1])`：写端 refcount = 0 → 内核释放写端 → 通知读端 → `read()` 返回 0（EOF）
- 如果遗忘：refcount 永不为 0 → 读端永远阻塞

**经验法则**：fork 之后，每个进程应立即关闭管道中自己不需要的那一端。子进程通常关闭写端（只读），父进程通常关闭读端（只写）。
:::

::: tip 重难点解析
**进程组（Process Groups）与会话（Sessions）——Shell 作业控制的底层机制**

当你按 Ctrl+Z 暂停程序，或用 `fg`/`bg` 控制前台/后台时，底层依赖的是进程组和会话机制。

**进程组（Process Group）**：

每个进程都属于一个进程组，由 `setpgid(pid, pgid)` 设置。进程组的典型用途是：shell 为每个管道（如 `ls | sort | wc`）创建一个新的进程组，包含该管道中所有进程。这样，当用户按 Ctrl+C 时，`SIGINT` 会发送给前台进程组中的**所有**进程，而不仅仅是管道中的某一个。

```c
// 创建新进程组的典型模式（在子进程中执行）
setpgid(0, 0);  // 参数 (0,0) 表示：创建新进程组，pgid = 自己的 pid
// 该进程成为新进程组的组长（process group leader）
```

**会话（Session）**：

会话是比进程组更高层的抽象。一个会话包含一个或多个进程组，其中最多一个可以是前台进程组。控制终端（controlling terminal）与控制它的会话关联。

**作业控制的具体流程**：

| 操作 | 内核行为 |
|------|---------|
| Ctrl+Z | 向前台进程组发送 `SIGTSTP` → 进程被暂停(stopped) |
| Ctrl+C | 向前台进程组发送 `SIGINT` → 默认终止进程 |
| `fg` 命令 | shell 调用 `tcsetpgrp()` 将该作业的进程组设为前台，然后发送 `SIGCONT` 恢复执行 |
| `bg` 命令 | shell 发送 `SIGCONT` 但不改变前台进程组，进程在后台恢复执行 |

```c
// fg 的简化实现
kill(-pgid, SIGCONT);              // 向整个进程组发送 SIGCONT（负 pid 表示进程组）
tcsetpgrp(STDIN_FILENO, pgid);     // 将该进程组设为前台进程组
```

**关键系统调用**：
- `setpgid(pid, pgid)` — 将进程 pid 移入进程组 pgid
- `getpgrp()` — 获取当前进程的进程组 id
- `tcsetpgrp(fd, pgid)` — 设置终端的控制进程组（前台进程组）
- `tcgetpgrp(fd)` — 获取终端的前台进程组

在 Assignment 3/4 的 shell (stsh) 实现中，你需要直接使用这些系统调用来实现作业控制——特别是 `fg` 和 `bg` 内置命令。
:::
