---
title: "非阻塞I/O：事件驱动编程与epoll模型"
description: "非阻塞I/O：事件驱动编程与epoll模型"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告
* 作业 8
    * 作业 8 今天周三发布，截止日期为下周四，12 月 7 日。
    * 非常慷慨的迟交政策。
    * 作业将自动评分，不进行代码审查，并在周日上午公布成绩。

* 今日讲课
    * 介绍非阻塞 I/O。
        * 讨论什么是慢速系统调用，以及为什么它们对需要调用它们的
          执行线程来说是个坏消息。
        * 讲解 `non-blocking-alphabet-client` 的实现。
        * 介绍 `OutboundFile` 抽象，它在内部依赖
          非阻塞 I/O 来联合（syndicate）文件的内容。
        * 在不担心 `OutboundFile` 实现细节的情况下，
          我们将依赖它们来实现一个非阻塞 Web 服务器，该服务器能够仅在一个执行线程中
          处理非常大量的客户端连接。
        * 深入讲解 `OutboundFile` 类的实现。

# 案例研究：`slow-alphabet-server`
* 考虑以下服务器实现：

    ```cpp
    static const string kAlphabet = "abcdefghijklmnopqrstuvwxyz";
    static const useconds_t kDelay = 100000; // 100000 microseconds is 100 ms is 0.1 seconds
    static void handleRequest(int client) {
      sockbuf sb(client);
      iosockstream ss(&sb);
      for (size_t i = 0; i < kAlphabet.size(); i++) {
        ss << kAlphabet[i] << flush;
        usleep(kDelay);
      }
    }

    static const unsigned short kSlowAlphabetServerPort = 41411;
    int main(int argc, char *argv[]) {
      int server = createServerSocket(kSlowAlphabetServerPort);
      ThreadPool pool(128);
      while (true) {
        int client = accept(server, NULL, NULL);
        pool.schedule([client]() { handleRequest(client); });
      }
      return 0;
    }
    ```

# 案例研究：`slow-alphabet-server`
  * 完整实现在[这里](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/slow-alphabet-server.cc)。
      * 操作方式类似于我们几节课前编写的顺序 `time-server-concurrent`。
      * 协议：
          * 等待传入连接，然后
          * 将处理该连接的责任委托给 `ThreadPool` 中的一个工作线程，然后
          * 让工作线程非常缓慢地在 2.6 秒内拼出英文字母表，然后
          * 关闭连接（在 `sockbuf` 析构函数被调用时发生）。
      * 这个服务器没有任何非阻塞的地方，但它故意缓慢，以模拟
        真正的服务器端计算可能需要的时间来生成完整的响应。
      * 许多服务器，像上面的那个，会向客户端推送部分响应。

::: tip 重难点解析
**slow-alphabet-server 的设计意图**：这个服务器故意在每个字符发送后 `usleep(100ms)`，总耗时 2.6 秒来发送 26 个字母。它的目的是模拟一个真实的慢速响应场景——比如服务器在生成结果时需要查询数据库、调用外部 API 或进行复杂计算。在这个场景下，每个客户端连接都由 ThreadPool 中的一个独立线程服务，这保证了其他客户端不会因为一个慢速请求而被阻塞。但这种"每连接一线程"的模式存在可扩展性问题：当连接数达到数千时，线程的创建、上下文切换和内存开销会变得不可接受。这正是后续非阻塞 I/O 方案要解决的问题。
:::

# 案例研究：`blocking-alphabet-client`
* 这里展示的是 `slow-alphabet-server` 的传统（即阻塞）客户端：

    ```cpp
    static const unsigned short kSlowAlphabetServerPort = 41411;
    int main(int argc, char *argv[]) {
      int client = createClientSocket("localhost", kSlowAlphabetServerPort);
      size_t numSuccessfulReads = 0;
      size_t numBytes = 0;
      while (true) {
        char ch;
        ssize_t count = read(client, &ch, 1);
        assert(count != -1); // simple sanity check, assume more robust in practice
        if (count == 0) break; // we are truly done
        numSuccessfulReads++;
        numBytes += count;
        cout << ch << flush;
      }
      close(client);

      cout << endl;
      cout << "Alphabet Length: " << numBytes << " bytes." << endl;
      cout << "Num reads: " << numSuccessfulReads << endl;
      return 0;
    }
    ```

