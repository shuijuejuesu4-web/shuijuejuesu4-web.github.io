# 公告

* 本周和下周安排
    * 作业 5 截止日期为周三晚上 11:59
    * 作业 6 于周三发布，截止日期为 11 月 15 日（星期三）

# 公告（续）

* 今日议程
    * 通过完成 `ice-cream-parlor` 模拟来结束多线程部分。
    * 开始网络编程！
        * 回顾 `telnet`、端口、Google、RSS 新闻订阅以及从命令行和浏览器访问 Facebook Graph。
        * 用两种方式实现一个时间服务器。
            * 引入 socket（套接字）的概念，它是一个经过包装的描述符，允许双向通信（这意味着可以在同一个描述符上进行读写操作）。
            * 使用原始 socket 和 `write` 系统调用实现一个时间服务器。
            * 实现第二个时间服务器，在客户端 socket 之上叠加一个第三方 C++ 流，以便我们可以使用 C++ 的流语义，而不是原始的 `read` 和 `write` 系统调用。
            * 实现第三个时间服务器，类似于第二个，但它使用了多线程。
        * 展示一组高层次类比，概述普通函数调用、系统函数调用、通过管道（pipe）进行的进程间通信以及（最后）通过 socket 进行的主机间通信在本质上是相同的。
        * 如果时间允许，实现几个网络客户端应用程序。
    * 阅读材料：
        * 阅读 Saltzer 和 Kaashoek 教材的[第 4.1 节和第 4.2 节](http://www.sciencedirect.com/science/article/pii/B978012374957400013X)。这两节对客户端-服务器模型做了精彩的讨论。
        * 阅读 Bryant 和 O'Hallaron 教材的第 11 章全部内容（这是你教材的第三章）。

::: tip 重难点解析
**为什么网络编程自然地引出多线程？**：网络服务器面临的核心问题是：处理一个客户端请求可能耗时较长（涉及磁盘 I/O、数据库查询等），如果服务器顺序处理请求，那么在处理当前请求期间，其他客户端必须排队等待。多线程允许服务器在处理一个请求的同时，让另一个线程去接受新的连接——这正是并发编程在网络服务中成为标配的原因。CS111 中会进一步讨论事件驱动（如 `epoll`、`select`）与线程驱动两种网络服务架构的优劣。
:::

# 实现服务器

* 操作隐喻
    * 通俗地说，服务器端应用程序就像在电话旁（某个特定的分机号）等待，祈祷有人——任何人——会打来电话。
    * 正式地说，服务器端应用程序创建一个监听特定端口的服务器 socket。
        * 服务器 socket 是一个整数标识符，关联到一个本地 IP 地址（可以类比为电话号码）和端口（可以类比为电话分机号）。
        * 你也应该将端口号视为一个虚拟进程 ID，主机将其与服务器应用程序的实际 pid 关联起来。

::: tip 重难点解析
**IP 地址、端口与 Socket 的三层关系**：理解网络通信的关键在于把握这三个概念的类比：IP 地址像是公司的总机号码（找到哪台机器），端口号像是分机号码（找到哪个进程），socket 则是建立连接后的通话线路（实际的数据通道）。一个服务器进程可以绑定到特定的 IP 和端口（使用 `bind`），然后在该端口上监听（`listen`），当有客户端连接时（`connect`），服务器通过 `accept` 获得一个新的 socket，专门用于与该客户端通信。最初的监听 socket 则继续等待新的连接。

CS111 中会详细讨论操作系统如何管理这些 socket 描述符（属于文件描述符的一种），以及网络协议栈在操作系统内核中的层次结构。
:::

# [宝宝的第一个服务器](http://3.bp.blogspot.com/-XUovTA5ae_0/TY_4BOMOYdI/AAAAAAAAFPk/7VjP57Izab8/s1600/photo-22.JPG)

* 一个时间服务器
    * 完整的服务器代码（减去 `createServerSocket` [接口](http://www.stanford.edu/class/cs110/autumn-2017/examples/networking/server-socket.h)和[实现](http://www.stanford.edu/class/cs110/autumn-2017/examples/networking/server-socket.cc)）可以在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/networking/time-server-descriptors.cc)找到。
    * 我们的时间服务器接受所有传入的连接请求，并快速发布当前时间，甚至不听取客户端要说的任何话。

    ```cpp
    static const short kDefaultPort = 12345;
    static const int kWrongArgumentCount = 1;
    static const int kServerStartFailure = 2;
    int main(int argc, char *argv[]) {
      if (argc > 1) {
        cerr << "Usage: " << argv[0] << endl;
        return kWrongArgumentCount;
      }

      int serverSocket = createServerSocket(kDefaultPort);
      if (serverSocket == kServerSocketFailure) {
        cerr << "Error: Could not start server on port " << kDefaultPort << "." << endl;
        cerr << "Aborting... " << endl;
        return kServerStartFailure;
      }

      cout << "Server listening on port " << kDefaultPort << "." << endl;
      while (true) {
        int clientSocket = accept(serverSocket, NULL, NULL);
        publishTime(clientSocket);
      }

      return 0;
    }
    ```

# [宝宝的第一个服务器](http://3.bp.blogspot.com/-XUovTA5ae_0/TY_4BOMOYdI/AAAAAAAAFPk/7VjP57Izab8/s1600/photo-22.JPG)（续）

* `publishTime` 的实现是直观的。
    * 然而，实现本身并不是重点。严格来说，它是在生成动态内容——虽然无聊，但确实是动态的——并通过 socket 描述符将其发布回客户端。

    ```cpp
    static void publishTime(int clientSocket) {
      time_t rawtime;
      time(&rawtime);
      struct tm *ptm = gmtime(&rawtime);
      char timeString[128]; // more than big enough
      /* size_t len = */ strftime(timeString, sizeof(timeString), "%c\n", ptm);

      size_t numBytesWritten = 0, numBytesToWrite = strlen(timeString);
      while (numBytesWritten < numBytesToWrite) {
        numBytesWritten += write(clientSocket,
                                 timeString + numBytesWritten,
                                 numBytesToWrite - numBytesWritten);
      }
      close(clientSocket);
    }
    ```

    * 这里的前五行生成了需要发布的完整时间字符串。让这五行代表更一般意义上的服务器端计算，为了服务产生输出所需。这里是当前时间，但它也可以是一个静态 HTML 页面、一个 Google 搜索结果、一个 RSS XML 文档、一张图片或一个 Netflix 视频。
    * 剩余的行将时间字符串——我们称之为载荷（payload）——发布到客户端 socket，使用了我们之前见过的原始的低级 I/O。

::: tip 重难点解析
**`write` 的循环写入模式**：注意 `publishTime` 中使用 `while` 循环来调用 `write` 并非过度谨慎。对于普通文件描述符，`write` 通常一次就能写完所有数据，但 socket 描述符绑定到网络驱动程序，其内部缓冲区空间有限。如果数据量超过缓冲区剩余空间，`write` 只写入部分数据并返回实际写入的字节数，调用者负责继续写入剩余数据。这就是为什么需要 `while (numBytesWritten < numBytesToWrite)` 循环——它确保所有数据都被发送出去。CS111 的文件系统和 I/O 子系统部分会详细讨论缓冲区管理和部分写入的原因。
:::

::: tip 重难点解析
**socket() 系统调用的内核行为**

当调用 `int s = socket(AF_INET, SOCK_STREAM, 0)` 时，Linux 内核执行以下操作：

1. **分配 socket 结构体**：内核在内存中分配一个 `struct socket`，包含协议族、类型、状态等信息。

2. **分配文件描述符**：内核分配一个未使用的最小 fd 号，将 socket 与文件描述符表中的一个条目关联。这也是为什么你可以用 `close()` 来关闭 socket——它在内核中确实是一个特殊的"文件"。

3. **关联协议操作集**：`SOCK_STREAM` 告诉内核使用 TCP 协议。内核将 socket 的 ops 指针指向 TCP 协议的操作函数表（`tcp_prot`），其中包含 `tcp_sendmsg`、`tcp_recvmsg`、`tcp_close` 等函数指针。这种面向对象的设计使得同一接口可以支持不同协议（TCP、UDP、Unix domain socket）。

4. **分配发送和接收缓冲区**：内核为 socket 分配内核空间的发送缓冲区（sk_sndbuf）和接收缓冲区（sk_rcvbuf）。这些缓冲区的大小通常由 `/proc/sys/net/core/wmem_default` 和 `rmem_default` 决定（默认约 16KB-208KB，可动态调整到 `wmem_max`/`rmem_max`，通常约 4MB-8MB）。

5. **初始化 TCP 状态**：设置 socket 的 TCP 状态为 `TCP_CLOSE`（初始关闭状态）。

这就是为什么 socket() 调用返回一个整数——它实际上是一个 fd 索引，指向内核分配的一整套数据结构。CS111 虚拟文件系统（VFS）章节会深入讨论这种"一切皆文件"的设计。
:::

::: tip 重难点解析
**bind() 与客户端的隐式绑定**

服务器调用 `bind()` 将 socket 绑定到知名端口（well-known port），原因是客户端需要提前知道端口号才能连接。客户端通常不调用 `bind()`，原因有二：

1. **端口号无关紧要**：客户端只需要一个端口号来进行此连接，具体值不重要。内核在 `connect()` 时自动从临时端口范围（ephemeral port range，通常为 32768-60999，可通过 `/proc/sys/net/ipv4/ip_local_port_range` 查看）分配一个未使用的端口。

2. **避免端口冲突**：如果客户端显式绑定到一个固定端口，而该客户端程序同时运行多个实例，第二个实例的 `bind()` 会失败（端口已被占用）。内核的自动分配机制避免了这个问题。

特殊情况下客户端也需要 bind：某些协议要求客户端使用特定端口（如某些安全策略或 NAT 穿越需求），或者调试时需要固定源端口。
:::

::: tip 重难点解析
**listen() 的 backlog 参数与 TCP 三次握手**

`listen(sockfd, backlog)` 中的 `backlog` 参数控制的是**已完成连接队列**（又称 accept queue）的最大长度，但实际上内核维护两个队列：

1. **未完成连接队列（SYN Queue / Incomplete Connection Queue）**：存放已收到客户端 SYN、已回复 SYN-ACK、但尚未收到客户端 ACK 的连接（状态 SYN_RCVD）。此队列大小由系统的 `net.ipv4.tcp_max_syn_backlog` 控制。

2. **已完成连接队列（Accept Queue）**：存放已完成三次握手、等待应用程序调用 `accept()` 的连接（状态 ESTABLISHED）。`backlog` 参数控制的就是此队列的大小。

TCP 三次握手与这两个队列的关系：
- 客户端发送 SYN → 服务器收到，连接进入 SYN Queue（状态 SYN_RCVD），回复 SYN-ACK
- 客户端收到 SYN-ACK，发送 ACK → 服务器收到 ACK，连接从 SYN Queue 移到 Accept Queue（状态 ESTABLISHED）
- 应用程序调用 `accept()` → 从 Accept Queue 中取出一个连接

如果 Accept Queue 已满（达到 backlog 限制），新的连接完成三次握手后会被丢弃或忽略。这是 Linux 的默认行为。

**SYN Flood 攻击**：攻击者发送大量伪造源 IP 的 SYN 包，服务器为每个 SYN 分配 SYN Queue 槽位并回复 SYN-ACK，但不收到 ACK（因为源 IP 是伪造的，对应主机不会回复 ACK）。SYN Queue 很快被填满，正常用户的 SYN 被拒绝——服务器无法服务。这种 DoS 攻击利用了 TCP 协议设计的信任假设。防御手段包括 SYN cookies（不在 SYN Queue 中存储状态，而是将状态编码在 SYN-ACK 的 sequence number 中）。
:::

# 服务器端网络编程与线程

* 网络编程和线程就像花生酱和果冻一样搭配。
    * 服务器为了满足客户端请求可能需要做的工作可能非常耗时——如此耗时以至于顺序实现可能会干扰服务器接受未来请求的能力。
    * 一种解决方案：一旦 `accept` 返回一个 socket 描述符，立即生成一个子线程，将任何密集的、耗时的计算从主线程上剥离。子线程可以利用第二个处理器或第二个核心，而主线程可以快速进入下一个 `accept` 调用。

# 服务器端网络编程与线程（续）

* 网络编程和线程天生是一对。
    * 下面是同一个时间服务器示例，不同之处在于它决定使用线程来计算并将时间发布回客户端。

    ```cpp
    int main(int argc, char *argv[]) {
      if (argc > 1) {
        cerr << "Usage: " << argv[0] << endl;
        return kWrongArgumentCount;
      }

      int serverSocket = createServerSocket(kDefaultPort);
      if (serverSocket == kServerSocketFailure) {
        cerr << "Error: Could not start time server to listen to port " << kDefaultPort << "." << endl;
        cerr << "Aborting... " << endl;
        return kServerStartFailure;
      }

      cout << "Server listening on port " << kDefaultPort << "." << endl;
      ThreadPool pool(4);
      while (true) {
        int clientSocket = accept(serverSocket, NULL, NULL);
        pool.schedule([clientSocket] { publishTime(clientSocket); });
      }
      return 0;
    }
    ```

    * 注意，这里使用了 `ThreadPool` 将服务器端计算从主线程上剥离。这样，主线程可以轮转回来，更快地推进到其他 `accept` 请求。

# 服务器端网络编程与线程（续）

* 网络编程和线程就像 Pepper 和 Iron Man 一样搭配。
    * `publishTime` 的实现**必须**改变，如果它要成为线程安全的话。这个改变很简单但很重要：我们需要调用 `gmtime` 的可重入（reentrant）、线程安全版本，称为 `gmtime_r`。
        * `gmtime` 返回一个指向单一的、静态分配的时间信息记录的指针，所有对它的调用都使用这个记录。如果两个线程竞争性地调用它，那么两个线程都会竞相从共享的、静态分配的记录中获取时间信息。
        * 一种解决方案是使用 `mutex` 来确保一个线程可以不受竞争地调用 `gmtime` 并随后将数据从静态记录复制到本地字符缓冲区中。
        * 另一种解决方案——不需要加锁并且我认为更好的方案——使用同一函数的第二个版本，称为 `gmtime_r`。第二个可重入版本只需要传入一个专用返回值所需的空间。

    ```cpp
    static void publishTime(int clientSocket) {
      time_t rawtime;
      time(&rawtime);
      struct tm tm;
      gmtime_r(&rawtime, &tm);
      char timeString[128]; // more than big enough
      /* size_t len = */ strftime(timeString, sizeof(timeString), "%c", &tm);
      sockbuf sb(clientSocket); // destructor closes socket
      iosockstream ss(&sb);
      ss << timeString << endl;
    }
    ```

::: tip 重难点解析
**可重入性（Reentrancy）与线程安全**：`gmtime` 与 `gmtime_r` 的区别展示了 C 标准库中一类常见的线程安全问题。`gmtime` 将结果写入静态分配的缓冲区，导致所有调用共享同一个返回地址——在多线程环境下，一个线程的结果可能被另一个线程的调用覆盖。`gmtime_r`（后缀 `_r` 代表 reentrant）通过让调用者提供输出缓冲区来解决这个问题。这种"内部静态缓冲区"的线程不安全模式在 C 标准库中广泛存在（如 `strtok`、`gethostbyname` 等），CS111 中会讨论操作系统如何设计可重入的内核函数以确保多核并发安全。
:::

# [宝宝的第二个服务器](https://w8m8b4g9.ssl.hwcdn.net/media.easynews.com/social/EN_blog_ipodhack3.png)

* 之前示例中围绕裸露的 `write` 调用的 `while` 循环这一次实际上是必要的。
    * socket 描述符绑定到一个网络驱动程序，该驱动程序具有有限的空间。
    * 在实践中，常见的情况是 `write` 的返回值小于通过第三个参数提供的值。
    * 理想情况下，我们会依赖 C 流（例如 `FILE *`）或 C++ 流（例如 `iostream` 类层次结构）来叠加数据缓冲区并为我们管理围绕裸露的 `write` 调用的 `while` 循环。
    * 幸运的是，我们可以访问一个提供此功能的第三方库。我们将把它当作标准 C++ 的一部分来使用。

    ```cpp
    static void publishTime(int clientSocket) {
      time_t rawtime;
      time(&rawtime);
      struct tm *ptm = gmtime(&rawtime);
      char timeString[128]; // more than big enough
      /* size_t len = */ strftime(timeString, sizeof(timeString), "%c", ptm);
      sockbuf sb(clientSocket);
      iosockstream ss(&sb);
      ss << timeString << endl;
    } // the sockbuf closes the socket when it's destroyed
    ```

    * 我们依赖相同的 C 库函数来生成时间字符串。
    * 然而，这次我们将该字符串插入到一个 `iosockstream` 中，它本身叠加在客户端 socket 之上。
        * 注意，中间的 `sockbuf` 类会获取 socket 的所有权，并在其析构函数被调用时关闭它。

::: tip 重难点解析
**流抽象的价值**：`sockbuf` + `iosockstream` 的组合本质上是在原始的 socket 描述符上叠加了一层 C++ 流缓冲。这使得我们可以使用熟悉的 `<<` 操作符和格式化输出，而不需要手动管理 `write` 循环。这种"在低级 I/O 描述符上叠加高级流抽象"的模式在系统编程中非常普遍——它体现了分层设计的思想：每个层次负责不同的抽象级别，上层利用下层提供的服务，同时向上层隐藏实现细节。CS111 操作系统的分层架构也是同样的哲学。
:::

::: tip 重难点解析
**accept() 系统调用详解**

`int clientSocket = accept(serverSocket, NULL, NULL)` 返回一个**全新的**文件描述符，专用于与此特定客户端的通信。关键要点：

1. **监听 socket vs 已连接 socket**：`serverSocket` 是监听 socket（调用过 `listen()`），它的唯一职责是接受新连接——不能用于 read/write。`accept()` 返回的 `clientSocket` 是已连接 socket，用于与特定客户端通信——read/write/send/recv 都可以。两者是不同的内核对象。

2. **accept 的后两个参数**：`struct sockaddr *addr` 和 `socklen_t *addrlen`。如果传入非 NULL，内核会填入客户端的 IP 地址和端口号。这是一个 value-result 参数：`addrlen` 传入时是 addr 缓冲区的大小，传出时是实际填入的数据大小。示例中传入 NULL 表示不关心客户端地址。

3. **阻塞行为**：如果 Accept Queue 为空（没有已完成握手的连接），`accept()` 默认会阻塞，直到有客户端连接。这是"监听"的本质——服务器阻塞在 `accept()` 上，等待连接到达。

4. **fd 消耗**：每个 accept 调用消耗一个 fd。如果服务器不 close 已连接 socket，最终会耗尽进程的 fd 限制（`ulimit -n`，通常为 1024）。因此每个连接处理完后必须 close。

5. **并发 accept**：多个线程/进程可以同时对同一个监听 socket 调用 `accept()`。Linux 内核保证每个连接只会被一个 accept 调用返回——这在多进程（pre-fork）或多线程服务器架构中很有用。
:::

::: tip 重难点解析
**网络字节序（Network Byte Order）深度解析**

**为什么是大端序？** 历史原因。早期的网络协议（TCP/IP）主要在 IBM 大型机和 SPARC 工作站上开发，这些平台使用大端序。TCP/IP 标准化时（RFC 1700），大端序被指定为网络字节序并固化下来。即使今天 x86（小端序）占据主导，为了向后兼容，网络字节序仍然是大端序。

如果两台字节序相同的机器通信（如两台 x86），字节序转换实际上是浪费的——但协议要求必须做。好消息是，`htonl`/`htons` 在大端机器上是空操作（no-op），不产生任何指令。

**四个转换函数**：
- `htons(uint16_t)`：Host TO Network Short（16 位，用于端口号）
- `htonl(uint32_t)`：Host TO Network Long（32 位，用于 IPv4 地址）
- `ntohs(uint16_t)`：Network TO Host Short（反向转换）
- `ntohl(uint32_t)`：Network TO Host Long（反向转换）

**检查当前平台字节序**：
```c
uint32_t x = 1;
if (*(uint8_t *)&x == 1) {
    printf("little-endian\n");  // x86, ARM (大多数现代 CPU)
} else {
    printf("big-endian\n");     // SPARC, PowerPC (部分模式下)
}
```

原理：数值 1 的 32 位表示在大端序下是 `00 00 00 01`，小端序下是 `01 00 00 00`。检查第一个字节就知道字节序。
:::

::: tip 重难点解析
**gethostbyname 为何被弃用：getaddrinfo 的优势**

`gethostbyname` 在技术上是已弃用的函数，有三个致命缺陷：

1. **线程不安全**：返回的 `struct hostent *` 指向一个静态分配的缓冲区。多线程同时调用时，会互相覆盖结果。虽然有 `gethostbyname_r`（可重入版本），但更建议直接用 `getaddrinfo`。

2. **仅支持 IPv4**：`gethostbyname` 的 `h_addrtype` 和 `h_length` 字段设计严重偏向 IPv4（4 字节地址）。虽然理论上可以通过 `h_addrtype == AF_INET6` 支持 IPv6，但 API 设计导致兼容性很差。

3. **不支持服务名解析**：`gethostbyname` 只能将主机名解析为 IP 地址。它无法解析"服务名"（如 "http" → 80, "ssh" → 22）。`getaddrinfo` 同时支持主机名和服务名解析。

`getaddrinfo` 的设计解决了所有这些问题：
- 返回的是调用者提供的缓冲区（通过 `struct addrinfo **res` 参数），线程安全
- 协议无关（通过 `ai_family` 字段传达对 IPv4/IPv6 的支持）
- 同时解析节点名（主机）和服务名（端口）
- 返回一个链表，支持一个主机名对应多个地址（如同时有 IPv4 和 IPv6 地址）

CS110 教材为了教学简洁继续使用 `gethostbyname`，但在实际项目中应始终使用 `getaddrinfo`。
:::

# 我们的第一个网络客户端！

* 刻意简单！
    * 完整程序文件在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/networking/time-client.cc)
    * 协议——非正式地说是客户端和服务器双方必须遵循的一组规则，以便它们能够相互通信——很简单。
    * 这里的协议是……
        * 客户端连接（即"拨打"服务的电话，在特定的"分机号"上，并等待服务器"接听"）
        * 客户端什么都不说。
        * 服务器通过在它自己的连接端发布当前时间来说话，然后挂断。
        * 客户端接收发布的文本（按照协议理解为只有一行），将其输出到控制台，然后自己也挂断。

    ```cpp
    int main(int argc, char *argv[]) {
      int clientSocket = createClientSocket("myth7.stanford.edu", 12345);
      if (clientSocket == kClientSocketError) {
        cerr << "Time server could not be reached" << endl;
        cerr << "Aborting" << endl;
        return 1;
      }
      sockbuf sb(clientSocket);
      iosockstream ss(&sb);
      string timeline;
      getline(ss, timeline);
      cout << timeline << endl;
      return 0;
    }
    ```

    * 我们将在周一讨论 `createClientSocket` 的实现，但目前把它看作一个内置函数，它在客户端和在指定主机和端口号上运行的服务器之间建立一个双向管道，这是可以的。

::: tip 重难点解析
**协议（Protocol）的定义**：在计算机网络中，协议是通信双方遵循的一组约定——不是法律意义上的"协议"，而是"规则约定"。一个最简单的协议可能只包含：(1) 谁先说话？(2) 消息格式是什么？(3) 什么时候结束通信？时间服务器的协议极其简单：客户端连接后保持沉默，服务器发送一行文本后关闭连接。但正是这种简单性，让我们能聚焦于 socket 编程的核心机制，而不被协议解析的复杂性分散注意力。CS111 中会讨论更复杂的网络协议栈（TCP/IP）以及操作系统如何实现这些协议。
:::

# 模拟 `wget`

* `wget` 是一个命令行实用程序，给定其 URL，可以下载单个文档（HTML 文档、XML 文档、JPG 或任何其他文件）。
    * 不太关注错误检查和健壮性，我们可以写一些非常简单的东西来模拟 `wget` 的核心功能。
    * 完整程序在[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/networking/web-get.cc)。
    * 这将让我能够说明 HTTP 协议的最基本部分，这些部分对今天发布的作业至关重要。
    * 我将在课堂上并排运行 `/usr/bin/wget`（内置版本）和 `web-get`（我们即将编写的版本）。
    * 以下是 `main` 入口点和实现：

    ```cpp
    static const string kProtocolPrefix = "http://";
    static const string kDefaultPath = "/";
    static pair<string, string> parseURL(string url) {
      if (startsWith(url, kProtocolPrefix)) // in "string-utils.h"
        url = url.substr(kProtocolPrefix.size());
      size_t found = url.find('/');
      if (found == string::npos)
        return make_pair(url, kDefaultPath); // defined in <utility>
      string host = url.substr(0, found);
      string path = url.substr(found);
      return make_pair(host, path);
    }

    int main(int argc, char *argv[]) {
      if (argc != 2) {
        cerr << "Usage: " << argv[0] << " <url>" << endl;
        return kWrongArgumentCount;
      }
      pullContent(parseURL(argv[1]));
      return 0;
    }
    ```

    * `parseURL` 函数是对程序化分割主机-路径边界的一个示意。在实践中，可能还会支持更多协议（例如 `https`）。

# 模拟 `wget`（续）

* 当然，`pullContent` 函数需要处理网络部分。
    * 我们已经在 `time-client` 程序中使用过 `createClientSocket` 函数。同样，我们将在周三的课上讲解它的实现以及 `createServerSocket` 的实现。
    * 当然，还有 `web-get` 特定的工作要做：

    ```cpp
    static void pullContent(const pair<string, string>& components) {
      int clientSocket = createClientSocket(components.first, 80);
      // error checking omitted
      sockbuf sb(clientSocket);
      iosockstream ss(&sb);
      issueRequest(ss, components.first, components.second);
      skipHeader(ss);
      savePayload(ss, getFileName(components.second));
    }
    ```

    * `issueRequest`、`skipHeader` 和 `savePayload` 的实现将客户端-服务器对话细分为可管理的小块。
    * 从以上内容中最重要的收获是，在两个示例中我们第二次在一个 `sockbuf` 之上叠加了一个 `iosockstream`，而 `sockbuf` 本身又叠加在我们的双向 socket 描述符之上。
        * 应该庆幸我们有 `socket++` 库，因为没有它，我们将需要进行大量手动的字符数组操作，编码将不再有趣。
        * 关于 `sockbuf` 类的一个非常重要的信息：它的析构函数会关闭在构造时传递给它的文件描述符，所以我们不应该自己对 `clientSocket` 调用 `close`。

# 模拟 `wget`（续）

* 辅助函数的实现相当直观：
    * 以下是 `issueRequest` 的实现。注意，我手动构造了可以想象到的最简短的、两行的请求并通过网络发送给服务器。
    * 标准的 HTTP 协议实践是，每一行（包括标记请求结束的空白行）都以 CRLF（回车换行的缩写，即 `\r` 后跟 `\n`）结尾。
    * `flush` 调用是必需的，以确保所有字符数据被推送到网络上并可在另一端消费。

    ```cpp
    static void issueRequest(iosockstream& ss, const string& host, const string& path) {
      ss << "GET " << path << " HTTP/1.0\r\n";
      ss << "Host: " << host << "\r\n";
      ss << "\r\n";
      ss.flush();
    }
    ```

    * 在 `flush` 之后，客户端从发言模式切换到监听模式。
        * `iosockstream` 是可读可写的，因为支持它的 socket 描述符是双向的。
    * 我们读入所有 HTTP 响应头行，直到遇到一个空白行或只包含一个 `\r` 的行。
    * 空白行确实应该是 `\r\n`，但有些服务器比较随意，所以我们应该将 `\r` 视为可选的。（回想一下，`getline` 会消耗 `\n`，但会在行尾保留 `\r`）。

    ```cpp
    static void skipHeader(iosockstream& ss) {
      string line;
      do {
        getline(ss, line);
      } while (!line.empty() && line != "\r");
    }
    ```

    * 这是一个相当合理的情况，其中 `do/while` 循环是正确的惯用法。

::: tip 重难点解析
**HTTP 协议的基本结构**：HTTP 请求和响应都遵循"头部 + 可选载荷"的结构。一个最小化的 HTTP/1.0 GET 请求只需要三行：请求行（`GET /path HTTP/1.0`）、Host 头部、和标记头部结束的空白行。注意请求行的格式是 `<METHOD> <path> <VERSION>`（METHOD 是 GET/POST 等，VERSION 是 HTTP/1.0 或 HTTP/1.1）。HTTP/1.0 与 HTTP/1.1 的关键区别之一：HTTP/1.0 默认在响应完成后关闭连接，而 HTTP/1.1 默认保持连接（keep-alive）以便复用——这也是为什么我们使用 HTTP/1.0 来简化客户端代码。
:::

# 模拟 `wget`（续）

* 当然，还有载荷部分。
    * 响应头之后的所有内容以及那个空白行之后的内容都被视为载荷——那就是文件、JSON、HTML、图片、猫咪视频。
    * 每一个通过连接传过来的字节都应该被按顺序复制到本地副本。

    ```cpp
    static string getFileName(const string& path) {
      if (path.empty() || path[path.size() - 1] == '/') {
        return "index.html"; // not always correct, but not the point
      }

      size_t found = path.rfind('/');
      return path.substr(found + 1);
    }

    static const size_t kBufferSize = 1024; // just a random, large size
    static void savePayload(iosockstream& ss, const string& filename) {
      ofstream output(filename, ios::binary); // don't assume it's text
      size_t totalBytes = 0;
      while (!ss.fail()) {
        char buffer[kBufferSize] = {'\0'};
        ss.read(buffer, sizeof(buffer));
        totalBytes += ss.gcount();
        output.write(buffer, ss.gcount());
      }
      cout << "Total number of bytes fetched: " << totalBytes << endl;
    }
    ```

    * HTTP/1.0 协议规定，空白行之后的所有内容都是载荷，并且一旦服务器发布了载荷的每一个字节，它就关闭它那一端的连接。服务器端的关闭就是客户端的 EOF，我们写入我们读取到的所有内容。
        * 注意，我们以二进制模式打开了一个 `ofstream`，主要是为了防止 `ofstream` 对偶发地是换行符的字节字符做任何奇怪的处理。
        * `gcount` 返回最近一次 `read` 调用读取的字节数。（这是我在写这个示例之前不知道的）。
    * HTTP/1.1 协议允许连接保持打开，即使在初始载荷已经传输完毕之后。我在这里特意使用 HTTP/1.0 来避免这种情况，因为 HTTP/1.0 不允许这样做。

::: tip 重难点解析
**二进制模式与文本模式的区别**：在 C++ 中，`ofstream` 默认以文本模式打开文件，这可能导致在某些平台（特别是 Windows）上自动转换换行符（`\n` → `\r\n`）。对于 HTTP 载荷，它可能是二进制数据（如图片、压缩文件），任何字节的自动转换都会损坏数据。使用 `ios::binary` 标志确保文件流不做任何转换，将原始字节原封不动地写入磁盘。这也是网络编程中的一条通用原则：除非你确定数据是纯文本，否则始终以二进制模式处理。
:::
