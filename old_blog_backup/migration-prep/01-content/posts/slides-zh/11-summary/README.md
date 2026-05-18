# 线程与并发

* 指定阅读
    * 你现在正在阅读第 12.1 节以及第 12.3 节到第 12.8 节的全部内容，跳过那些涉及网络和服务器的子节。
        * 第 12 章在教材中实际对应教材的第四章。
        * 代码示例是用 C 语言编写的，但这些概念在不同语言之间大体相同。
    * 等我们学完网络编程之后，会回过头来讲解一些被跳过的章节。

* 今日议程
    * 通过 `ticket-agents` 示例，分析为什么除了最终版本之外的所有版本都有缺陷，可能产生错误的结果。
    * 讲解经典的"哲学家就餐问题"（Dining Philosophers）并发示例——这个问题几乎在每一门教授多线程和并发的课程中都会出现。
        * 我们将实现共 **四个** 哲学家就餐模拟。
        * 每一个模拟都将推动我们走向一个更好的解决方案：该方案依赖多线程，同时确保那些无法进行有意义工作的线程浪费最少的 CPU 资源。

# 回顾售票代理示例

* 上一讲幻灯片中的售票代理示例是我们目前最复杂的示例。
    * 我们依赖全局变量来存储需要与所有线程共享的数据。（线程通常用于将处理器划分为多个轻量级进程，这些进程共同朝着一个目标努力，售票代理示例正是这样一个例子。）
    * 代码的某些部分不能被部分执行然后从处理器上撤下，因为这样做会引入一种称为**竞态条件**（race condition）的同步问题——如果另一个线程获得处理器时间，并在同一代码块中取得部分进展的话。
    * 即使是像 `remainingTickets--;` 这样简单的 C++ 表达式也不总是原子的，因为它们可能被编译成两条或更多条汇编代码指令。单条汇编代码指令总是原子的，但两条或三条指令的组合不能保证在同一个时间片内按顺序执行。
    * 必须完整执行而不受其他线程干扰的代码块（甚至可能是单个表达式）被称为**临界区**（critical region）。

::: tip 重难点解析
**竞态条件（Race Condition）**：当多个线程同时访问共享数据，且最终结果依赖于线程执行的精确时序时，就发生了竞态条件。可以这样想象：两个人同时去银行从同一个账户取钱，如果系统没有正确同步，两个人可能都看到账户余额为 1000 元，然后各取走 1000 元——结果银行损失了 1000 元。在多线程编程中，这种时序相关的 bug 非常难以复现和调试，因为它们的出现依赖于调度器的"坏运气"。
:::

::: tip 重难点解析
**临界区（Critical Region）**：临界区是指访问共享资源（如全局变量、静态变量、文件等）的代码段，且这些代码段在同一时刻只能由一个线程执行。识别并正确保护临界区是多线程编程的核心技能。一个简单的判断原则：任何读写多个线程共享变量的代码段，都应该被视为临界区。
:::

# 回顾售票代理示例（续）

* 我们是如何修复的？
    * 我们引入了 `mutex`（互斥锁）的概念，它提供了称为 `lock` 和 `unlock` 的原子方法。
        * 如果一个 `mutex` 是解锁状态的（构造时就是如此），那么调用 `lock` 会将 `mutex` 切换为锁定状态并立即返回，不会阻塞或让出处理器。
        * 如果一个 `mutex` 被某个其他线程锁定，那么另一个线程尝试对同一个 `mutex` 调用 `lock` 会导致该线程在 `lock` 调用中无限期阻塞，直到拥有该 `mutex` 锁所有权的线程通过调用 `unlock` 释放它。
        * `lock` 和 `unlock` 通常用于标记临界区的开始和结束，因为这样做可以保证在任意时刻最多只有一个线程在执行该临界区的任何部分。只有在该线程退出临界区（并正确调用 `unlock`）之后，其他线程才能获取同一个 `mutex` 的锁并进入同一个临界区。
        * 总体效果是，在任意时刻最多只有一个线程在临界区内，因此临界区内的所有代码实际上作为一个原子事务执行。
    * `lock` 和 `unlock` 的实现依赖于特权操作系统访问（例如关闭中断的能力）和/或专用的汇编代码指令（例如 test-and-set），以确保它们自身被实现为原子操作，不会受到任何竞态条件的威胁。
    * 一般来说，在不引入任何并发问题的前提下，将临界区保持得尽可能小被认为是良好的编程习惯。