# 案例研究：`blocking-alphabet-client`
  * 完整实现在[这里](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/blocking-alphabet-client.cc)。
      * 依赖 `createClientSocket` 返回的传统客户端 socket。
      * 传递给 `read` 的字符缓冲区大小为 1，因此合法返回值的范围被限制在 [-1, 1]。
      * 假设 `slow-alphabet-server` 正在运行，在同一台机器上运行的
        客户端将可靠地表现出以下行为：

    ```sh
    myth7> ./slow-alphabet-server &
    [1] 7516
    myth7> ./blocking-alphabet-client
    abcdefghijklmnopqrstuvwxyz
    Alphabet Length: 26 bytes.
    Num reads: 26
    myth7> time ./blocking-alphabet-client
    abcdefghijklmnopqrstuvwxyz
    Alphabet Length: 26 bytes.
    Num reads: 26
    0.000u 0.002s 0:02.60 0.0%      0+0k 0+8io 0pf+0w
    myth7> kill -KILL 7516
    [1]    Killed                        ./slow-alphabet-server
    ```

::: tip 重难点解析
**阻塞客户端的运行特征**：注意 `time` 命令的输出——总墙钟时间 2.6 秒，但 CPU 时间（用户 + 系统）几乎为 0。这 2.6 秒几乎全部花在等待服务器的数据到达上，客户端线程在 `read` 调用中阻塞（休眠），不消耗任何 CPU。这恰好说明阻塞 I/O 的线程在等待时对 CPU 的使用是高效的——但代价是它在这段时间内无法做任何其他事情。

**每次只读一个字节的设计**：这里 `read(client, &ch, 1)` 每次都只请求 1 个字节，导致 26 次 `read` 调用。在生产代码中你不会这么做——你通常会请求更大的缓冲区来减少系统调用开销。这里故意使用 1 字节缓冲区是为了更清晰地展示阻塞和非阻塞行为之间的差异。
:::

::: tip 重难点解析
**O_NONBLOCK 的内核实现 —— 设置这个标志后内核发生了什么？**

设置 `O_NONBLOCK` 标志（通过 `fcntl(fd, F_SETFL, flags | O_NONBLOCK)`）并不会改变文件描述符的数据通道本身，它改变的是内核在数据不可用时的行为决策。

**阻塞模式下的 `read` 系统调用路径**：

```
用户调用 read(fd, buf, size)
  -> 内核 sys_read()
    -> 检查 fd 对应 socket 的接收缓冲区
       |-- 有数据？ -> 拷贝数据到用户空间，返回字节数
       |-- 无数据？ -> 将当前进程状态设为 TASK_INTERRUPTIBLE
       |              将进程加入该 socket 的等待队列 (wait_queue)
       |              调用 schedule() 让出 CPU，切换到其他进程
       |              ═══ 进程在此休眠，不消耗 CPU ═══
       |              （当数据到达：NIC 中断 -> 协议栈处理 ->
       |               数据放入接收缓冲 -> 唤醒等待队列中的进程
       |               -> schedule() 再次调度它）
       |              -> 重新检查缓冲区，拷贝数据到用户空间，返回
       |-- 连接关闭且缓冲区空？ -> 返回 0 (EOF)
```

**非阻塞模式下的 `read` 系统调用路径**：

```
用户调用 read(fd, buf, size)
  -> 内核 sys_read()
    -> 检查 fd 对应 socket 的接收缓冲区
       |-- 有数据？ -> 拷贝数据到用户空间，返回字节数
       |-- 无数据但连接仍打开？ -> 不睡眠！立即返回 -1/EAGAIN
       |-- 连接关闭且缓冲区空？ -> 返回 0 (EOF)
```

**关键区别仅在一个判断**：当缓冲区为空时，阻塞模式走"将进程加入等待队列并调度出去"的路径；非阻塞模式走"直接返回 -1"的路径。`write` 系统调用的逻辑同理——当发送缓冲区满时，阻塞模式令进程休眠直到有空间，非阻塞模式返回 -1/EAGAIN。

