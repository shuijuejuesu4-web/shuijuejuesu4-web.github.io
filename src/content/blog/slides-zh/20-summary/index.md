---
title: "非阻塞I/O进阶：事件循环与高并发服务器"
description: "非阻塞I/O进阶：事件循环与高并发服务器"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# 公告
* 重要日期
    * 最终作业 8 截止日期为周四晚上 11:59
        * 如果你在周四提交，你的分数将乘以 1.08。
        * 如果你在周五晚上前提交，则没有惩罚，但也没有奖励。
        * 如果你在周六晚上前提交，则你的成绩上限为 90%。
        * 你不能在周六晚上 11:59 之后提交作业 8。
    * 期末考试将于 12 月 13 日（周三）下午 3:30 在两个地点举行
        * 你姓的首字母决定了你应该在哪里参加考试：
            * 姓以 A、B、C、...... 或 K 开头的：Skilling Auditorium
            * 姓以 L、M、N、...... 或 Z 开头的：Gates B01
        * 将深入涵盖第 1 讲至第 18 讲（包括我的系统原则讲座）的所有内容，并
          期望对我在上周三开始并在上周一和今天讲授的非阻塞 I/O 内容有表面理解。
        * 考试为闭卷、闭笔记、闭电脑，但你可以携带并参考两张 8.5" x 11"
          的纸张，正反两面共四页，可以尽可能多地填满信息。

