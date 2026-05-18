# 公告
* 作业 6 和作业 7。
    * 作业 6 截止日期为周三晚上。
    * 作业 7 要求你构建自己的 HTTP 代理，将于周三发布，
      并在我们假期回来后的周三截止。

* 今日议程
    * 介绍 `struct hostent` 和 `sockaddr` 层次结构。
    * 深入讲解两个函数的实现，这两个函数自我们进入本季度的网络编程部分以来一直被视为理所当然：`createServerSocket`
      和 `createClientSocket`。
    * 通过最后一个网络编程示例，展示一种新颖的服务器端架构，
      这种架构与 Facebook 服务器生成新闻动态、LinkedIn 生成个人资料以及 Google 生成搜索结果的方式非常接近。

# Scrabble API
* 模仿 [Lexical Word Finder](http://www.lexicalwordfinder.com/)
    * 假设存在独立的 `scrabble-word-finder` 可执行文件。
    * 构成 `scrabble-word-finder` 的代码并不知道自己可能会
      被用于服务器，代码在[这里](http://cs110.stanford.edu/autumn-2017/examples/networking/scrabble-word-finder.cc)。
        * 使用直接的带剪枝的过程化递归实现。
        * 几乎没有为速度进行优化——没有缓存，只使用了最明显的剪枝策略。

* 我们希望实现一个服务器来共享 `scrabble-word-finder` 的功能。
    * 方法：允许 URL 指定字母组合。
    * `http://myth4.stanford.edu:13133/ieclxal` 应该返回所有
      可以由 `ieclxal` 组成的单词。

    ```js
    {
      time: 0.223399,
      cached: false,
      possibilities: [
        'ace',
          // several words omitted
        'lex',
        'lexica',
        'lexical',
        'li',
        'lice',
        'lie',
        'lilac',
        'xi'
      ]
    }
    ```

# Scrabble API
* 服务器相关的计算逻辑已经存在
    * 重新实现是不好的，重复造轮子是浪费时间和精力的。
    * `scrabble-word-finder` 作为可执行文件，已经以纯文本形式输出
      我们想要服务的核心内容，如下所示：

    ```sh
    myth4> ./scrabble-word-finder ieclxal
    ace
    lex
    lexica
    lexical
    li
    lice
    lie
    lilac
    xi
    myth4>
    ```

* 我们能编写一个利用现有功能并以不同方式包装的服务器吗？
    * 当然可以，否则我就不必提问了。
    * 利用作业 3 中的 `subprocess` 实现。

    ```cpp
    struct subprocess_t {
      pid_t pid;
      int supplyfd;
      int ingestfd;
    };
    subprocess_t subprocess(char *argv[],
                            bool supplyChildInput,
                            bool ingestChildOutput) throw (SubprocessException);
    ```

::: tip 重难点解析
**subprocess 与 IPC 的封装**：`subprocess` 函数是对 UNIX 进程间通信（IPC）管道模式的封装。它通过 `fork` + `execvp` 创建子进程，并建立两个单向管道：`supplyfd`（父进程向子进程写入数据）和 `ingestfd`（父进程从子进程读取数据）。这种模式是构建"以可执行文件为功能单元、以管道为通信媒介"的服务器架构的基础。在真实的生产环境中，类似的思想被广泛应用于微服务之间的通信。
:::

::: tip 重难点解析
**fork + exec + pipe 的内部机制与陷阱**

`subprocess` 函数的实现涉及三个关键步骤，每一步都有需要注意的细节：

**1. pipe() 系统调用的内核实现**

`pipe(int fds[2])` 创建一个内核缓冲区（在 Linux 上默认 64KB，可通过 `fcntl(fds[1], F_SETPIPE_SZ, size)` 调整），并返回两个文件描述符：`fds[0]` 用于读、`fds[1]` 用于写。内核将这个缓冲区实现为一个环形队列（circular buffer），维护读写指针。关键性质：
- 从 `fds[0]` `read`：如果缓冲区为空且没有 writer（所有写端已关闭），返回 0（EOF）；如果缓冲区为空但仍有 writer 打开，阻塞直到有数据或写端关闭
- 向 `fds[1]` `write`：如果缓冲区满，阻塞直到有空间（除非设置了 `O_NONBLOCK`，此时返回 `EAGAIN`）；如果所有读端已关闭，内核发送 `SIGPIPE` 信号（默认终止进程）

**2. fork() 后文件描述符的继承**

`fork()` 复制整个文件描述符表——子进程拥有与父进程相同的 fd 编号指向相同的内核打开文件描述（open file description）。这意味着父子进程共享管道端点的同一内核缓冲区。**关键清理步骤**：
- 父进程必须关闭不用的管道端：如果要向子进程写入，父进程关闭 `fds[0]`（读端）；如果要读取子进程输出，父进程关闭自己管道的写端
- 子进程在 `execvp` 之前必须：(a) `dup2(fds[1], STDOUT_FILENO)` 将管道写端重定向到标准输出（如果要把输出发给父进程）；(b) 关闭所有不再需要的管道文件描述符，避免子进程继承后意外保持管道打开，导致父进程的 `read` 永远不会收到 EOF

**3. 僵尸进程与 waitpid 的必要性**

每个子进程终止后，内核保留其进程描述符（PCB）和退出状态，直到父进程调用 `waitpid` 来"收割"（reap）它。如果父进程不调用 `waitpid`，子进程成为僵尸进程（zombie）——它不再消耗 CPU 或内存（除了 PCB），但占用 PID 资源。如果父进程先于子进程终止，子进程被 init（PID 1）收养，init 会自动收割。在实际的 `subprocess` 实现中，必须调用 `waitpid(sp.pid, &status, 0)` 来获取退出状态并回收资源。
:::

::: tip 重难点解析
**posix_spawn —— 为什么需要 fork+exec 的替代方案？**

`fork()` 在"分叉"时复制父进程的整个地址空间（页表条目），对于一个大型进程（例如有 10GB 地址空间的服务器），`fork` 的开销即使在写时复制（Copy-on-Write, COW）机制下也不可忽视——页表的复制本身就需要遍历和复制大量条目。更糟的是，`fork` 之后几乎总是立即调用 `exec` 来替换整个地址空间，这使得 `fork` 的复制工作变得毫无意义。

**posix_spawn 的设计**：`posix_spawn`（以及 Linux 的 `posix_spawnp`）将 `fork + exec` 合并为一个原子操作，在某些平台上可以避免地址空间的复制。特别是在 macOS 上，`fork` 因其庞大的地址空间结构（Mach VM 的复杂性）而特别昂贵，`posix_spawn` 是 Apple 推荐的替代方案。

**CS110 仍然使用 fork+exec 来教学的原因**：`fork` 是你理解进程模型的基础——它展示了 UNIX "通过复制来创建"的设计哲学。而且在实际中，`fork` + COW 对于中等大小的进程（如 CS110 作业中的服务器）性能完全可以接受。理解 `fork` 的工作原理是理解进程地址空间隔离的前提。
:::

::: tip 重难点解析
**子进程退出状态 (exit status) 的解读**

`waitpid(sp.pid, &status, 0)` 返回后，`status` 整数编码了子进程的终止信息。不能直接将其当作退出码使用，必须用宏来解码：

```c
if (WIFEXITED(status)) {
    // 正常退出: WEXITSTATUS(status) 获取退出码 (0-255)
    printf("exited with code %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    // 被信号杀死: WTERMSIG(status) 获取信号编号
    printf("killed by signal %d\n", WTERMSIG(status));
} else if (WIFSTOPPED(status)) {
    // 被信号暂停 (如 SIGSTOP): WSTOPSIG(status)
    printf("stopped by signal %d\n", WSTOPSIG(status));
}
```

在生产代码中，**必须检查子进程的退出状态**——如果 `scrabble-word-finder` 崩溃（segfault），父进程需要通过 `WIFSIGNALED` 检测到，而不能默默地将空输出当作"没有匹配的单词"返回给客户端。实际代码中还应该记录这些异常情况用于监控和调试。
:::

# Scrabble API
* 每个请求由 `ThreadPool` 中的专用线程处理
    * 线程例程使用 `subprocess` 将 `scrabble-word-finder`
      的纯文本输出编排为 JSON，并将该 JSON 作为 HTTP 响应的负载发布。
    * 以下是服务器端计算的核心：

    ```cpp
    static void publishScrabbleWords(int clientSocket) {
      sockbuf sb(clientSocket);
      iosockstream ss(&sb);
      string letters = getLetters(ss); // extracts tail of path from GET <path> <protocol>
      skipHeaders(ss); // skips everything else
      const char *command[] = {"./scrabble-word-finder", letters.c_str(), NULL};
      subprocess_t sp = subprocess(const_cast<char **>(command), false, true);
      vector<string> formableWords = pullFormableWords(sp.ingestfd);
      waitpid(sp.pid, NULL, 0);
      ostringstream payload;
      constructPayload(formableWords, payload); // posts JSON to payload
      sendResponse(ss, payload.str()); // publishes HTTP response to ss out of payload
    }
    ```

    * 辅助函数、缓存和错误检查在上面的代码中省略了，但已包含在
      [完整代码](http://cs110.stanford.edu/autumn-2017/examples/networking/scrabble-word-finder-server.cc) 中。

::: tip 重难点解析
**服务器架构模式 —— 包装现有命令行工具**：这个 Scrabble 服务器的设计体现了一种重要的工程哲学：不要重新发明轮子。当已有可执行文件能够完成核心计算时，服务器只需通过 `subprocess` 调用它、解析其输出、按 HTTP 协议封装后返回给客户端。`sockbuf` + `iosockstream` 将 socket 封装为 C++ 流对象，使得 socket 编程像操作 `cin`/`cout` 一样自然——这是 CS110 中"将一切抽象为文件"思想的集中体现。

**ThreadPool 的必要性**：如果直接在 `accept` 之后同步调用 `publishScrabbleWords`，那么每次只能服务一个客户端，其他客户端必须排队等待。ThreadPool 使得每个客户端连接可以独立处理，互不阻塞。但线程并非银弹——线程数量受限于系统资源，对于超高并发场景（C10K 问题），后续介绍的非阻塞 I/O 才是更优解。
:::

::: warning 注意事项
**`const_cast` 的使用**：代码中 `const_cast<char **>(command)` 是一个危险的操作。`execvp` 的原型接受 `char *const argv[]`，理论上不会修改 `argv` 的内容，但由于历史原因其签名未使用 `const`。在实际工程中，如果 `execvp` 的实现意外修改了传入的字符串，而你使用了 `const_cast` 来绕过编译检查，可能会导致未定义行为。更安全的做法是直接声明一个非 const 的数组。
:::

::: tip 重难点解析
**ThreadPool 的内部实现原理**

ThreadPool 是"每连接一线程"模型的执行引擎，其经典设计包含以下组件：

**1. 核心数据结构 —— 共享任务队列 + 工作线程数组**

```
                    +-----------+
    main thread --> | enqueue   | --> 共享任务队列 (queue<function<void()>>)
                    | schedule()|     受 mutex + condition_variable 保护
                    +-----------+
                          |
          +---------------+--------------+
          |               |              |
     Worker 1        Worker 2   ...  Worker N
     (等在 queue    (等在 queue       (等在 queue
      非空条件)      非空条件)         非空条件)
```

**2. 工作线程如何等待任务 —— 条件变量 (condition variable)**

每个工作线程运行如下核心循环：

```cpp
void worker() {
    while (true) {
        unique_lock<mutex> lock(queue_mutex);
        // 当 queue 为空且未设置关闭标志时，线程在此休眠
        cv.wait(lock, [this]{ return !tasks.empty() || shutdown; });
        if (shutdown && tasks.empty()) break;  // 优雅退出
        auto task = std::move(tasks.front());
        tasks.pop();
        lock.unlock();  // 提取任务后立即释放锁
        task();         // 执行任务（不持有锁！）
    }
}
```

关键点：`cv.wait(lock, predicate)` 等价于 `while(!predicate()) cv.wait(lock);`——这种双重检查模式防止虚假唤醒（spurious wakeup）。

**3. 有界队列 vs 无界队列的权衡**

- **无界队列**（unbounded）：任务可无限排队，永远不会因为队列满而拒绝任务。代价：如果生产者速度持续超过消费者，内存会耗尽（OOM）。
- **有界队列**（bounded, size=N）：队列满时生产者阻塞（back pressure）。优点：天然限流，保护系统不过载。缺点：可能导致死锁（如果任务间有依赖）。

CS110 的 ThreadPool 使用无界队列（`std::queue` + `std::condition_variable`），适合作业场景的负载。生产环境（如 Java 的 `ThreadPoolExecutor`）通常提供可配置的队列类型。

**4. 线程池大小如何确定**

- **CPU-bound 任务**（计算密集型）：线程数 = CPU 核心数。更多线程只能增加上下文切换开销，不会加速。经典公式：`N_threads = N_cores + 1`，额外的一个用于补偿偶发的 I/O 停顿。
- **I/O-bound 任务**（等待网络/磁盘）：线程数可以远大于核心数。合理估计：`N_threads = N_cores * (1 + wait_time / compute_time)`。如果一个任务 80% 时间在等待 I/O，20% 在计算，那么每个核心可以支持约 5 个线程而不至于过度竞争。

CS110 示例中 `ThreadPool pool(128)` 的 128 是故意超额配置的，因为 Scrabble 请求大部分时间花在等待 `scrabble-word-finder` 子进程完成，属于 I/O-bound 场景。

**5. 优雅关闭 (graceful shutdown) —— 两种经典模式**

- **毒丸模式 (poison pill)**：向队列中推送特殊标记任务（例如每个工作线程一个），当工作线程收到毒丸时退出循环。简单但需要知道工作线程数量。
- **原子标志模式**：设置 `atomic<bool> shutdown = true`，然后 `cv.notify_all()` 唤醒所有线程。工作线程醒来后检查标志后退出。这是 CS110 ThreadPool 使用的模式——更简洁可靠。

无论哪种模式，在析构 ThreadPool 前都必须调用 shutdown 并 `join` 所有工作线程，否则 `std::thread` 的析构会调用 `std::terminate`，导致进程异常退出。
:::