**忙轮询 (Busy Polling) 的 CPU 浪费量化**：

回顾 `non-blocking-alphabet-client` 的数据：2.6 秒内约 1100 万次 `read` 调用，仅 26 次成功。每次失败的 `read` 都是一次完整的系统调用——从用户空间陷入内核（trap）、检查缓冲区、返回用户空间。每次约 150-300ns 的开销。计算：

```
CPU 浪费 = 11,268,964 次失败读 * 250ns/次 ≈ 2.8 秒 CPU 时间
```

这与 `time` 命令显示的用户 + 系统时间约 2.6 秒高度吻合。换句话说，非阻塞客户端将几乎所有 CPU 时间都花在了"空转"上。这就是为什么非阻塞 I/O 必须与 `epoll`/`select`/`kqueue` 等事件通知机制配合使用——只在数据就绪时才执行 I/O 操作。
:::

# 案例研究：`non-blocking-alphabet-client`
* 这里展示的是 `slow-alphabet-server` 的客户端，它依赖
  非阻塞 I/O：

    ```cpp
    static const unsigned short kSlowAlphabetServerPort = 41411;
    int main(int argc, char *argv[]) {
      int client = createClientSocket("localhost", kSlowAlphabetServerPort);
      setAsNonBlocking(client);

      size_t numReads = 0;
      size_t numSuccessfulReads = 0;
      size_t numUnsuccessfulReads = 0;
      size_t numBytes = 0;
      while (true) {
        char ch;
        ssize_t count = read(client, &ch, 1);
        numReads++;
        if (count == 0) break; // we are truly done
        if (count > 0) {
          numSuccessfulReads++;
          numBytes += count;
          cout << ch << flush;
        } else {
          assert(errno == EWOULDBLOCK || errno == EAGAIN);
          numUnsuccessfulReads++;
        }
      }
      close(client);

      cout << endl;
      cout << "Alphabet Length: " << numBytes << " bytes." << endl;
      cout << "Num reads: " << numReads << " (" << numSuccessfulReads << " successful, " << numUnsuccessfulReads << " unsuccessful)." << endl;
      return 0;
    }
    ```

# 案例研究：`non-blocking-alphabet-client`
  * 完整实现在[这里](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/non-blocking-alphabet-client.cc)。
      * 与第一个版本一样，依赖 `createClientSocket` 返回的传统客户端 socket，
        但立即将其转换为非阻塞模式。
          * 现在，上面使用的 `read` 无法阻塞。
              * 数据可用？期望 ch 被更新并返回值为 1。
              * 永远没有数据可用？期望返回值为 0。
              * 当前没有数据可用，但将来可能有？期望返回值为
                -1 且 `errno` 被设置为 `EAGAIN`。

# 案例研究：`non-blocking-alphabet-client`
  * 查看 `non-blocking-alphabet-client` 的输出：

    ```sh
    myth7> ./slow-alphabet-server &
    [1] 9801
    myth7> ./non-blocking-alphabet-client
    abcdefghijklmnopqrstuvwxyz
    Alphabet Length: 26 bytes.
    Num reads: 11394590 (26 successful, 11394563 unsuccessful).
    myth7> time ./non-blocking-alphabet-client
    abcdefghijklmnopqrstuvwxyz
    Alphabet Length: 26 bytes.
    Num reads: 11268991 (26 successful, 11268964 unsuccessful).
    0.399u 2.202s 0:02.60 99.6%     0+0k 0+0io 0pf+0w
    myth7> kill -KILL 9801
    myth7>
    [1]    Killed                        ./slow-alphabet-server
    myth7>
    ```

  * 看看有多少次对 `read` 的调用因为数据尚未准备好而放弃。
  * 合理的问题：为什么这样更好？为什么要依赖非阻塞 I/O，难道仅仅是因为我们可以这样做吗？
      * 这个问题将在后续示例中得到解答。

