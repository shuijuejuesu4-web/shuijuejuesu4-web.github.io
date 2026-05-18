# 公告

* 只有几条公告：
    * 作业 1 今晚截止。
    * 作业 1 将在周日下午前批改完毕，成绩报告随后不久通过邮件发出。
    * 作业 2 比作业 1 稍复杂一些，所以请尽早开始。
    * 讨论课本周开始。本周的讨论课讲义已经发布，讨论课答案将在本周末发布。

* 今日主题：
    * 继续我们关于 `fork` 的讨论，如何使用它以及它的工作原理。
        * 请参考本周一的幻灯片获取今天两个示例的内容，希望这些内容能顺利进行，然后我们可以继续讲解今天发布的新内容。
        * 如果一切顺利，你将在下周中之前理解标准 UNIX shell 的工作原理。很棒的东西，这个 `fork`。
        * 同步 `/usr/class/cs110/lecture-examples/autumn-2017` 以获取本周一和今天讲座的示例。

# 生成多个进程

* 回收多个子进程：
    * 当然，父进程可以多次调用 `fork`，只要它最终在子进程退出后回收它们。
    * 如果我们想在子进程退出时立即回收它们，而不关心它们 fork 的顺序，那么这可以做到：
    * 看看这个（代码在讲座示例文件夹 `processes/reap-as-they-exit.c` 中，也可以在[这里](http://cs110.stanford.edu/autumn-2017/examples/processes/reap-as-they-exit.c)查看）

    ```c
    int main(int argc, char *argv[]) {
      for (size_t i = 0; i < kNumChildren; i++) {
        pid_t pid = fork();
        exitIf(pid == -1, kForkFail, stderr, "Fork function failed.\n");
        if (pid == 0) exit(110 + i);
      }
    
      while (true) {
        int status;
        pid_t pid = waitpid(-1, &status, 0);
        if (pid == -1) break;
        if (WIFEXITED(status)) {
          printf("Child %d exited: status %d\n", pid, WEXITSTATUS(status));
        } else {
          printf("Child %d exited abnormally.\n", pid);
        }
      }
    
      exitUnless(errno == ECHILD, kWaitFail, stderr, "waitpid failed.\n");
      return 0;
    }
    ```

    * 请注意，我们向 `waitpid` 的第一个参数传入了 -1。这个 -1 表示我们想在**任何**子进程退出时收到通知。
    * 最终，所有子进程退出（正常或异常），waitpid **正确地**返回 -1 表示所有子进程都已结束。当 `waitpid` 返回 -1 时，它将一个名为 `errno` 的全局变量设置为常量 `ECHILD`，作为返回 -1 是因为所有子进程已终止的信号。有趣的是，这正是我们想要的"错误"。

::: tip 重难点解析
**waitpid 的两种回收模式**：

1. **按退出顺序回收（reap-as-they-exit）**：使用 `waitpid(-1, &status, 0)`，无论哪个子进程先退出，父进程都会立即回收它。这种方式响应最快，但回收顺序与 fork 顺序无关。

2. **按 fork 顺序回收（reap-in-fork-order）**：使用 `waitpid(children[i], &status, 0)`，按数组顺序依次等待每个特定的子进程。即使第 5 个子进程比第 1 个更早退出，父进程也会在第 1 个子进程上阻塞，直到它完成。

**errno 与 ECHILD**：`errno` 是一个全局变量，当系统调用失败时被设置以指示错误原因。`waitpid` 返回 -1 且 `errno == ECHILD` 时，表示"没有子进程可等待"——即所有子进程都已回收，这不是真正的错误，而是"任务完成"的信号。理解这一点对于正确编写回收循环至关重要。

**子进程 fork 时的注意事项**：在子进程中使用 `exit(110 + i)` 而不是 `return` 是一个好习惯——`return` 在某些情况下可能不会立即终止进程（如果有 `atexit` 处理函数等），而 `exit` 保证进程终止。在 fork 的场景中，你通常不希望子进程继续执行父进程的剩余代码。
:::

::: tip 重难点解析
**僵尸进程（Zombie Process）——为何不 wait 会耗尽系统资源**：

当一个子进程通过 `exit` 或信号终止时，它并非立刻从系统中消失。进程的 `task_struct` 和内核栈等大部分资源被释放，但**进程表条目（PID 槽位）和退出状态信息被保留**——此时子进程处于"僵尸"（zombie）状态（`ps` 显示为 `Z`）。

**僵尸存在的目的**：保留退出状态，等待父进程通过 `waitpid` 读取。只有父进程调用 `waitpid` 后，内核才彻底释放该 PID 槽位。

**如果不 wait 会发生什么？**
- 每个僵尸占用一个 PID 槽位。Linux 默认 PID 上限为 32768（`/proc/sys/kernel/pid_max`）。
- 如果父进程长时间运行且反复 fork 而不 wait，僵尸会累积直到 PID 耗尽，系统无法创建新进程。
- 这种行为称为"PID 泄漏"，类似于内存泄漏。

**使用 WNOHANG 的非阻塞回收循环**——这是编写事件驱动程序（如 shell、网络服务器）的标准模式：
```c
// 非阻塞回收所有已退出的子进程
while (true) {
    int status;
    pid_t pid = waitpid(-1, &status, WNOHANG);
    if (pid <= 0) break;  // 没有更多退出的子进程了
    // 处理 pid 的退出状态 ...
}
```
`WNOHANG` 让 `waitpid` 立即返回（不阻塞）：如果有已退出但未被回收的子进程，返回其 PID；如果没有，返回 0；如果出错（如没有子进程），返回 -1。

**孤儿进程**：如果父进程先于子进程退出，子进程变成"孤儿"。内核自动将孤儿进程的父 PID 重设为 1（init 进程），init 会定期调用 `waitpid(-1, ...)` 回收所有被它收养的僵尸。这是防止僵尸永久残留的兜底机制。

**SIGCHLD 信号**：当子进程状态改变（退出、被停止等）时，内核向父进程发送 `SIGCHLD` 信号。父进程可以捕获此信号并在信号处理器中调用 `waitpid`，从而实现异步的子进程回收——无需阻塞等待。
:::

# 生成多个进程，第二版

* 我们可以做同样的事情，但按照它们被 fork 的顺序回收。
    * 看看这个可爱的八胞胎程序（代码在讲座示例文件夹 `processes/reap-in-fork-order.c` 中，也可以在[这里](http://cs110.stanford.edu/autumn-2017/examples/processes/reap-in-fork-order.c)查看）

    ```c
    int main(int argc, char *argv[]) {
      pid_t children[kNumChildren];
      for (size_t i = 0; i < kNumChildren; i++) {
        children[i] = fork();
        exitIf(children[i] == -1, kForkFail, stderr, "Fork function failed.\n");
        if (children[i] == 0) exit(110 + i);
      }
    
      for (size_t i = 0; i < kNumChildren; i++) {
        int status;
        exitUnless(waitpid(children[i], &status, 0) == children[i],
                   kWaitFail, stderr, "Intentional wait on child %d failed.\n", children[i]);
        exitUnless(WIFEXITED(status) && WEXITSTATUS(status) == 110 + i,
                   kExitFail, stderr, "Correct child %d exited abnormally.\n");
      }
    
      return 0;
    }
    ```

     * 此版本以某种先产生先回收（让我们发明一个缩写：FSFR）的方式产生和回收子进程。
     * 当然，要理解子进程并不需要按 FSFR 顺序退出或以其他方式终止。理论上，第一个子进程可能最后完成，而回收循环可能在第一次迭代时就被阻塞，直到第一个子进程实际完成。但进程僵尸（如它们被称呼的那样）确实是按照它们被 fork 的顺序回收的。

# 进入 `execvp` 命令

* `execvp` 有效地重新启动一个进程，使其从头运行一个不同的程序。
    * `execvp` 有很多变体（`execle`、`execlp` 等。输入 `man` `execvp` 以查看所有变体）。
    * 以下是原型：

    ```c
    int execvp(const char *path, char *argv[]);
    ```

    * 参数和返回类型的含义如下：
        * `path` 标识应该被调用的可执行文件的名称。
        * `argv` 是应该被传递给新可执行文件的 `main` 函数的参数向量。
        * 通常，至少对于 CS110 的目的而言，`path` 和 `argv[0]` 最终是同一个字符串。
        * 如果 `execvp` 未能接管进程并在其中安装新进程镜像，`execvp` 将返回 -1。

::: tip 重难点解析
**exec 家族函数——"变脸"而不是"分身"**：`fork` 和 `exec` 常被混淆，但它们的职能完全不同：
- `fork`：创建当前进程的副本（分身术），新进程与原进程执行相同的代码。
- `exec`：用新的程序镜像替换当前进程的地址空间（变脸术），进程 ID 不变，但执行的代码完全不同。

**exec 家族的命名规则**：
- `exec` 后跟 `l`（list）：参数以可变参数列表方式传递，如 `execl("/bin/ls", "ls", "-l", NULL)`。
- `exec` 后跟 `v`（vector）：参数以 `char *argv[]` 数组方式传递，如 `execv("/bin/ls", args)`。
- `exec` 后跟 `p`（path）：在 `PATH` 环境变量中搜索可执行文件，无需提供完整路径。
- `exec` 后跟 `e`（environment）：允许指定新的环境变量数组。

所以 `execvp` = vector + path 搜索，是 shell 实现中最常用的变体。

**exec 只有失败才返回**：如果 `exec` 成功了，调用它的代码行之后的代码永远不会执行——因为整个进程镜像被替换了。`exec` 返回 -1 的唯一情况是失败（文件不存在、权限不足等）。这也是为什么 `exec` 调用之后通常紧跟一个错误处理或 `exit`。
:::

::: tip 重难点解析
**`execvp` 的内核实现——进程镜像的完全替换**：

`exec` 系列系统调用不会创建新进程——它在**当前进程**中用新的程序替换全部执行内容。PID 保持不变，但进程的"灵魂"被换掉了。以下是内核中 `sys_execve` 的核心流程：

1. **查找并加载可执行文件**：
   - 内核读取文件头（ELF magic bytes），验证格式。
   - 从 ELF header 中读取 program headers，找出 `.text`（代码段）、`.data`（数据段）、`.bss`（未初始化数据）等段的加载地址和大小。
   - 为每个段调用 `mmap` 将文件内容映射到虚拟地址空间（使用 demand paging——页面在实际访问时才从磁盘加载）。

2. **销毁旧的地址空间**：
   - 释放旧的 `mm_struct`（内存描述符），创建新的。
   - 这意味着所有旧的代码段、数据段、堆、栈全部消失。只有极少数资源跨 `exec` 保留（见下文）。

3. **设置新进程的执行上下文**：
   - 设置指令指针（IP/RIP）指向可执行文件的 `_start` 入口点（由 CRT/crt0 提供，不是 `main`）。
   - 设置栈——包含 `argc`、`argv`、`envp` 和辅助向量（aux vector）。
   - 将 `argv` 和 `envp` 字符串数组复制到新栈的顶部。

4. **动态链接器的角色（ld.so）**：
   对于动态链接的可执行文件，ELF 的 `PT_INTERP` 段指定了动态链接器的路径（如 `/lib64/ld-linux-x86-64.so.2`）。内核将动态链接器加载到内存中，并将入口点设置为动态链接器的 `_start`。动态链接器随后：
   - 加载所有依赖的共享库（`.so` 文件，如 libc.so.6）。
   - 执行重定位（relocation）——将符号引用绑定到实际地址。
   - 调用 `_init` 函数（共享库的构造函数）。
   - 最终跳转到程序的 `_start` → `__libc_start_main` → `main`。

5. **哪些资源跨 exec 保留？**

| 资源 | 保留/销毁 | 说明 |
|------|----------|------|
| PID, PPID, PGID | 保留 | 进程身份不变 |
| fd 表 | 保留（除非 FD_CLOEXEC） | 每个 fd 的 `O_CLOEXEC`/`FD_CLOEXEC` 标志决定 |
| 信号处理 | 恢复为 SIG_DFL | `SIG_IGN` 对 SIGCHLD 可能保留 |
| 内存映射 | 销毁 | 新的 mm_struct 替换旧的 |
| 环境变量 | 可替换 | `execve`/`execle` 传入新环境 |
| 当前目录 (cwd) | 保留 | 文件系统上下文保留 |
| 进程会计信息 | 保留 | rusage 等累加器保留 |

**关键设计：FD_CLOEXEC（close-on-exec）**
```c
// 方法 1：在 open 时设置
int fd = open("file.txt", O_RDONLY | O_CLOEXEC);

// 方法 2：用 fcntl 设置
fcntl(fd, F_SETFD, FD_CLOEXEC);
```
`exec` 成功后会遍历 fd 表，关闭所有设置了 close-on-exec 标志的 fd。这是防止子进程意外继承父进程不需要的 fd（如套接字、管道端点）的标准安全机制。在 shell 实现中，管道端点在子进程中必须关闭——如果在 fork 和 exec 之间手动 close 失败，`FD_CLOEXEC` 作为兜底保障。
:::

# 使用 `execvp`

* `mysystem` 的核心实现（以模拟 `system` 内置命令）
    * 这里我们展示了 `mysystem` 函数自己的实现，它通过调用 `"/bin/sh -c command"` 来执行提供的 `command`（保证是一个以 `'\0'` 结尾的 C 字符串），并最终在代理的 `command` 完成后返回。
    * 如果 `command` 的执行正常退出（无论是通过 `exit` 系统调用，还是通过 `main` 中的正常 return 语句），那么我们的 `mysystem` 实现应该返回该完全相同的退出状态。
    * 如果执行异常退出（例如发生段错误），那么我们将假设它是因为某个信号被忽略而中止的，并返回该信号编号的负值（例如 `SIGSEGV` 返回 -11）。
    * 以下是实现（在线[这里](http://cs110.stanford.edu/autumn-2017/examples/processes/mysystem.c)）

    ```c
    static int mysystem(const char *command) {
      pid_t pid = fork();
      if (pid == 0) {
        char *arguments[] = {"/bin/sh", "-c", (char *) command, NULL};
        execvp("/bin/sh", arguments);
        exitIf(true, kExecFailed, stderr, "execvp failed to invoke this: %s.\n", command);        
      }
    
      int status;
      waitpid(pid, &status, 0);
      if (WIFEXITED(status))
        return WEXITSTATUS(status);
      else
        return -WTERMSIG(status);
    }
    ```

    * 这是一个简单的单元测试，我将在讲座中运行以证明这确实有效：

    ```c
    static const size_t kMaxLine = 2048;
    int main(int argc, char *argv[]) {
      char buf[kMaxLine];
      while (true) {
        printf("> ");
        fgets(buf, kMaxLine, stdin);
        if (feof(stdin)) break;
        buf[strlen(buf) - 1] = '\0'; // overwrite '\n'
        printf("retcode = %d\n", mysystem(buf));
      }
    
      printf("\n");
      return 0;
    }
    ```

::: tip 重难点解析
**mysystem——fork/exec/waitpid 三部曲的完整示范**：这个函数堪称本课程前半部分的集大成者，它将之前学到的所有概念串联在一起：

1. **fork**：创建子进程。
2. **execvp**（在子进程中）：用 `/bin/sh -c <command>` 替换子进程的镜像。`sh -c` 的作用是让 shell 解析并执行任意命令字符串（包括管道、重定向等），因此 `mysystem("ls -la | grep foo")` 也能正常工作。
3. **waitpid**（在父进程中）：阻塞等待子进程完成。
4. **退出状态提取**：通过 `WIFEXITED`/`WEXITSTATUS` 宏检查子进程是否正常退出，或通过 `WTERMSIG` 提取异常终止的信号编号。

**信号编号转负值的约定**：`return -WTERMSIG(status)` 返回信号编号的负值——这是为了区分"正常退出码"（0-255）和"被信号终止"两种情况。因为正常退出码总是非负的，负值明确表示异常终止。例如，如果子进程因段错误（SIGSEGV = 11）被杀死，`mysystem` 返回 -11。

**理解整个调用链**：
```
用户输入 "ls -la"
  -> mysystem("ls -la")
    -> fork() 创建子进程
    -> 子进程: execvp("/bin/sh", ["/bin/sh", "-c", "ls -la", NULL])
      -> /bin/sh 解析并执行 "ls -la"
    -> 父进程: waitpid() 等待子进程结束
    -> 父进程: 返回退出码给调用者
```

这个模式也是所有 Unix shell（bash、zsh 等）实现的核心逻辑。在后续课程中，你还将看到如何在 fork 和 exec 之间操作文件描述符来实现管道（pipe）和重定向（redirection）。
:::

::: tip 重难点解析
**管道（Pipe）的内核实现——环形缓冲区与阻塞语义**：

管道是 Unix 中最古老、最简单的 IPC（进程间通信）机制。`pipe(int fds[2])` 创建一对文件描述符：`fds[0]` 为读端，`fds[1]` 为写端。数据从写端流入内核缓冲区，从读端流出——单向、先进先出（FIFO）。

**内核管道缓冲区**：Linux 内核为每个管道分配一个环形缓冲区（circular buffer），默认大小为 16 个页面（`PIPE_DEF_BUFFERS = 16` × `PAGE_SIZE = 4KB` = **64KB**）。可以通过 `fcntl(fd, F_SETPIPE_SZ, size)` 调整（上限由 `/proc/sys/fs/pipe-max-size` 控制，通常为 1MB）。

**环形缓冲区结构**（简化）：
```c
pipe_inode_info:
  +---------------------------+
  |  buf[] (16 个 struct pipe_buffer 页面指针)  |
  |  head = 3  (写位置索引)    |     head 指向下一个写入位置
  |  tail = 1  (读位置索引)    |     tail 指向下一个读取位置
  |  readers = 1              |     读者计数
  |  writers = 1              |     写者计数
  +---------------------------+
          |
          v (指向的环形页数组)
  [buf[0]] [buf[1]] [buf[2]] [buf[3]] ... [buf[15]]
     ↑         ↑
   (空)     (tail)                    (head)
                                      带数据的页
```

**阻塞行为详解**：

1. **从管道读取**（`read(fds[0], ...)`）：
   - 缓冲区非空：复制数据到用户空间，移动 tail 指针。返回实际读取的字节数。
   - 缓冲区为空且有活跃写者（`writers > 0`）：阻塞等待，直到有数据可读或信号中断。
   - 缓冲区为空且无活跃写者（`writers == 0`）：返回 0（EOF），表示所有写端已关闭。
   
2. **向管道写入**（`write(fds[1], ...)`）：
   - 缓冲区有空间：复制数据到环形缓冲区，移动 head 指针。返回实际写入的字节数。
   - 缓冲区满：阻塞等待，直到有空间可用。
   - 写入量 > 管道容量：数据分块写入，每次填满缓冲区后阻塞等待读者消费。
   - **原子性保证**：`write` 不超过 `PIPE_BUF`（POSIX 要求至少 512 字节，Linux 通常为 4096 字节）的数据量时，写入是原子的——不会与其他写者的数据交错。超过 `PIPE_BUF` 则可能交错。

3. **关闭读写端**：
   - 关闭写端 → `writers` 递减 → 当 `writers == 0` 时，读者看到的 `read` 返回 0（EOF）。
   - 关闭读端 → `readers` 递减 → 当 `readers == 0` 时，内核向写者发送 `SIGPIPE` 信号（默认行为是终止写进程），且 `write` 返回 -1 并设置 `errno` 为 `EPIPE`。

**shell 管道实现的标准四步模式**（以 `ls | wc` 为例）：
```c
int fds[2];
pipe(fds);                          // 1. 创建管道

pid_t pid1 = fork();
if (pid1 == 0) {                    // 2. 第一个子进程 (ls)
    close(fds[0]);                  // 关闭读端（不需要）
    dup2(fds[1], STDOUT_FILENO);    // 重定向 stdout → 管道写端
    close(fds[1]);                  // 关闭原 fds[1]
    execvp("ls", ls_args);
}

pid_t pid2 = fork();
if (pid2 == 0) {                    // 3. 第二个子进程 (wc)
    close(fds[1]);                  // 关闭写端（不需要）
    dup2(fds[0], STDIN_FILENO);     // 重定向 stdin → 管道读端
    close(fds[0]);                  // 关闭原 fds[0]
    execvp("wc", wc_args);
}

close(fds[0]);                      // 4. 父进程关闭管道两端
close(fds[1]);
waitpid(pid1, NULL, 0);             // 等待两个子进程
waitpid(pid2, NULL, 0);
```
关键细节：父进程必须在 fork 后关闭自己持有的管道端点，否则管道永远不会有 `writers == 0` 的时刻，读者的 `read` 永远不会返回 EOF，`wc` 会永远阻塞。
:::