::: tip 重难点解析
**互斥锁（Mutex）的工作原理**：互斥锁本质上是操作系统提供的一个原子性保障机制。你可以把它想象成一把物理钥匙——只有拿到钥匙的人才能进入房间（临界区），其他人必须在门外排队等待。关键在于，"检查钥匙是否可用"和"拿走钥匙"这两个操作必须是原子的（不可分割的），否则两个线程可能同时看到钥匙在桌上，并都认为自己拿到了钥匙。现代 CPU 通过特殊的原子指令（如 x86 的 `CMPXCHG` 或 ARM 的 `LDREX/STREX`）来支持 mutex 的实现。
:::

::: tip 重难点解析
**Mutex 的内核实现：futex 机制**

Linux 中 mutex 的底层实现基于 futex（fast userspace mutex）。其核心思路是"能用户态解决就不进内核"：

1. **快速路径（无竞争）**：`lock()` 使用一条原子 compare-and-swap（CAS）指令尝试获取锁。如果锁是空闲的（值为 0），CAS 将其原子地改为 1，线程获得锁——全程无系统调用，仅需几十个 CPU 周期。
2. **慢速路径（有竞争）**：如果 CAS 失败（锁已被持），线程调用 `futex(FUTEX_WAIT)` 系统调用进入内核。内核将线程放入该 futex 的等待队列，标记为 TASK_INTERRUPTIBLE，然后调度其他线程运行。
3. **唤醒**：`unlock()` 使用原子操作将锁值设回 0。如果有线程在 futex 上等待，调用 `futex(FUTEX_WAKE)` 唤醒一个（或所有）等待线程。被唤醒的线程重新尝试 CAS 获取锁。

futex 的设计权衡反映了系统编程的核心哲学：将常见情况（无竞争）优化到极致，罕见情况（有竞争）才付出内核态切换的代价。CS111 锁实现章节会展开讨论 futex 及其他锁实现（自旋锁、排队锁、RCU 等）。
:::

# [哲学家就餐问题](http://en.wikipedia.org/wiki/Dining_philosophers_problem)（Dining Philosophers Problem）

* 经典的并发问题，用于说明死锁的威胁。
    * 五位哲学家围坐在一张圆桌旁，每人面前都有一大盘意大利面。
    * 每两位相邻的哲学家之间各放一把叉子。
    * 每天，每位哲学家来到桌前思考、吃饭、思考、吃饭、思考、吃饭。这是三次长时间的思考之后的三顿丰盛的意大利面正餐。
        * 每位哲学家在思考时都沉浸在自己的世界里。有时他思考很长时间，有时他只思考一小会儿。
        * 每位哲学家思考一段时间后，他会开始吃他的三顿日常餐中的一顿。为了吃饭，他必须同时抓住两把叉子——他左边的一把和右边的一把。一旦做到这一点，他就大口吃意大利面来滋养他那颗伟大的、善于思考的大脑。当他吃饱后，他按拿起叉子的相同顺序放下叉子，然后回去继续思考一段时间。

::: tip 重难点解析
**哲学家就餐问题**：这是由 Dijkstra 提出、Hoare 推广的经典同步问题，被广泛用于说明死锁（deadlock）和饥饿（starvation）问题。关键矛盾在于：每位哲学家需要两把共享的叉子才能进餐，但如果所有哲学家同时拿起左边的叉子，然后等待右边的叉子，系统将陷入死锁——每个人都在等待，没有人能继续。这个问题完美地展示了并发编程中"部分分配"策略的危险性。
:::