::: tip 重难点解析
**非阻塞客户端的惊人数据 —— 超过 1100 万次 `read` 调用！** 在 2.6 秒内，非阻塞客户端发起了约 1100 万次 `read` 调用，但其中只有 26 次成功读取到了数据，其余全部返回 -1/`EWOULDBLOCK`。这个数据揭示了一个关键问题：纯轮询（busy polling）的非阻塞 I/O 极其浪费 CPU。注意 `time` 命令的输出——非阻塞版本消耗了 2.6 秒的 CPU 时间（99.6% 的 CPU 利用率），而阻塞版本几乎不消耗 CPU。这说明：仅仅将 socket 设为非阻塞是不够的，你还需要一个高效的机制来"在正确的时机才去读取"——这就是后续课程中 `epoll` 等 I/O 多路复用技术存在的理由。

**关键认知**：非阻塞 I/O 的真正力量不在于"循环尝试读取"，而在于与事件通知机制（`epoll`、`kqueue`）结合使用——只在操作系统通知你有数据可读时才去读，从而避免无谓的 CPU 空转。
:::

::: warning 注意事项
**`EWOULDBLOCK` 与 `EAGAIN` 的区别**：在 Linux 下，`EWOULDBLOCK` 和 `EAGAIN` 被定义为相同的值，可以互换使用。但在其他 UNIX 系统（如 BSD）上，它们可能有不同的值。可移植代码应该同时检查两个值。
:::

::: tip 重难点解析
**非阻塞 I/O 的根本洞察 —— 单线程中的连接交错 (Connection Interleaving)**

非阻塞 I/O 的真正力量在于一个简单但深刻的观察：**在一个线程中，当连接 A 的数据未就绪时，不必阻塞等待 A，而可以将 CPU 用于服务连接 B、C、D...**

这是对阻塞 I/O 模型的根本性颠覆：

**阻塞模型**（每连接一线程）：
```
Thread 1: [read A]──阻塞──> [收到数据] [处理A] [write A]──阻塞──> [写完]
Thread 2: [read B]──阻塞──> [收到数据] [处理B] [write B]──阻塞──> [写完]
Thread 3: [read C]──阻塞──> [收到数据] [处理C] [write C]──阻塞──> [写完]
```
每个线程独立运行，但 90%+ 的时间花在等待网络。

**非阻塞模型**（单线程交错）：
```
Thread 1: [try read A: 无数据, 跳过] [try read B: 有数据!] [处理B]
          [try write B: 写了一半] [try read A: 仍无数据]
          [try read C: 有数据!] [处理C] [try write B: 写完剩余]
          [try read A: 有数据!] ...
```
单线程在一轮循环中推进所有连接的进度，没有人阻塞任何人。

**连接状态机 (Connection State Machine)**：

每个客户端连接可以建模为有限状态机：

```
                   +----------+
        accept --> | READING  | -- 收到完整请求后 --> +------------+
                   +----------+                      | PROCESSING |
                        ^                            +------------+
                        |                                  |
                        | (等待更多数据                   | (处理完成)
                        |  时保持状态)                     v
                        |                          +--------------+
                        +-- (还需要更多数据) ----   | WRITING      |
                                                   | (发送响应)   |
                                                   +--------------+
                                                         |
                                                         v
                                                    (关闭连接)
```

**关键实现**：每个连接维护一个结构体，包含：
- `int fd`：socket 文件描述符（设为非阻塞）
- `char buffer[N]`：该连接的读/写缓冲区
- `size_t buf_offset`：当前在缓冲区中的读写位置
- `enum State { READING, PROCESSING, WRITING, CLOSING } state`

在主循环中，根据连接的状态和 fd 的就绪情况推进状态机：
- READING 且 fd 可读 -> `read` 直到 EAGAIN，解析请求
- PROCESSING -> 生成响应内容（纯计算，快速完成）
- WRITING 且 fd 可写 -> `write` 直到 EAGAIN，更新写偏移
- 所有数据写完 -> 关闭连接并回收资源

这就是 **nginx** 和 **Node.js** 事件循环的核心设计模式——所有连接的处理被交错在一个线程中，通过状态机管理每个连接的进度。OutboundFile 是将 WRITING 状态提炼为独立对象的示例。
:::