# 公告
* 今日讲课
    * 非阻塞 I/O 和事件驱动编程（`epoll`、
      `kqueue` 和 `libev`/`libuv` 软件包）、
      跨语言编译、[Tornado](http://www.tornadoweb.org) Web 服务器、
      [node.js](http://nodejs.org/) 和 Google 的 [V8](https://developers.google.com/v8/) 引擎。
        * 讨论 `epoll` 函数套件：`epoll_create`、`epoll_ctl` 和
          `epoll_wait`。
        * 讨论边缘触发（edge-triggered）和水平触发（level-triggered）事件之间的区别。
        * 使用非阻塞 I/O 在单个进程和单个执行线程中实现一个高效利用 CPU 的
          事件驱动 HTML 服务器。

# I/O 事件驱动编程
* 介绍 `epoll` I/O 通知工具！
    * `epoll` 是一套 Linux 例程，帮助非阻塞服务器
      在知道有工作要做之前让出处理器，直到它知道有一个或多个
      打开的客户端连接需要处理。
    * `epoll` 套件中有三个函数，我将在课堂上简要介绍，
      如果不是今天，那就是周三。
        * `epoll_create`，创建所谓的监视集（watch set），
          它本身是一组我们想要监视的文件描述符。返回值本身是一个
          用于标识监视集的文件描述符。因为它是一个文件描述符，
          监视集可以包含其他监视集。#深度
        * `epoll_ctl` 是一个控制函数，允许我们向监视集添加描述符、
          从监视集中移除描述符，以及重新配置监视集中已有的文件描述符。
        * `epoll_wait` 等待 I/O 事件，阻塞调用线程直到检测到
          一个或多个事件。
    * 我将在课堂上使用的示例太大，无法放在幻灯片中，所以你应该
      参考[在线副本](http://cs110.stanford.edu/autumn-2017/examples/non-blocking-io/efficient-server.cc)。
      我将在课堂上运行新的服务器，解释它的功能，并涵盖整体设计的关键部分
      以及它如何使用 `epoll` 套件来高效地作为一个非阻塞 Web 服务器运行，
      能够同时管理数万个打开的连接。

::: tip 重难点解析
**epoll —— 解决"何时去读"的问题**：回顾上一讲的非阻塞客户端，它在 2.6 秒内发起了 1100 万次无意义的 `read` 调用。`epoll` 的核心价值就在于消除这种浪费——你不再需要主动轮询"有数据了吗？"，而是告诉内核"当这些文件描述符有数据可读时通知我"。这就像从"不停敲门问好了没"变成了"好了叫我一声"。

**epoll 的三个函数分工**：
- `epoll_create`：创建一个 epoll 实例，返回一个文件描述符。这本身是一个优雅的设计——因为 epoll 实例也是文件描述符，它也可以被另一个 epoll 实例监视，从而实现层级化的事件管理。
- `epoll_ctl`：向 epoll 实例注册、修改或删除要监视的文件描述符及其感兴趣的事件类型（可读 `EPOLLIN`、可写 `EPOLLOUT` 等）。
- `epoll_wait`：阻塞等待，直到任一被监视的文件描述符上发生了注册的事件。返回就绪的文件描述符列表，调用者只需处理那些就绪的描述符。

**与之前方案的对比**：第 19 讲的静态文件服务器使用忙轮询（遍历 OutboundFile 列表并调用 `sendMoreData`）。虽然它是非阻塞的，但主循环在无数据时也会空转。而使用 `epoll` 的 `efficient-server` 会在没有任何 I/O 事件时让出 CPU（`epoll_wait` 内部将进程置于休眠状态），从而在空闲时几乎不消耗 CPU，同时仍能在单个线程中管理数万个连接——这就是著名的 C10K 问题的解决方案。

**epoll 在工业界的应用**：nginx 使用 `epoll`（Linux）/`kqueue`（BSD/macOS）作为其事件驱动架构的底层 I/O 多路复用机制；Node.js 的 `libuv` 库在 Linux 上也基于 `epoll` 实现事件循环。理解 `epoll` 的原理，你就理解了现代高性能 Web 服务器和异步 I/O 框架的核心。
:::

::: tip 重难点解析
**epoll 的内核数据结构 —— 红黑树 + 就绪链表**

epoll 的高效并非来自魔法，而是来自精心设计的内核数据结构。理解这些结构，是理解 epoll 为什么能做到 O(1) 每事件复杂度的关键。

**epoll 实例的内存布局**：

```
                    epoll 实例 (eventpoll 结构体)
                    +===================================+
                    | struct eventpoll {                |
                    |   rbr:  红黑树根节点 ------------+ |
                    |   rdllist: 就绪链表头 -----------+ |
                    |   wq:    等待队列 (epoll_wait)   | |
                    | }                                 | |
                    +===================================+
                               |               |
                               v               v
          +-------------------+         +------------+
          | 红黑树 (rbtree)    |         | 就绪链表    |
          | key = fd 编号      |         | (rdllist)  |
          | +-------+          |         | 双向链表    |
          | | fd=3  |<---+     |         | +--------+ |
          | | event |    |     |      +---->| fd=5   | |
          | +-------+    |     |      |  | | ready   | |
          |              |     |      |  | +--------+ |
          | +-------+    |     |      |  |            |
          | | fd=7  |<---+-----+------+  | +--------+ |
          | | event |    |     |         | | fd=12  | |
          | +-------+    |     |         | | ready   | |
          |              |     |         | +--------+ |
          | +-------+    |     |         |            |
          | | fd=12 |<---+     |         | (当 fd 变为 |
          | | event |          |         |  就绪时动态 |
          | +-------+          |         |  插入/移除) |
          +-------------------+         +------------+
```

**红黑树 (Red-Black Tree)**：用于管理所有被监视的文件描述符，支持 O(log N) 的插入、删除和查找。每个节点存储 `(fd, events, callback_function_pointer)`：
- `fd` 作为查找键
- `events` 记录了该 fd 上感兴趣的事件（EPOLLIN, EPOLLOUT 等）
- `callback` 是向设备驱动注册的回调函数指针

**就绪链表 (Ready List)**：一个双向链表，存储当前处于"就绪"状态的文件描述符。当数据到达时，设备驱动回调将该 fd 的 `epitem` 节点从红黑树链接到就绪链表末尾。`epoll_wait` 直接从这个链表取回就绪的 fd 列表——O(1) 每事件。

**为什么 epoll 比 select/poll 快？**
- `select`/`poll`：每次调用时，内核必须遍历所有被监视的 fd（O(N) 复杂度），无论它们是否就绪
- `epoll`：内核在数据到达时就标记了就绪 fd（通过回调机制），`epoll_wait` 只需返回就绪链表中的 fd——只有真正活跃的 fd 才会被处理

**完整的回调流程 —— 从网络包到 epoll_wait 唤醒**：

```
1. 网络数据包到达 NIC
   -> NIC 发起硬件中断

2. 中断处理程序 (ISR)
   -> 将数据包从 NIC 缓冲区拷贝到内核内存
   -> 触发软中断 (softirq) 进行后续处理

3. 内核协议栈 (TCP/IP stack)
   -> 解包 Ethernet -> IP -> TCP
   -> 将数据放入目标 socket 的接收缓冲区 (sk_buff receive queue)
   -> 检查该 socket 是否有注册的 epoll 回调

4. epoll 回调函数 (ep_poll_callback)
   -> 将该 fd 对应的 epitem 节点添加到 epoll 实例的 rdllist（就绪链表）
   -> 如果 epoll_wait 正在休眠，唤醒等待队列中等待的进程
   -> 返回

5. epoll_wait 被唤醒
   -> 遍历 rdllist 就绪链表
   -> 将就绪事件拷贝到用户空间的 events 数组
   -> 返回就绪事件数量
```

**关键洞察**：第 3-4 步发生在内核上下文中，可以被理解为"数据到达途中顺便标记了就绪状态"。而 `select`/`poll` 必须在每次被调用时主动扫描所有 fd 的接收缓冲区。这就是 epoll 能支持数十万文件描述符而性能不降级的原因。
:::

::: tip 重难点解析
**边缘触发（Edge-Triggered, ET）vs 水平触发（Level-Triggered, LT）**：

这是 `epoll` 中最重要的概念区分之一，直接影响到程序能否正确地读取所有数据：

- **水平触发（LT，默认模式）**：只要文件描述符处于"就绪"状态（例如 socket 缓冲区中还有未读数据），每次调用 `epoll_wait` 都会返回该文件描述符。这是更安全的模式——即使你这次没有读完所有数据，下次 `epoll_wait` 还会通知你。
- **边缘触发（ET）**：只有当文件描述符的状态从"未就绪"变为"就绪"时，`epoll_wait` 才会返回一次。这要求你必须在收到通知后循环 `read` 直到返回 `EAGAIN`，确保将所有可用数据一次性读完，否则剩余数据可能永远不会被通知。

ET 模式的性能更高（因为减少了不必要的通知次数），但编程难度也更大。在实际项目中，LT 更常见于对正确性要求高的场景，ET 更常见于对性能要求极高的场景。LT 和 ET 的选择类似于中断（interrupt）和轮询（polling）之间的权衡。

**补充参考**：如果你在学习 CS111（操作系统），你会发现 `epoll` 的内核实现涉及等待队列（wait queue）和回调机制——当网络数据到达时，内核协议栈将数据放入 socket 的接收缓冲区，并唤醒在该 socket 上等待的进程/线程。这是理解"内核如何支持异步 I/O"的最佳切入点。
:::

::: tip 重难点解析
**EPOLLONESHOT —— 防止多线程 epoll 中的惊群效应**

`EPOLLONESHOT` 是 `epoll_ctl` 的一个标志选项。设置后，一旦该 fd 上触发了一个事件并被 `epoll_wait` 返回，该 fd 就会从 epoll 的"活动监视"中移除——直到用户通过 `epoll_ctl(EPOLL_CTL_MOD)` 重新激活它。

**解决的问题：惊群效应 (Thundering Herd)**

在多线程 epoll 模型中（多个线程共享同一个 epoll 实例，都在 `epoll_wait` 上等待），当一个 fd 变为可读时：

- **无 EPOLLONESHOT 时**：两个或多个线程可能同时被唤醒并收到同一个 fd 的就绪通知。线程 A 开始读取数据，线程 B 也尝试读取——但此时数据可能已被 A 读完，B 读到的可能是下一个请求的部分数据（如果是 keep-alive 连接），或者立即遇到 EAGAIN。这导致了数据竞争和错误。

- **有 EPOLLONESHOT 时**：fd 就绪后只被放入就绪链表一次，只有一个线程能拿到它。该线程处理完事件后，显式调用 `epoll_ctl(EPOLL_CTL_MOD)` 重新激活该 fd。在此期间，其他线程不会收到该 fd 的通知——避免了多个线程争抢同一连接的尴尬。

**典型使用模式**：

```cpp
// 注册时带上 EPOLLONESHOT
event.events = EPOLLIN | EPOLLONESHOT;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);

// 处理完 I/O 后重新激活
// ... read/write ...
event.events = EPOLLIN | EPOLLONESHOT;  // 或 EPOLLOUT，视后续期望的事件
epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &event);
```

与边缘触发 (EPOLLET) 的配合：EPOLLONESHOT + EPOLLET 的组合确保：(1) 只有一个线程处理该连接；(2) 该线程必须读取到 EAGAIN，然后重新激活监视。这是多线程 epoll 编程的最安全模式。
:::

::: tip 重难点解析
**select vs poll vs epoll —— 全面对比**

这是三个 I/O 多路复用机制的核心差异对比：

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| **API 签名的清晰性** | fd_set (3 个位图：read/write/except) | struct pollfd 数组 | epoll_ctl 逐个添加 |
| **每次调用传递的内容** | 整个 fd_set 拷贝到内核 | 整个 pollfd 数组拷贝到内核 | 只返回就绪的 fd 列表 |
| **算法复杂度** | O(N) 每次调用 | O(N) 每次调用 | O(1) 每就绪事件 |
| **最大 fd 数量** | 1024 (FD_SETSIZE，可重编译但不可移植) | 无硬性限制 | 无硬性限制 |
| **内核数据结构** | 无持久状态 | 无持久状态 | 红黑树 + 就绪链表 |
| **随着 fd 增加如何伸缩？** | 差——每次必须遍历 0..maxfd | 差——每次必须遍历整个数组 | 优秀——只处理就绪的 fd |
| **修改监视集合成本** | 无（每次构造新的 fd_set） | 无 | O(log N)（红黑树插入/删除） |
| **触发模式** | 仅水平触发 | 仅水平触发 | 水平触发（默认）+ 边缘触发 |
| **跨平台** | POSIX，到处可用 | POSIX，比 select 更现代 | Linux 特有 |

**为什么 select 有 1024 限制？**

`fd_set` 底层是一个位图（bitmask），大小固定为 `FD_SETSIZE`（通常 1024 位）。每个位对应一个 fd 编号。当你要监视 fd=1000 时，内核检查位图的第 1000 位。如果 fd 编号超过 1023，位图就没有对应的位了。虽然可以通过重编译 glibc 来增加 `FD_SETSIZE`，但这破坏了二进制兼容性。

**什么时候用哪个？**

- **select**：仅当跨平台可移植性是首要考虑，且 fd 数量极少（<100）
- **poll**：在非 Linux 平台上需要监视较多 fd（100-1000），且不需要边缘触发
- **epoll**：Linux 平台上的最佳选择，尤其当 fd 数量 >100 或多个连接大部分时间处于空闲状态时
:::

::: tip 重难点解析
**从 epoll 到更高级的抽象**：直接使用 `epoll` 编写服务器是可行的，但代码繁琐且不可移植（macOS 使用 `kqueue`，Windows 使用 IOCP）。因此工业界发展出了多个跨平台的异步 I/O 抽象层：

- **libev / libuv**：libev 是一个精简的事件循环库，libuv 最初为 Node.js 开发，在 Linux 上使用 `epoll`，在 macOS 上使用 `kqueue`，在 Windows 上使用 IOCP，提供统一的跨平台异步 I/O 接口。
- **Tornado**：一个 Python 的异步 Web 框架，使用单线程事件循环处理高并发连接。
- **Node.js**：基于 V8 JavaScript 引擎和 libuv，将事件驱动 + 非阻塞 I/O 的模式带入了服务器端 JavaScript 的主流。

CS110 教你理解这些技术背后的原理——`epoll`、非阻塞 socket、事件循环——使得你无论将来使用哪个框架，都能理解它"为什么这样设计"以及"内部发生了什么"。
:::

::: tip 重难点解析
**kqueue —— macOS/BSD 上的 epoll 等价物**

macOS 和 FreeBSD 不使用 `epoll`，而是提供 `kqueue`/`kevent` 接口。理解 `kqueue` 对于跨平台开发非常重要。

**API 对比**：

| epoll (Linux) | kqueue (macOS/BSD) | 说明 |
|---------------|-------------------|------|
| `epoll_create(1)` | `kqueue()` | 创建监视实例，返回 fd |
| `epoll_ctl(epfd, ADD, fd, &ev)` | `EV_SET(&kev, fd, EVFILT_READ, EV_ADD\|EV_ENABLE, 0, 0, NULL); kevent(kq, &kev, 1, NULL, 0, NULL);` | 注册/修改监视 |
| `epoll_wait(epfd, events, max, timeout)` | `kevent(kq, NULL, 0, events, max, &timeout);` | 等待事件 |

**kqueue 的独特优势**：

1. **更广泛的监视对象**：kqueue 不仅监视文件描述符，还可以监视进程退出 (EVFILT_PROC)、信号 (EVFILT_SIGNAL)、文件系统变化 (EVFILT_VNODE)、定时器 (EVFILT_TIMER)、甚至 Mach port。这比你用 `epoll` + `signalfd` + `timerfd` 的组合更统一。

2. **文件系统事件监视**：
```c
EV_SET(&kev, fd, EVFILT_VNODE, EV_ADD | EV_ENABLE | EV_CLEAR,
       NOTE_WRITE | NOTE_DELETE | NOTE_RENAME, 0, NULL);
```
这可以监视一个文件的写入、删除和重命名事件——与 `inotify` (Linux) 类似但更优雅地集成在同一个 API 中。

3. **进程监视**：
```c
EV_SET(&kev, pid, EVFILT_PROC, EV_ADD | EV_ENABLE,
       NOTE_EXIT | NOTE_FORK, 0, NULL);
```
无需 `waitpid` + `WNOHANG` 轮询——内核会在进程退出时自动通知你。

**边缘触发在 kqueue 中**：通过 `EV_CLEAR` 标志实现，行为类似 `EPOLLET`。设为 `EV_CLEAR` 后，事件仅在状态变化时报告，需要读取直到 `EAGAIN`。
:::

::: tip 重难点解析
**libuv 事件循环的完整阶段 —— Node.js 的核心引擎**

`libuv` 是 Node.js 底层的跨平台异步 I/O 库。理解其事件循环的各个阶段对于理解 Node.js 的行为至关重要。

**libuv 事件循环的 7 个阶段**：

```
  +---> [ 1. timers ] --------+
  |                           v
  |    [ 2. pending callbacks ]
  |                           |
  +--- [ 7. close callbacks ] |
  ^                           v
  |    [ 6. check ]     [ 3. idle, prepare ]
  ^                           |
  |    [ 5. poll ] <-- [ 4. poll (I/O) ]
  |         |
  +---------+
```

每个阶段的职责：

1. **Timers（定时器阶段）**：执行 `setTimeout()` 和 `setInterval()` 的回调。注意：定时器保证的是"至少 delay ms 后执行"，而非"精确 delay ms 后执行"——如果 poll 阶段执行时间过长，timer 回调会被延迟。

2. **Pending Callbacks（挂起回调阶段）**：执行推迟到下一轮循环的 I/O 回调。主要是某些系统操作（如 TCP 错误）的回调。

3. **Idle / Prepare（空闲/准备阶段）**：内部使用，通常不暴露给用户。

4. **Poll（轮询阶段）**：事件循环的核心。在此阶段：
   - 计算需要阻塞多久（基于最近的 timer 到期时间）
   - 调用 `epoll_wait`/`kqueue`/`IOCP` 等待 I/O 事件
   - 执行已就绪的 I/O 回调（如 `read`、`connect` 回调）
   - 如果没有 timer 且没有活跃的 I/O 句柄，事件循环在此退出

5. **Check（检查阶段）**：执行 `setImmediate()` 的回调。这个阶段在 poll 之后立即执行，优先于下一轮 timer。

6. **Close Callbacks（关闭回调阶段）**：执行关闭事件回调，如 `socket.on('close', ...)`。

**关键顺序保证**：

```
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

执行顺序**不确定**（取决于事件循环启动时是否已到定时器时间）。如果两者都在 poll 阶段之后被调度，`setImmediate` 先执行（check 先于下一轮 timer）；如果 timer 在 poll 之前到期，`setTimeout` 先执行。

**CS110 中的对应**：你在作业中写的事件循环（while 循环遍历连接）是 libuv poll 阶段的简化版本。CS110 让你理解轮询循环的本质，而 libuv 在此基础上添加了 timer 管理、异步操作排队和各阶段的有序执行。
:::