# 哲学家就餐问题（续）

* 下面是一个模拟我们刚才所描述内容的 C++ 程序的核心代码（点击[这里](http://cs110.stanford.edu/autumn-2017/examples/threads-cpp/dining-philosophers-with-deadlock.cc)查看完整程序）：

    ```cpp
    static const unsigned int kNumPhilosophers = 5; // must be 2 or greater
    static const unsigned int kNumForks = kNumPhilosophers;
    static const unsigned int kNumMeals = 3;

    static mutex forks[kNumForks]; // forks modeled as mutexes

    static void think(unsigned int id) {
      cout << oslock << id << " starts thinking." << endl << osunlock;
      sleep_for(getThinkTime());
      cout << oslock << id << " all done thinking. " << endl << osunlock;
    }

    static void eat(unsigned int id) {
      unsigned int left = id;
      unsigned int right = (id + 1) % kNumForks;
      forks[left].lock();
      forks[right].lock();
      cout << oslock << id << " starts eating om nom nom nom." << endl << osunlock;
      sleep_for(getEatTime());
      cout << oslock << id << " all done eating." << endl << osunlock;
      forks[left].unlock();
      forks[right].unlock();
    }

    static void philosopher(unsigned int id) {
      for (unsigned int i = 0; i < kNumMeals; i++) {
        think(id);
        eat(id);
      }
    }

    int main(int argc, const char *argv[]) {
      thread philosophers[kNumPhilosophers];
      for (unsigned int i = 0; i < kNumPhilosophers; i++)
        philosophers[i] = thread(philosopher, i);
      for (thread& p: philosophers)
        p.join();
      return 0;
    }
    ```

# 哲学家就餐问题（续）

* 它是如何工作的，以及它在哪里有问题？
    * 程序将每把叉子建模为一个 `mutex`。每位哲学家要么拥有叉子，要么没有，我们希望抓取叉子的动作是原子且事务性的。
    * 这个程序表面上看起来运行良好（我们会在课堂上运行几次），但它不能防范这种可能性：每位哲学家从深度思考中醒来，成功抓住了左边的叉子，然后因为时间片用完而被从处理器上撤下。
        * 如果这种病态的调度模式出现，最终所有哲学家都会被阻塞，无法抓到右边的叉子。
        * 这将使程序进入一种所有五个线程都陷入死锁的状态，因为每位哲学家都卡在等待他右边的哲学家放下其左边的叉子。
        * 这就像一场大型的、智力上的沉默竞赛，程序无法取得任何进展。

::: tip 重难点解析
**死锁的四个必要条件**：这个例子完美地展示了死锁（deadlock）的四个必要条件（Coffman 条件）：(1) **互斥**——叉子一次只能被一位哲学家持有；(2) **持有并等待**——哲学家持有一把叉子，同时等待另一把；(3) **不可抢占**——不能强行从哲学家手中抢走叉子；(4) **循环等待**——每位哲学家都在等待下一位手中的叉子，形成闭环。破坏这四个条件中的任意一个，就能预防死锁。CS111 中会对此做更深入的理论探讨。
:::

::: tip 重难点解析
**死锁预防策略详解**

对应四个必要条件，各有对应的预防手段：

1. **破坏互斥**：尽量使用无锁数据结构（lock-free data structures），或通过资源虚拟化（如打印机假脱机 spooling）让看起来需要互斥的资源变得可共享。局限性：很多资源本质上是不可共享的（如写文件时的 inode 锁）。

2. **破坏持有并等待**：要求线程一次性获取所有所需资源，而不是分步获取。在代码中体现为同时获取多个锁——C++11 提供了 `std::lock(l1, l2, l3)` 来原子地同时获取多个 mutex。如果无法获取全部，则释放已获取的。

3. **破坏不可抢占**：允许系统从持有资源的线程手中夺走资源。在数据库系统中，这体现为事务回滚（rollback）。

4. **破坏循环等待**：**锁排序**（lock ordering）——为所有锁定义一个全局顺序，线程只能按此顺序获取锁。例如在哲学家问题中，让偶数编号的哲学家先拿左叉、奇数编号的先拿右叉（非对称获取）。

在日常编程中最实用的是策略 4（锁排序），这也是 CS111 操作系统锁设计中反复强调的实践。
:::

# 防止死锁

* 在使用线程编程时，你需要确保：
    * 永远不存在任何竞态条件，无论概率有多小，并且……
    * 没有死锁的可能性，以免一部分线程被永远阻塞并因缺乏处理器时间而饥饿。
* 正如我们在售票代理示例中看到的，`mutex` 通常是解决竞态条件的方案。
* 死锁可以通过**编程方式**防止，通过植入指令来限制尝试参与可能导致死锁的线程数量。
    * 例如，我们可以认识到三位或更多哲学家不可能同时进餐，这可以通过简单的鸽巢原理来论证（例如，三位哲学家可以同时进餐当且仅当有六把叉子，而实际上并没有）。因此，我们可以限制同时抓取叉子的哲学家数量为 2。
    * 我们也可以论证允许最多四位（但不是全部五位）哲学家进入思考-进餐循环中的进餐部分是安全的，因为知道至少有一位会成功抓住两把叉子。

::: tip 重难点解析
**鸽巢原理在并发中的应用**：5 把叉子最多支持 2 位哲学家同时进餐（因为每人需要 2 把叉子，2 人占用 4 把，剩余 1 把不足以让第 3 人进餐）。这是资源分配的一个基本原理——通过限制同时竞争的线程数量不超过可同时满足的最大数量，可以彻底避免死锁。这种方法称为"资源排序"或"限制并发度"策略。
:::

::: tip 重难点解析
**哲学家就餐问题的多种解法与权衡**

五种经典的解决方案，从简单到完善：

1. **限制并发度（PPT 中的方案）**：通过 semaphore 限制最多 4 位哲学家同时尝试进餐。简单有效，但限制了理论上可能的最大并发度（例如，如果哲学家 0 和 2 同时进餐，它们不共享叉子，完全不需要限制）。

2. **非对称获取（Asymmetric Acquisition）**：偶数编号的哲学家先拿左叉再拿右叉，奇数编号的先拿右再拿左。这破坏了循环等待条件——如果所有哲学家都先拿左叉，互锁形成闭环；非对称打破了这个环。优点是零额外开销。

3. **仲裁者/服务员（Arbitrator/Waiter）模式**：引入一个额外的"服务员"线程或互斥锁。哲学家在拿叉子前必须获得服务员许可，服务员确保不会因此产生死锁。这相当于将叉子的分配集中管理。

4. **资源层次（Resource Hierarchy）**：给每把叉子编号（如 F0 到 F4），要求所有哲学家按编号递增顺序获取叉子。最后一位哲学家必须按 F4→F0 获取（或反过来）——编号最大的叉子被"违规"获取，打破环。

5. **`try_lock` 与回退**：哲学家尝试获取左叉（使用 `mutex` 的 `try_lock`），如果成功，尝试获取右叉；如果右叉获取失败，释放左叉，随机退避一段时间后重试。这破还了持有并等待条件，但可能引入活锁（livelock）——所有哲学家不断拿起-放下，无人进餐。

**选择标准**：(a) 公平性——所有哲学家都能获得公平的进餐机会？(b) 并发度——能否支持最大可能的并发？(c) 复杂度——理解和实现难度如何？CS111 调度与同步章节会讨论这些权衡在真实操作系统中的体现。
:::

::: tip 重难点解析
**死锁检测：资源分配图与环检测**

防范死锁之外的另一种方法是**死锁检测与恢复**。操作系统可以维护一张**资源分配图**：

- 节点：线程（圆形）和资源（方形）
- 分配边（Assignment Edge）：资源 → 线程，表示资源已分配给该线程
- 请求边（Request Edge）：线程 → 资源，表示线程正在等待该资源

检测算法：在资源分配图中搜索环。如果存在环，系统处于死锁状态。对于每种类型的资源只有一个实例的情况，等价于在有向图中检测环（DFS 即可）。

恢复策略包括：(1) **终止线程**（选择一个环中的线程终止，释放其资源），(2) **资源抢占**（从线程手中夺走资源）。选择"牺牲者"的策略考虑线程优先级、已执行时间、持有资源数量等因素。

对于哲学家问题：如果发现五个线程全部卡在 `forks[right].lock()`，就检测到了死锁。恢复方法是强制一位哲学家放下左叉（破坏持有并等待）。

在 CS111 中会看到，数据库系统（two-phase locking）和操作系统（死锁检测守护进程）大量使用这种检测机制。
:::

# 防止死锁（续）

* 下面是一个采用第二种方法的核心程序代码（点击[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-cpp/dining-philosophers-with-busy-waiting.cc)查看完整程序）：

    ```cpp
    static const unsigned int kNumPhilosophers = 5;
    static const unsigned int kNumForks = kNumPhilosophers;
    static const unsigned int kNumMeals = 3;
    static mutex forks[kNumForks];
    static unsigned int numAllowed = kNumPhilosophers - 1; // impose limit to avoid deadlock
    static mutex numAllowedLock;

    static void think(unsigned int id) {
      cout << oslock << id << " starts thinking." << endl << osunlock;
      sleep_for(getThinkTime());
      cout << oslock << id << " all done thinking. " << endl << osunlock;
    }

    static void waitForPermission() {
      while (true) {
        numAllowedLock.lock();
        if (numAllowed > 0) break;
        numAllowedLock.unlock();
        sleep_for(10);
      }
      numAllowed--;
      numAllowedLock.unlock();
    }

    static void grantPermission() {
      numAllowedLock.lock();
      numAllowed++;
      numAllowedLock.unlock();
    }

    static void eat(unsigned int id) {
      unsigned int left = id;
      unsigned int right = (id + 1) % kNumForks;
      waitForPermission();
      forks[left].lock();
      forks[right].lock();
      cout << oslock << id << " starts eating om nom nom nom." << endl << osunlock;
      sleep_for(getEatTime());
      cout << oslock << id << " all done eating." << endl << osunlock;
      grantPermission();
      forks[left].unlock();
      forks[right].unlock();
    }
    ```

# 忙等待（Busy Waiting）

* 我们修复了什么？我们又破坏了什么？
    * 上一张幻灯片中的程序始终能正常工作，因为不可能所有五位哲学家在同一时刻互相阻塞。我们只允许四位继续并抓取叉子。
    * 然而，它确实存在一个设计缺陷，我们将讨论这个设计缺陷，以此作为提出更好解决方案的途径。

* 我所说的这个设计缺陷是什么？
    * 该解决方案使用了**忙等待**（busy waiting），在系统程序员的眼中，这通常是一个大忌，除非你别无选择。
    * 要理解我的意思，请关注 `waitForPermission` 的实现：

    ```cpp
    static void waitForPermission() {
      while (true) {
        numAllowedLock.lock();
        if (numAllowed > 0) break;
        numAllowedLock.unlock();
        sleep_for(10);
      }
      numAllowed--;
      numAllowedLock.unlock();
    }
    ```

    * 其逻辑相当直观：持续轮询 `numAllowed` 的值，直到该值为正。此时，将其递减以模拟共享资源的消耗——在本例中，是一张允许哲学家开始抓取叉子的许可条。由于有多个线程可能同时在检查和递减 `numAllowed`，需要将这块临界区标识为需要用 `mutex` 来保护的区域。如果哲学家发现 `numAllowed` 的值为 0（即所有许可条都已发出），则释放锁并将处理器让给其他可能真正能做有用工作的哲学家线程。
    * 问题在哪？上述解决方案使用了忙等待，这是一个并发术语，用于描述线程周期性检查某些条件是否已改变，以便可以继续做更有意义的工作。在大多数情况下，忙等待的问题在于，它时不时地独占处理器，而更好的做法是确保其他线程——那些大概有有意义工作可以做的线程——获得处理器。
    * 更好的解决方案：如果哲学家没有获得继续进行的许可（例如 `numAllowed` 被原子地确认为零），那么该线程应该被**无限期**地置于睡眠状态，直到某个其他线程发现有理由将其唤醒。在这个例子中，另一位哲学家线程在 `grantPermission` 中递增 `numAllowed` 之后，可以通知被无限期阻塞的线程现在有一些许可条可用了。
    * 实现这个想法需要一个更复杂的并发指令，它支持一种不同的线程通信形式。幸运的是，C++11 提供了一个标准的指令，称为 `condition_variable_any`。

::: tip 重难点解析
**忙等待 vs 阻塞等待**：忙等待（busy waiting）的线程虽然暂时无法做有用工作，但仍然占用 CPU 时间片，不断检查条件是否满足——就像一个人不停地按电梯按钮，希望电梯来得更快。阻塞等待（blocking wait）则让线程主动放弃 CPU 进入睡眠，直到条件满足时被唤醒——这就像在电梯口静坐等待，电梯到了自然会通知你。在 CPU 资源紧张时，忙等待会严重浪费系统资源，但在等待时间极短且上下文切换开销较大时，忙等待（也称自旋锁，spin lock）反而可能更高效——这需要根据具体场景权衡。
:::

# 无忙等待的死锁预防

* 一个更好的解决方案——不允许线程忙等待的方案——可以用 `condition_variable_any` 来构建。
    * 第三版：程序的大部分保持不变，但有一些新的全局变量，并且 `waitForPermission` 和 `grantPermission` 的实现有所改变。你可以点击[这里](http://www.stanford.edu/class/cs110/autumn-2017/examples/threads-cpp/dining-philosophers-with-condition-variable.cc)查看完整程序。
    * 以下代码总结了第三版的不同之处：

    ```cpp
    static mutex forks[kNumForks];
    static int numAllowed = kNumForks - 1;
    static mutex m;
    static condition_variable_any cv;

    static void waitForPermission() {
      lock_guard<mutex> lg(m);
      cv.wait(m, []{ return numAllowed > 0; });
      numAllowed--;
    }

    static void grantPermission() {
      lock_guard<mutex> lg(m);
      numAllowed++;
      if (numAllowed == 1) cv.notify_all();
    }
    ```

    * 暂时忘记名为 `m` 的 `mutex`，将注意力集中在名为 `cv` 的 `condition_variable_any` 上。
    * `condition_variable_any` 是核心的并发指令，它可以抢占并阻塞一个线程，直到某个条件被满足。
        * 在这个例子中，寻求进食许可的哲学家无限期地 `wait`（等待），直到某个条件成立（除非该条件已经成立，此时它不需要等待）。
            * 如果在 `wait` 被调用时 `numAllowed` 为正，则它立即返回而不阻塞。
            * 如果在 `wait` 被调用时 `numAllowed` 为零，则调用线程被从处理器上撤下，标记为不可运行并阻塞，直到线程管理器被（另一个线程）告知 `numAllowed` 的值已改变。
        * 在这个例子中，刚吃完一顿饭的哲学家递增 `numAllowed`，并且如果 `numAllowed` 的值从 0 变为 1，同一位哲学家通过 `notify_all` 向所有在 `condition_variable_any` 上阻塞的线程发出信号，告知有意义的更新已发生。这会提示线程管理器代表所有被 `wait` 阻塞的线程重新检查条件，并可能允许其中一个或多个线程从它们的长眠中苏醒，继续它们之前不被允许进行的工作。

::: tip 重难点解析
**条件变量（Condition Variable）**：条件变量解决的是"等待条件成立"问题。它必须与互斥锁配合使用，因为检查条件和开始等待这两个操作必须是原子的。常见的使用模式是：(1) 持有锁，(2) 检查条件，(3) 如果条件不满足则调用 `wait`（它会原子地释放锁并阻塞线程），(4) 被唤醒时自动重新获取锁，(5) 重新检查条件（因为可能有假唤醒或条件已被其他线程改变）。这种等待-检查-释放-重新获取的模式称为 Mesa 语义，是大多数条件变量实现的默认行为。
:::

::: tip 重难点解析
**条件变量为什么需要 mutex：丢失唤醒问题（Lost Wakeup）**

这个问题的本质是：条件的"检查"和"等待"之间存在间隙，信号可能正好在这个间隙中到达。

```cpp
// 错误写法（不用 mutex 保护条件检查）
if (numAllowed == 0)           // T1: 检查条件
    // <-- 信号可能在此刻到达！
    cv.wait();                 // T1: 进入等待（但信号已经丢失）

// T2 在别处：
numAllowed++;                  // T2: 改变条件
cv.notify_all();               // T2: 发送信号（但 T1 还没开始 wait）
```

问题出在哪里？如果 T2 的 `notify_all()` 正好在 T1 检查条件之后、调用 `wait()` 之前执行，那么 T1 将错过这个 signal，永远阻塞。这被称为**丢失唤醒**（lost wakeup）问题。

正确的做法是让 mutex 来保护条件：
```cpp
lock_guard<mutex> lg(m);       // 锁定 mutex
cv.wait(m, []{ return numAllowed > 0; });  // 条件检查和进入等待是原子的
```

`cv.wait(m, pred)` 的内部实现等价于：
```cpp
while (!pred()) {
    // 原子操作：释放 m, 然后阻塞当前线程
    // 被唤醒后：重新获取 m, 然后返回 while 循环检查 pred
}
```

关键设计：`wait` 内部**原子地释放 mutex 并进入阻塞状态**。这两个操作的原子性由内核保证——内核在将线程从运行队列移除时，才释放 mutex。这样，任何 `notify` 操作都只能发生在"线程已经进入等待状态"之后，消除丢失唤醒的可能。

CS111 中会详细讨论内核中 futex 如何实现这个"原子释放并阻塞"的语义。
:::

::: tip 重难点解析
**假唤醒（Spurious Wakeup）与 while 循环的必要性**

即使没有线程调用 `notify_one` 或 `notify_all`，`wait` 也可能被唤醒——这称为**假唤醒**（spurious wakeup）。POSIX 标准允许这种行为，原因有二：

1. **实现效率**：底层使用 futex 的系统可能收到虚假的内核事件（如信号处理、中断），与其在内核中精确区分唤醒原因，不如让用户态重新检查条件——这保持了内核实现的简单和快速。
2. **notify_all 与多消费者**：当使用 `notify_all` 广播时，多个被唤醒的线程竞争同一个条件。只有一个能"抢到"条件（如唯一的一份资源），其他线程醒来后发现条件又变回 false——对它们而言，这是一次"虚假的"唤醒。

因此，条件变量的正确使用方式是**始终在 while 循环中等待**：

```cpp
lock_guard<mutex> lg(m);
while (numAllowed == 0) {     // while，不是 if！
    cv.wait(m);                // 如果被假唤醒，while 循环会重新检查条件
}
// 此时 numAllowed > 0 一定成立
numAllowed--;
```

C++11 的 `cv.wait(m, pred)` 已经内建了这种 while 循环语义，lambda 谓词会被反复检查。这等价于上面的手动 while 循环。
:::

::: tip 重难点解析
**notify_one vs notify_all：何时使用哪个**

这是条件变量使用中最常见的困惑之一：

- **`notify_one()`**：唤醒一个在 cv 上等待的线程（具体哪个由调度器决定，通常是等待时间最长的）。适用场景：所有等待者检查相同的条件，且每次只有一个线程能利用条件（如互斥资源、一份工作）。

- **`notify_all()`**：唤醒所有在 cv 上等待的线程。适用场景：
  1. **多个资源单位可用**：如果 numAllowed 从 0 变为 3（释放了 3 份资源），notify_one 只会唤醒一个线程，浪费了另外两份资源。
  2. **不同等待者检查不同的条件**：多个消费者在同一个 cv 上等待不同的谓词（不推荐的设计，但实际存在）。
  3. **不确定唤醒哪个**：当无法确定哪一个等待线程能够利用条件时，广播让每个线程自行检查。

本示例中 `grantPermission` 使用 `notify_all()` 是安全的，虽然资源每次只增 1（`numAllowed` 从 0 变 1），使用 `notify_one()` 就足够了——这里使用 `notify_all()` 起到了示范作用。在实践中，如果确定只有一个线程能满足条件，用 `notify_one()` 更高效，因为避免了无谓的上下文切换开销（被唤醒但看到条件仍不满足的线程会重新进入等待）。

一个容易出错的情况：如果使用 `notify_one` 但被唤醒的线程条件检查失败（如其他线程抢走了资源），且没有其他通知会到来，剩余的等待线程可能永远阻塞。这种场景必须使用 `notify_all`。
:::

# 细节

* 这些细节微妙但重要。
    * 再看一遍代码：

    ```cpp
    static mutex forks[kNumForks];
    static int numAllowed = kNumForks - 1;
    static mutex m;
    static condition_variable_any cv;

    static void waitForPermission() {
      lock_guard<mutex> lg(m);
      cv.wait(m, []{ return numAllowed > 0; });
      numAllowed--;
    }

    static void grantPermission() {
      lock_guard<mutex> lg(m);
      numAllowed++;
      if (numAllowed == 1) cv.notify_all();
    }
    ```

    * 因为 `numAllowed` 被多个线程并发地检查和可能修改，并且因为一个以它为条件的条件（即 `numAllowed > 0`）影响线程是阻塞还是继续不受阻碍地运行，我们这里需要一个传统的 `mutex`，以便竞争的线程可以锁定对 `numAllowed` 相关所有内容的独占访问。
    * `lock_guard` 类用于自动管理 `mutex` 的锁定和解锁。
        * `lock_guard` 构造函数绑定一个内部引用到所提供的 `mutex` 并对其调用 `lock`（在构造函数内阻塞，直到锁可以被获取）。
        * `lock_guard` 析构函数释放对同一个 `mutex` 的锁。
        * 总体效果——至少在上述两个函数中——是两个实现的组合被标记为一个大的临界区。
            * Java 高手注意：`lock_guard` 类是 C++ 中模拟 Java 的 `synchronized` 关键字的最佳选择。
        * `lock_guard` 是一个模板，因为其实现只要求其模板类型响应零参数的 `lock` 和 `unlock` 方法。`mutex` 类当然可以做到，但许多其他类（例如 C++11 的 `recursive_mutex` 或其 `timed_mutex`）也可以，而 `lock_guard` 不想硬编码为 `mutex`。
    * `condition_variable_any::wait` 要求提供一个 `mutex` 来帮助管理它对所提供条件的同步。
    * 如果 `condition_variable_any::wait` 发现所提供的条件未得到满足，它会将当前线程无限期地置于睡眠状态并释放所提供的 `mutex`。当 `condition_variable_any` 被通知并且一个等待的线程被允许继续时，该线程会重新获取它在进入睡眠前释放的 `mutex`。

::: tip 重难点解析
**RAII 与 lock_guard**：`lock_guard` 是 C++ RAII（Resource Acquisition Is Initialization）惯用法在并发编程中的经典应用。"资源获取即初始化"的含义是：在构造函数中获取资源（这里是指锁定 mutex），在析构函数中释放资源（解锁 mutex）。这样做的好处是：即使函数因为异常而提前退出，析构函数仍会被自动调用，从而保证锁一定会被释放，避免死锁。与手动调用 `lock()`/`unlock()` 相比，`lock_guard` 更安全、更简洁。CS110 和 CS111 都会大量使用这种模式。
:::