# `OutboundFile` 类实现概述
* 完整实现
    * [完整实现](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/outbound-file.cc)
      包含大量的意大利面条式代码（spaghetti code）。
    * 特别是，真正的文件描述符和 socket 描述符在某些地方需要区别对待——
      尤其是在检测所有数据何时已经被刷新到接收端描述符（可能是本地文件、
      控制台或远程客户端机器）时，情况并不那么优雅。
    * 然而，我的实现分解得足够好，我认为许多方法——我将在课堂上展示的那些——
      很容易理解，并提供了一个清晰的叙述。至少，
      我一定会让你相信 `OutboundFile` 的实现对于刚完成 CS110 的人来说
      是可以理解的。

# `OutboundFile` 类实现概述
* 完整实现
    * 以下是 `OutboundFile` 类的精简接口文件。

    ```cpp
    class OutboundFile {
     public:
      OutboundFile();
      void initialize(const std::string& source, int sink);
      bool sendMoreData();

     private:
      int source, sink;
      static const size_t kBufferSize = 128;
      char buffer[kBufferSize];
      size_t numBytesAvailable;
      size_t numBytesSent;
      bool isSending;

      bool dataReadyToBeSent() const;
      void readMoreData();
      void writeMoreData();
      bool allDataFlushed();
    };
    ```

    * 这在之前的幻灯片中已经展示过，不过现在我在暴露
      私有数据成员。
        * `source` 和 `sink` 是绑定到
          数据源和数据接收方的描述符，两者都是非阻塞的。
        * `buffer` 是一个大小合理的字符数组，帮助
          将通过 `read` 调用从 `source` 获取的字节
          通过 `write` 调用传送到 `sink`。我们
          不应该对 `read` 和 `write`
          参与其中感到惊讶。
        * `numBytesAvailable` 存储驻留在 `buffer`
          中的有意义字符的数量。
        * `numBytesSent` 跟踪驻留在 `buffer` 中
          已写入接收方的字节数。
            * 当 `numBytesAvailable` 和 `numBytesSent`
              相等时，我们知道 `buffer` 实际上是空的，或许
              需要再次调用 `read`。
        * `isSending` 跟踪是否所有数据都已被从 `source` 拉取
          并推送到接收方 `sink`。

::: tip 重难点解析
**OutboundFile 的缓冲区设计 —— 生产者-消费者模型的变体**：`buffer` 在两个方向上都扮演着关键角色。从 `source` 读入数据时，它是"目的地"；向 `sink` 写出数据时，它是"来源"。`numBytesAvailable` 和 `numBytesSent` 这两个游标将 buffer 分为三部分：已发送的、可发送但尚未发送的、空闲空间。这种"滑动窗口"式的缓冲区管理是网络编程中最常见的模式之一。

**为什么需要 buffer？** 直接从一个非阻塞描述符读取然后立即写入另一个非阻塞描述符看似简单，但存在一个问题：读取和写入的速度可能不匹配。source 可能暂时无数据（`read` 返回 -1），而 sink 可能暂时不可写（`write` 返回 -1）。buffer 起到了解耦和缓冲的作用，使得 push 操作可以分步进行——先尽可能从 source 拉取到 buffer，再尽可能从 buffer 推送到 sink。
:::

# `OutboundFile` 类实现概述
* 完整实现
    * 以下是 `OutboundFile` 实现的足够部分，以清楚地说明
      它是如何工作的。

    ```cpp
    OutboundFile::OutboundFile() : isSending(false) {}

    void OutboundFile::initialize(const string& source, int sink) {
      this->source = open(source.c_str(), O_RDONLY | O_NONBLOCK);
      this->sink = sink;
      setAsNonBlocking(this->sink);
      numBytesAvailable = numBytesSent = 0;
      isSending = true;
    }
    ```

    * 构造函数和 `initialize` 的实现是完整的。
        * `source` 始终是绑定到某个本地文件的
          文件描述符。
            * 注意，文件以只读方式打开（`O_RDONLY`），并且
              描述符被配置为非阻塞（`O_NONBLOCK`）。
            * 出于我们上次讨论过的原因，`source` 是否为非阻塞
              并不是非常重要，因为它绑定到本地文件。但本着非阻塞示例的精神，
              将其设为非阻塞也无妨。我们只是不应该期望从 `read`
              调用返回太多（如果有的话）-1。
        * `sink` 被显式标记为非阻塞，因为我们不应该要求
          客户端提前将其转换为非阻塞。

