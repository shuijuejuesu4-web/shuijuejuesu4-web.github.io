---
title: "网络编程：HTTP协议与Web客户端-服务器模型"
description: "网络编程：HTTP协议与Web客户端-服务器模型"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告

* 作业 5 和作业 6
    * 作业 5 截止日期为今晚 11:59。
    * 作业 6 于今天发布，截止日期为 11 月 15 日（星期三）晚上 11:59。

* 今日议程
    * 讲解几个简单服务器的实现。
    * 讲解一个简单客户端应用程序的实现。
    * 讲解一个名为 `web-get` 的程序的实现，该程序模拟一个名为 `wget` 的 Linux 用户程序的功能。
        * 我们将说明 `web-get` 需要如何工作。
        * 我们将在课堂上实现它。
        * 我们将揭示 HTTP 引导的对话是多么精心设计，用于下载 HTML 文档、图片、视频或任何其他可以通过 HTTP 检索的内容。
    * 如果时间允许，我们将讨论用于实现 `createServerSocket` 和 `createClientSocket` 的 Unix 数据结构。

# 主机名解析

* `gethostbyname` 和 `gethostbyaddr`
    * Linux C 提供了用于将主机名（例如 "www.facebook.com"）转换为 IP 地址（例如 "31.13.75.17"）以及反向转换的指令。名为 `gethostbyname` 和 `gethostbyaddr` 的函数，虽然技术上讲已被弃用，但仍然非常普遍，你应该知道如何使用它们。事实上，你的 B&O 教材只提到了这些已弃用的函数：

    ```cpp
    struct hostent *gethostbyname(const char *name);
    struct hostent *gethostbyaddr(const char *addr, int len, int type);
    ```

    * 每个函数填充一个静态分配的 `struct hostent`，描述互联网上的某台主机。
        * `gethostbyname` 假设参数是一个主机名，例如 `www.google.com`。
        * `gethostbyaddr` 假设第一个参数是 IP 地址的二进制表示（例如不是字符串 "171.64.64.137"，而是一个字符数组的基地址，其中 ASCII 值 171、64、64 和 137 以**网络字节序**并排存放）。第二个参数对于 IPv4 地址（我们熟悉的那些）通常是 4，但对于 IPv6 地址可能会更大。第三个参数通常是 `AF_INET`（用于 IPv4 地址），但也可能是 `AF_INET6`（用于 IPv6 地址）或指定其他地址族。在 CS110 中我们将只使用 `AF_INET`。

* `struct hostent`
    * `struct hostent` 记录打包了关于互联网上某台特定主机的所有信息。

    ```cpp
    struct hostent {
      char *h_name;        // official name of host
      char **h_aliases;    // NULL-terminated list of aliases
      int h_addrtype;      // host address type, e.g. AF_INET
      int h_length;        // length of address (4 for IPv4 addresses)
      char **h_addr_list;  // NULL-terminated list of IP addresses
    }; // h_addr_list is really a struct in_addr ** when known to be IPv4 addresses

    struct in_addr {
      unsigned int s_addr  // stored in network byte order (big endian)
    };
    ```

::: tip 重难点解析
**网络字节序（Network Byte Order）**：网络字节序就是大端序（big-endian），即高位字节存放在低地址。而 x86 处理器使用小端序（little-endian，低位字节在低地址）。这意味着在网络上传输多字节数据（如 IP 地址、端口号）时，需要进行字节序转换，否则会导致接收方解码出错误的值。相关的转换函数包括 `htons`（host to network short，16 位）、`htonl`（host to network long，32 位）、`ntohs` 和 `ntohl`（反向转换）。CS111 中讨论网络协议栈时也会涉及字节序问题。

`gethostbyname` 和 `gethostbyaddr` 已被 `getaddrinfo` 和 `getnameinfo` 取代，主要原因是旧函数使用静态缓冲区（非线程安全），且不支持 IPv6。但在学习网络编程的基本概念时，旧 API 更简洁，更容易理解核心思想。
:::

# 解析 IP 地址

* `gethostbyname` 经常在网络应用程序中使用。
    * 这并不令人惊讶，因为用户更喜欢 "www.facebook.com" 这样的主机命名方案，但网络通信最终使用的是 "31.13.75.17" 的二进制表示。
    * 以下是[完整程序](http://www.stanford.edu/class/cs110/autumn-2017/examples/networking/resolve-hostname.cc)的核心，它查询用户输入的主机名并使用 `gethostbyname` 来显示关于它们的信息：

    ```cpp
    static void publishIPAddressInfo(const string& host) {
      struct hostent *he = gethostbyname(host.c_str());
      if (he == NULL) { // NULL return value means resolution attempt failed
        cout << host << " could not be resolved to an address." << endl;
        return;
      }

      cout << "Official name is \"" << he->h_name << "\"" << endl;
      cout << "IP Addresses: " << endl;
      struct in_addr **addressList = (struct in_addr **) he->h_addr_list;
      while (*addressList != NULL) {
        cout << "+ " << inet_ntoa(**addressList) << endl;
        addressList++;
      }
    }
    ```

    * 注意两个实现细节：
        * `h_addr_list` 的类型是 `char *` 数组，暗示它是一个 C 字符串数组，甚至是点分十进制 IP 地址。
            * 这是不正确的。`h_addr_list` 实际上是一个 `struct in_addr *` 的数组。
            * 每个 `in_addr` 是一个不必要的 `struct` 包装，包装了一个无符号整数，刚好足以存储 IP 地址的四个字节。这四个字节以网络字节序（即大端序）存储。
        * `inet_ntoa` 接受 `struct in_addr`s 并返回其点分十进制等价物（例如 "171.45.34.199"），作为静态分配的 C 字符串。
    * 技术上，`gethostbyname` 和 `gethostbyaddr` 是已弃用的，应该替换为 `getaddrinfo`、`getnameinfo` 和 `freeaddrinfo`。但教材依赖这些已弃用的版本，而且我仍然看到开发者在用它们，所以我也放心地使用它们。

::: tip 重难点解析
**C 语言中的类型双关（Type Punning）**：`h_addr_list` 被声明为 `char **` 但实际存储的是 `struct in_addr *`——这是经典 C 语言中"一个字段，多种解释"的类型双关做法。反过来看，这种做法不安全，需要显式强制转换 `(struct in_addr **)`，也容易让初学者感到困惑。现代网络编程应使用 `getaddrinfo`，它通过 `struct addrinfo` 提供了更清晰的类型结构。
:::

# `sockaddr` 层次结构

* 下面展示了三个数据结构，它们帮助我们建模 IP 地址/端口对。
    * 它们在这里：

    ```cpp
    struct sockaddr_in { // IPv4 Internet-style socket address record
      unsigned short sin_family; // protocol family for socket
      unsigned short sin_port;   // port number (in network byte order)
      struct in_addr sin_addr;   // IP address (in network byte order)
      unsigned char sin_zero[8]; // pad to sizeof(struct sockaddr)
    };

    struct sockaddr_in6 { // IPv6 Internet-style socket address record
      unsigned short sin6_family;  // protocol family for socket
      unsigned short sin6_port;    // port number (in network byte order)
      // more fields, total size is > sizeof(struct sockaddr_in)
    };

    struct sockaddr { // generic socket address record
      unsigned short sa_family; // protocol family for socket
      char sa_data[14];         // address data (and defines full size to be 16 bytes)
    };
    ```

    * `sockaddr_in` 结构体专门用于 IPv4 地址/端口对。
        * `sin_family` 字段应始终初始化为 `AF_INET`，这是一个常量，用于明确表示正在使用 IPv4 地址。
            * 如果一个专用于 IPv4 地址的记录需要存储一个常量来将其他字段中的信息标记为 IPv4，这看似冗余的话，那么请继续往下看。
        * `sin_port` 字段以网络字节序（即大端序）存储端口号。
        * `sin_addr` 字段将 IPv4 地址存储为一个压缩的大端序整数，正如你在 `gethostbyname` 和 `struct hostent` 中看到的那样。
        * `sin_zero` 字段通常被忽略（虽然通常设置为全零字节）。它的存在主要是为了将记录填充到 16 字节。
    * `sockaddr_in6` 结构体专门用于 IPv6 地址/端口对。
        * `sin6_family` 字段应始终设置为 `AF_INET6`。与 `sin_family` 字段一样，`sin6_family` 占据其所在记录的前两个字节。
        * `sin6_port` 字段保存一个两字节的、网络字节序的端口号，就像 `sin_port` 一样。
        * 我不列出其余字段，但你可以想象它们存储了 128 位 IPv6 地址的某种表示。
    * `struct sockaddr` 类型是 C 语言对抽象基类的最佳模仿。
        * 你很少甚至从不会声明 `struct sockaddr` 类型的变量，但许多系统调用接受 `struct sockaddr *` 类型的参数。
        * Linux 没有为 IPv4 地址和 IPv6 地址分别定义一套网络系统调用，而是为两者定义了一套统一的系统调用。
        * 如果一个系统调用接受 `struct sockaddr *` 类型的参数，它实际上期望的是 `struct sockaddr_in` 或 `struct sockaddr_in6` 的地址。该系统调用依赖前两个字节中的值——`sa_family` 字段——来确定实际的记录类型。

::: tip 重难点解析
**C 语言中的"继承"模拟**：`sockaddr` / `sockaddr_in` / `sockaddr_in6` 的设计展示了 C 语言中模拟面向对象多态的一种经典模式：(1) 所有"子类"结构体的第一个字段（`sa_family` / `sin_family` / `sin6_family`）类型和位置相同，用作"类型标识"；(2) 系统调用接收"基类"指针 `struct sockaddr *`，检查前两个字节确定实际类型后，再按对应的结构体布局解析后续字段；(3) 调用者负责传入正确的结构体类型和大小（通过 `sizeof` 参数）。这种模式虽然有效，但类型安全性较差——编译器无法检查你是否传入了正确的结构体。现代 C++ 代码中，C 风格的 socket 地址结构通常被封装在更安全的类中。
:::

::: tip 重难点解析
**sockaddr 层次结构的完整版图**

完整的 socket 地址结构层次如下：

```c
// 1. 通用基类——从不直接实例化
struct sockaddr {
    sa_family_t sa_family;    // 地址族 (AF_INET, AF_INET6, AF_UNIX)
    char        sa_data[14];  // 协议特定地址 (14 字节填充)
};
// 总大小: 16 字节

// 2. IPv4 地址结构 (netinet/in.h)
struct sockaddr_in {
    sa_family_t    sin_family;  // AF_INET (必须是这个值)
    in_port_t      sin_port;    // 端口号 (网络字节序, 16 位)
    struct in_addr sin_addr;    // IPv4 地址 (网络字节序, 32 位)
    unsigned char  sin_zero[8]; // 填充为 0, 使大小 = 16 字节
};
// 总大小: 16 字节 (与 sockaddr 一致)

// 3. IPv4 地址本身
struct in_addr {
    in_addr_t s_addr;  // uint32_t, 网络字节序
};

// 4. IPv6 地址结构
struct sockaddr_in6 {
    sa_family_t     sin6_family;   // AF_INET6
    in_port_t       sin6_port;     // 端口号 (网络字节序)
    uint32_t        sin6_flowinfo; // IPv6 流信息
    struct in6_addr sin6_addr;     // IPv6 地址 (128 位)
    uint32_t        sin6_scope_id; // 作用域 ID (如网络接口索引)
};
// 总大小: 28 字节 (大于 sockaddr)

// 5. 通用存储结构——分配地址存储的正确方式
struct sockaddr_storage {
    sa_family_t  ss_family;     // 地址族
    char         __ss_pad1[...];  // 内部填充, 实现相关
    // ... 足够容纳任何地址类型
};
// 总大小: 至少 128 字节 (足以容纳 IPv6 及未来扩展)
```

**关键设计要点**：

1. `sockaddr_in` 和 `sockaddr` 都是 16 字节——`sin_zero[8]` 的存在就是为了填充到相同大小,使 cast 安全（`sockaddr_in` 的前 16 字节布局和 `sockaddr` 一致）。

2. `sockaddr_in6` 是 28 字节——大于 `sockaddr`，这意味着将 `sockaddr_in6 *` cast 为 `sockaddr *` 后不能安全地通过 `sizeof(sockaddr)` 确定大小。这就是为什么 `accept()` 和 `getpeername()` 使用 `socklen_t *addrlen` 作为 value-result 参数——传入时指定缓冲区大小,传出时返回实际大小。

3. `sockaddr_storage` 是**最佳实践**——当你需要分配一个能容纳任意类型地址的缓冲区时（如 accept 的客户端地址），始终使用 `sockaddr_storage`，而不是 `sockaddr` 或 `sockaddr_in`。它保证有足够空间容纳任何地址类型（包括 Unix domain socket 路径）。

4. `sin_family` 与 `sa_family` 位于相同偏移量——这就是"继承"的关键：内核检查 `((struct sockaddr *)addr)->sa_family` 来判断实际类型,然后按对应结构体布局解析后续字段。
:::

::: tip 重难点解析
**getaddrinfo：现代 DNS 解析的正确方式**

`gethostbyname` 的现代替代品 `getaddrinfo` 签名如下：

```c
int getaddrinfo(const char *node,         // 主机名 (如 "www.google.com") 或 NULL
                const char *service,      // 服务名 (如 "http") 或端口号字符串 (如 "80")
                const struct addrinfo *hints,  // 输入：指定期望的地址类型
                struct addrinfo **res);        // 输出：结果链表

void freeaddrinfo(struct addrinfo *res);       // 释放结果链表
```

**struct addrinfo 结构**：
```c
struct addrinfo {
    int              ai_flags;      // 输入标志 (AI_PASSIVE, AI_CANONNAME 等)
    int              ai_family;     // AF_INET, AF_INET6, 或 AF_UNSPEC (自动)
    int              ai_socktype;   // SOCK_STREAM (TCP) 或 SOCK_DGRAM (UDP)
    int              ai_protocol;   // 0 表示自动选择(对 SOCK_STREAM 是 IPPROTO_TCP)
    socklen_t        ai_addrlen;    // 输出：ai_addr 的长度
    struct sockaddr *ai_addr;       // 输出：指针,指向实际地址结构
    char            *ai_canonname;  // 输出：规范主机名
    struct addrinfo *ai_next;       // 输出：链表中的下一个结果
};
```

**hints 参数的使用**（用于过滤结果）：

- `ai_family = AF_UNSPEC`：同时接受 IPv4 和 IPv6 地址（协议无关编程的关键）
- `ai_family = AF_INET`：只接受 IPv4 地址
- `ai_socktype = SOCK_STREAM`：只接受 TCP 协议的地址
- `ai_flags = AI_PASSIVE`：用于服务器——将 node 设为 NULL 并配合此标志,返回的地址适合 bind() (通常是 INADDR_ANY 或 IN6ADDR_ANY)
- `ai_flags = AI_CANONNAME`：要求返回规范主机名（填充 ai_canonname 字段）

**getaddrinfo 为何优于 gethostbyname**：

1. **线程安全**：结果存储在调用者通过 `res` 参数提供的指针中,不需要静态缓冲区。
2. **协议无关**：通过 hints 的 `ai_family` 可以同时请求 IPv4 和 IPv6 地址,单个返回值处理所有协议。
3. **服务名解析**：`service` 参数接受字符串 "http"、"ssh" 等(通过 `/etc/services` 解析)或直接传端口号字符串 "80"。
4. **返回链表**：一个主机名可能对应多个地址(如同时有 IPv4 和 IPv6),链表结构自然支持这种情况。

使用示例（客户端）：
```c
struct addrinfo hints = {0};
hints.ai_family = AF_UNSPEC;     // IPv4 或 IPv6 都可以
hints.ai_socktype = SOCK_STREAM; // TCP

struct addrinfo *result;
int ret = getaddrinfo("www.google.com", "80", &hints, &result);
if (ret != 0) {
    fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(ret));
    return -1;
}

// 遍历结果链表,尝试 connect
struct addrinfo *rp;
int sfd;
for (rp = result; rp != NULL; rp = rp->ai_next) {
    sfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    if (sfd == -1) continue;
    if (connect(sfd, rp->ai_addr, rp->ai_addrlen) != -1)
        break;  // 成功连接
    close(sfd);
}
freeaddrinfo(result);  // 必须释放！
```

CS111 会讨论 DNS 解析在内核网络栈中的完整流程（从 `/etc/hosts` → DNS 查询 → 缓存分层查找）。
:::

# 实现 `createClientSocket`

* 我们依赖 `createClientSocket` 来建立与在指定主机名和端口上运行的服务器的连接。
    * 以下是建立与目标服务器/端口对连接的代码：

    ```cpp
    static const int kClientSocketError = -1;
    int createClientSocket(const string& host, unsigned short port) {
      struct hostent *he = gethostbyname(host.c_str());
      if (he == NULL) return kClientSocketError;

      int s = socket(AF_INET, SOCK_STREAM, 0);
      if (s < 0) return kClientSocketError;

      struct sockaddr_in serverAddress;
      memset(&serverAddress, 0, sizeof(serverAddress));
      serverAddress.sin_family = AF_INET;
      serverAddress.sin_port = htons(port);
      serverAddress.sin_addr.s_addr = ((struct in_addr *)he->h_addr)->s_addr;

      if (connect(s, (struct sockaddr *) &serverAddress,
                  sizeof(serverAddress)) == 0) return s;
      close(s);
      return kClientSocketError;
    }
    ```

    * 最终，`s` 是客户端的 socket 描述符，可用于管理与在远程主机/端口对上运行的服务的双向对话。
    * `connect` 系统调用如何将 socket 描述符与主机/端口对关联起来，显然存在一些神秘之处，但因为这是系统级服务（系统调用本来就是如此，不是吗？），我们别无选择，只能假设它能正常工作。

::: tip 重难点解析
**客户端 Socket 创建的四个步骤**：
1. `gethostbyname()`：DNS 解析，将主机名转换为 IP 地址。
2. `socket(AF_INET, SOCK_STREAM, 0)`：创建一个 TCP socket。`AF_INET` 指定 IPv4 地址族，`SOCK_STREAM` 指定可靠的字节流传输（即 TCP），第三个参数 0 表示自动选择协议（对 `SOCK_STREAM` 来说就是 TCP）。
3. 填充 `sockaddr_in` 结构体：设置地址族、端口（通过 `htons` 转换为网络字节序）、以及目标 IP 地址。
4. `connect()`：向操作系统发起 TCP 三次握手（SYN → SYN-ACK → ACK），建立与服务器的连接。

CS111 会从操作系统内核的角度深入讨论 TCP 协议栈的实现，包括拥塞控制、流量控制和连接状态机。
:::

# 实现 `createServerSocket`

* 代码很密集，但对我们来说完全可以理解。
    * 假设一个服务器准备在其任意 IP 地址上监听指定端口，以下函数返回一个正确配置的服务器 socket：

    ```cpp
    static const int kServerSocketFailure = -1; // sentinel for no valid socket
    static const int kReuseAddresses = 1;   // 1 means true here
    static const int kDefaultBacklog = 128; // allow 128 clients to queue up before they are "accept"ed, drop/ignore 129th
    int createServerSocket(unsigned short port) {
      int serverSocket = socket(AF_INET, SOCK_STREAM, 0);
      if (serverSocket < 0) return kServerSocketFailure;
      if (setsockopt(serverSocket, SOL_SOCKET, SO_REUSEADDR,
                     &kReuseAddresses, sizeof(int)) < 0) {
        close(serverSocket);
        return kServerSocketFailure;
      } // setsockopt used here so port becomes available even is server crashes and reboots

      struct sockaddr_in serverAddress; // IPv4-style socket address
      memset(&serverAddress, 0, sizeof(serverAddress));
      serverAddress.sin_family = AF_INET; // sin_family field used to self-identify sockaddr type
      serverAddress.sin_addr.s_addr = htonl(INADDR_ANY);
      serverAddress.sin_port = htons(port);

      if (bind(serverSocket, (struct sockaddr *) &serverAddress, sizeof(struct sockaddr_in)) == 0 &&
          listen(serverSocket, kDefaultBacklog) == 0) return serverSocket;

      close(serverSocket);
      return kServerSocketFailure;
    }
    ```

    * 注意，`createServerSocket` 返回一个我们监听传入连接的 socket。
        * 它被归类为描述符，所以当我们用完它时需要将其关闭。
        * 它也会像任何描述符一样在 `fork` 边界被克隆。
        * 然而，服务器 socket 不兼容 `read` 和 `write` 系统调用。它唯一兼容的"读取"导向系统调用是 `accept`。

::: tip 重难点解析
**服务器 Socket 创建的六个步骤**：
1. `socket()`：创建 socket 描述符。
2. `setsockopt(SO_REUSEADDR)`：允许重用地址。当服务器崩溃重启后，如果不设置此选项，端口可能仍处于 `TIME_WAIT` 状态（通常持续 30-120 秒），导致 `bind` 失败。`SO_REUSEADDR` 解决了这个痛点。
3. `bind()`：将 socket 绑定到指定的 IP 地址（`INADDR_ANY` 表示绑定到本机所有网络接口）和端口。
4. `listen()`：将 socket 标记为被动模式（监听 socket），`kDefaultBacklog` 参数指定内核中等待 `accept` 的连接队列的最大长度。
5. `accept()`（在后续的 while 循环中调用）：从已完成 TCP 握手的连接队列中取出一个连接，返回一个新的 socket 描述符，用于与特定客户端通信。
6. `close()`：使用完毕后关闭 socket。

注意区分两个 socket：监听 socket（用于接受新连接）和已连接 socket（用于与特定客户端通信）。前者只调用 `accept`，后者才调用 `read`/`write` 进行数据交互。这与管道（pipe）的"一端读、一端写"有所不同——socket 是双向的，可以同时读写。CS111 中会详细讨论 socket 在内核中的数据结构以及网络协议栈的工作流程。
:::

::: warning 注意事项
**套接字编程中的常见陷阱**：
1. **忘记字节序转换**：端口号和 IP 地址必须使用 `htons`/`htonl` 转换为网络字节序，否则在 little-endian 机器上会产生错误值。
2. **不检查返回值**：几乎每个 socket 系统调用（`socket`、`bind`、`listen`、`accept`、`connect`）都可能失败，必须检查返回值。
3. **文件描述符泄漏**：如果在错误处理路径中不关闭已获取的 socket 描述符，会导致描述符泄漏，长期运行的服务器最终会耗尽文件描述符。
4. **端口号冲突**：知名端口（0-1023）需要 root 权限才能绑定。CS110 中请使用 1024 以上的端口号。
5. **监听 socket 与已连接 socket 混淆**：监听 socket 不能用于数据读写，已连接 socket 不能用于 `accept`。
:::