# `OutboundFile` 类实现概述
* 完整实现
    * 以下是 `OutboundFile` 实现的足够部分，以清楚地说明
      它是如何工作的。

    ```cpp
    bool OutboundFile::sendMoreData() {
      if (!isSending) return !allDataFlushed();
      if (!dataReadyToBeSent()) {
        readMoreData();
        if (!dataReadyToBeSent()) return true;
      }
      writeMoreData();
      return true;
    }
    ```

    * `sendMoreData` 的实现是不完整的，但这足以让你
      理解完整的故事。（再次说明，完整实现在
      [这里](http://cs110.stanford.edu/winter-2017/examples/non-blocking-io/outbound-file.cc)）。
        * 回顾一下，当不再需要调用 `sendMoreData` 时，
          它返回 `false`，如果不确定则返回 `true`。
        * 第一行检测所有数据都已被从 `source` 读取并写入
          `sink` 的情况，并返回 `true`，
          除非它能够进一步确认写入 `sink` 的所有数据已经
          实际到达（即我们已确认它已被刷新到最终目的地）。
        * 第一次调用 `dataReadyToBeSent` 检查 `buffer`
          是否包含尚未推送出去的字符。如果没有，则尝试
          `readMoreData`。如果在读取更多数据后缓冲区仍然为空——即
          对 `read` 的调用导致了 -1/`EWOULDBLOCK` 对，那么我们
          返回 `true`，表明没有数据需要写入，不需要尝试写入，
          但稍后再来看看情况是否会改变。
        * 对 `writeMoreData` 的调用是将数据推送到 `sink` 的机会。
        * 最后我们返回 `true`，因为我们需要回来看看是否有更多数据
          可以读入，或者我们是否已经完成读取和写入并且也已经设法将所有内容刷新
          到最终目的地。

::: tip 重难点解析
**`sendMoreData` 的状态机逻辑**：这个函数的巧妙之处在于它将"应该继续尝试"的语义编码到了返回值中——返回 `true` 表示"可能还有工作要做，请稍后再调用我"，返回 `false` 表示"工作全部完成"。调用者（服务器主循环）只需要在一个循环中不断轮询各个 OutboundFile 是否返回 `true`，就可以逐个推进所有文件传输，而不会在任何单个传输上被阻塞。这是一种协作式多任务处理的模式。

**为什么本地文件读很少返回 -1/EWOULDBLOCK？** 因为磁盘 I/O 通常由内核缓冲（page cache）支持。当你从本地文件 `read` 时，如果数据已经在内核的页缓存中（可能来自之前的读取或预读），读取几乎是即时的。即使数据在磁盘上，内核也会在后台发起磁盘 I/O 并在数据就绪后返回结果——但从调用者的角度看，这个过程是被内核在内部处理的，不会像网络 socket 那样长期暴露"数据不可用"的状态。
:::

# 案例研究：`OutboundFile` 类
* `OutboundFile` 类被设计为读取本地文件并将其内容通过
  提供的描述符推送出去，并且在整个过程中永远不会阻塞。
    * 以下是接口文件的简化版本：

    ```cpp
    class OutboundFile {
     public:
      OutboundFile();
      void initialize(const std::string& source, int sink);
      bool sendMoreData();

     private:
      // implementation details omitted for the moment
    };
    ```

    * 构造函数只是默认构造一个 `OutboundFile` 类的实例。
      `initialize` 方法标识应将哪个本地文件用作数据源，以及
      数据应按原样写入的描述符。`sendMoreData` 方法尽可能多地将数据
      推送到提供的接收端，而不会阻塞。如果可能还有更多数据需要发送，它返回
      `true`，如果所有数据都已被完全推送出去，则返回 `false`。
      完整的接口文件（泄露了一些实现细节，因为你看到了私有部分）
      在[这里](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/outbound-file.h)。

# `OutboundFile` 的单元测试
* 这是我用来确保 `OutboundFile` 类正常工作的简单程序。
    * 它是一个简单的程序，将单元测试的源代码打印到标准输出。#meta
    * 代码副本在[这里](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/outbound-file-test.cc)。

    ```cpp
    /**
     * File: outbound-file-test.cc
     * ---------------------------
     * Demonstrates how one should use the OutboundFile class
     * and can be used to confirm that it works properly.
     */
    #include "outbound-file.h"
    int main(int argc, char *argv[]) {
      OutboundFile obf;
      obf.initialize("outbound-file-test.cc", STDOUT_FILENO);
      while (obf.sendMoreData()) {;}
      return 0;
    }
    ```

# 静态文件服务器
* 现在考虑以下服务器。
    * 这是一个实现非阻塞服务器的程序，它愉快地向客户端提供服务器代码本身的副本。
    * 程序的完整副本在[这里](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/expensive-server.cc)。
    * 以下是该程序的代码：

    ```cpp
    static const unsigned short kDefaultPort = 12345;
    static const string kFileToServe("expensive-server.cc");
    int main(int argc, char *argv[]) {
      int serverSocket = createServerSocket(kDefaultPort);
      if (serverSocket == kServerSocketFailure) {
        cerr << "Could not start server.  Port " << kDefaultPort << " is probably in use." << endl;
        return 0;
      }

      setAsNonBlocking(serverSocket);
      cout << "Static file server listening on port " << kDefaultPort << "." << endl;
      list<OutboundFile> outboundFiles;
      size_t numConnections = 0;
      size_t numActiveConnections = 0;

      while (true) {
        int clientSocket = accept(serverSocket, NULL, NULL);
        if (clientSocket == -1) {
          assert(errno == EWOULDBLOCK);
        } else {
          OutboundFile obf;
          obf.initialize(kFileToServe, clientSocket);
          outboundFiles.push_back(obf);
          cout << "Connection #" << ++numConnections << endl;
          cout << "Queue size: " << ++numActiveConnections << endl;
        }

        auto iter = outboundFiles.begin();
        while (iter != outboundFiles.end()) {
          if (iter->sendMoreData()) {
            ++iter;
          } else {
            iter = outboundFiles.erase(iter);
            cout << "Queue size: " << --numActiveConnections << endl;
          }
        }
      }
    }
    ```

    * `setAsNonBlocking` 的实现是 UNIX 的晦涩代码（gobbledygook），如下所示：

    ```cpp
    void setAsNonBlocking(int descriptor) {
      int flags = fcntl(descriptor, F_GETFL);
      if (flags == -1) flags = 0; // if first call to fcntl fails, just go with 0
      fcntl(descriptor, F_SETFL, flags | O_NONBLOCK); // preserve other set flags
    }
    ```

::: tip 重难点解析
**单线程事件循环 —— 非阻塞 I/O 的真正威力**：这个静态文件服务器是理解事件驱动编程的关键示例。注意它的主循环结构：

1. 尝试 `accept`（非阻塞）新连接，如果有则创建 `OutboundFile` 加入队列
2. 遍历所有活跃连接的 `OutboundFile`，调用 `sendMoreData()` 推进每个文件的传输
3. 如果某个连接传输完成（`sendMoreData()` 返回 `false`），从队列中移除
4. 无限循环

这一切都在单个线程中完成！与 ThreadPool 方案相比，这里的优势在于：(a) 没有线程创建/销毁的开销；(b) 没有线程间上下文切换的开销；(c) 没有锁和同步的复杂性。当你在作业 7 中构建 HTTP 代理时，ThreadPool 是合理的选择；但当连接数增长到数千甚至数万时，这种单线程事件循环的架构就显现出优势——这也是 nginx 和 Node.js 等高性能服务器的核心设计理念。

**`fcntl` 设置非阻塞标志的惯用法**：`setAsNonBlocking` 的实现展示了 UNIX 系统编程中的一个重要模式——先读取当前标志（`F_GETFL`），再用按位或加上新标志（`O_NONBLOCK`），然后写回（`F_SETFL`）。直接 `fcntl(descriptor, F_SETFL, O_NONBLOCK)` 会清除之前设置的所有其他标志（如 `O_APPEND`），因此必须遵循"读取-修改-写回"的模式。
:::
