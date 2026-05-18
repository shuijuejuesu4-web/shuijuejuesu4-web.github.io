---
title: xv6 riscv book chapter 9：Concurrency revisited
date: 2025-08-04
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---
# xv6 riscv book chapter 9：Concurrency revisited

在内核设计中，要同时拥有良好的平行效能、并发下的正确性，以及可理解的代码，是一项重大挑战。 直接使用 lock 是达成正确性最可靠的方式，但这并非总是可行的。 本章会列出一些例子：有些情况 xv6 被迫以复杂的方式使用 lock，也有些情况则采用了类似 lock 的技巧，但实际上没有用 lock

## 9.1 Locking patterns

对于 cache 项目的锁定常常是一项难题。 例如，文件系统的 block cache（[kernel/bio.c:26](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L26)）会存储最多 `NBUF` 个 disk block 的副本。 系统必须确保同一个 disk block 在 cache 中最多只能有一份副本，否则不同的 process 可能会对同一个 block 的不同副本进行互相冲突的修改。 每个被 cache 的 block 都存储在一个 `struct buf`（[kernel/buf.h:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/buf.h#L1)）中。 `struct buf` 本身有一个 lock 栏位，用来确保同一时间只有一个 process 能使用该 block

然而，这样的 lock 并不够：假如某个 block 还不在 cache 中，且有两个 process 同时想使用它，这时因为还没有对应的 `struct buf` 存在，所以根本没东西可锁。 xv6 通过一个额外的 lock（`bcache.lock`）来保护「目前已被 cache 的 block 的集合」

任何需要查找某个 block 是否已被 cache（例如 `bget`, [kernel/bio.c:59](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L59)），或修改 cache 集合的程序，都必须持有 `bcache.lock`。 一旦找到了所需的 block 与其对应的 `struct buf`，即可释放 `bcache.lock`，改为锁住该 block 对应的 `struct buf`。 这是一个常见的模式：一把锁保护整个集合，另一把锁保护集合中的个别项目

一般来说，获取 lock 的函数会在同一个函数中释放该 lock。 但更精确的说法是：lock 会在一段需要表现为原子操作的序列开始时被获取，而在这段序列结束时释放。 如果这段序列的开始与结束发生在不同的函数、不同的 thread，甚至不同的 CPU 上，那么 lock 的获取与释放也必须跨越这些边界。 lock 的目的是阻止其他用户介入，而不是将数据绑定在某个特定 agent 上

一个例子是 `yield` 中的 `acquire`（[kernel/proc.c:512](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L512)），其对应的释放发生在 scheduler thread，而非原本调用者所在的 process。 另一个例子是 `ilock` 中的 `acquiresleep`（[kernel/fs.c:293](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/fs.c#L293)），该程序在读取硬盘时可能会 sleep，而醒来时可能已经在不同的 CPU 上了，因此 `acquire` 与 `release` 可能会发生于不同 CPU

释放内部含有 lock 的对象的操作会非常敏感，因为单纯持有该 lock 并不足以保证释放行为是正确的。 典型的问题是：当某个 thread 正在 `acquire` 该 lock 等待使用这个对象时，若此时另一个 thread 将该对象释放，那么这样也会隐含地释放掉 lock 本身，导致正在等待的 thread 发生错误

对此的一种解法是跟踪这个对象的引用计数（reference count），只有当最后一个引用消失时才释放该对象。 请参见 `pipeclose`（[kernel/pipe.c:59](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/pipe.c#L59)）作为范例； 其中 `pi->readopen` 与 `pi->writeopen` 用来跟踪是否仍有 file descriptor 指向该 pipe

```c
void
pipeclose(struct pipe *pi, int writable)
{
  acquire(&pi->lock);
  if(writable){
    pi->writeopen = 0;
    wakeup(&pi->nread);
  } else {
    pi->readopen = 0;
    wakeup(&pi->nwrite);
  }
  if(pi->readopen == 0 && pi->writeopen == 0){
    release(&pi->lock);
    kfree((char*)pi);
  } else
    release(&pi->lock);
}
```

通常我们会在一连串针对一组相关数据的读写操作的外围加上 lock，这样可以保证其他 thread 在使用时，只会看到已完整更新的数据（只要它们自己也有加锁）。 那么如果只需要写入一个共享变量该怎么办？ 例如 `setkilled` 和 `killed`（[kernel/proc.c:619](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L619)）在操作 `p->killed` 这个变量时就有使用 lock。 若没有加锁，可能会发生一个 thread 写入 `p->killed` 的同时，另一个 thread 正在读取它，这就构成一个 race

根据 C 语言的标准，这类 race 属于 undefined behavior，也就是该程序可能会当机或生成错误结果（参见 cppreference 上的「[Threads and data races](https://en.cppreference.com/w/c/language/memory_model.html)」）。 加上 lock 就能避免这样的 race，也就避免了 undefined behavior

race 会破坏程序正确性的其中一个原因是，如果没有使用 lock 或其他等效机制，编译器可能会生成与原始 C 代码逻辑完全不同的机器码来进行内存的读写。 例如，一个 thread 调用 `killed` 时，其机器码可能会将 `p->killed` 的值复制到寄存器中，之后就只读这个缓存的值； 这将导致该 thread 永远看不到其他 thread 对 `p->killed` 所做的任何写入。 而使用 lock 则能避免这种缓存行为

## 9.2 Lock-like patterns

在 xv6 的许多地方，系统会以引用计数或 flag 的方式，模拟类似 lock 的行为，用来表示某个对象目前处于已分配（allocated）状态，因此不应该被释放或重新使用。 像是 process 的 `p->state` 就具有这种效果，而 `file`、`inode` 与 `buf` 结构中的引用计数也是如此。 虽然这些 flag 或引用计数本身都会受到 lock 保护，但真正防止对象被过早释放的，其实是这些引用计数本身

文件系统会使用 `struct inode` 的引用计数作为一种共享 lock，这样可以让多个 process 同时持有，以避免那些使用一般 lock 可能发生的 deadlock。 例如，`namex` 中的循环（[kernel/fs.c:652](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/fs.c#L652)）会依序对每个 path component 所对应的目录加锁

而 `namex` 在每次循环结束时必须释放该锁，否则若 path 中包含 `.`（例如 `a/./b`），就可能会导致自身 deadlock。 也可能与另一个正在查找 `..` 的 thread 发生死结。 如第八章中解释的，解法是让循环将目前的 directory inode 带到下一轮，但只增加引用计数而不加锁

有些数据项目在不同时期会受到不同机制的保护，有时甚至并非通过明确的 lock，而是通过 xv6 代码的结构隐含地避免了并发访问。 例如，当一个 physical page 处于 free 状态时，它会受到 `kmem.lock` 的保护（[kernel/kalloc.c:24](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kalloc.c#L24)）。 若该 page 被分配成一个 pipe（[kernel/pipe.c:23](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/pipe.c#L23)），它则会受到另一把 lock（`pi->lock`）的保护。 如果这个 page 被分配为某个新 process 的 user memory，那它甚至完全没有受到任何 lock 保护，而是仰赖分配器的行为保证：只要 page 尚未被释放，就不会分配给其他 process，因此也不会有并发使用的问题

一个新 process 的 memory 拥有权的转移过程相当复杂：一开始由 parent 在 `fork` 中分配与操作，然后由 child 使用，最后当 child 结束时，又回到 parent 手上，并交由 `kfree` 释放。 这里可学到两点：第一，某个数据对象在生命周期的不同阶段，可能会以不同方式被保护； 第二，这种保护有时可能是隐含的结构性设计，而非显式地使用 lock

最后一个类似 lock 的例子是：在调用 `mycpu()`（[kernel/proc.c:83](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L83)）时需要关闭中断。 关闭中断可以让该段程序在执行期间，不会被 timer interrupt 打断，进而强制发生 context switch，将 process 切换到另一颗 CPU

## 9.3 No locks at all

在 xv6 中，有些地方会共用可变数据而完全不使用 lock。 其中一个例子是 spinlock 的实现，但你也可以认为 RISC-V 的 atomic 指令本质上是由硬件提供的 lock。 另一个例子是 `main.c` 中的 `started` 变量（[kernel/main.c:7](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/main.c#L7)），用来在 CPU 0 完成 xv6 的初始化前，阻止其他 CPU 开始执行； 其中的 `volatile` 关键字可以确保编译器会确实生成 load 与 store 指令

xv6 中有些情况会发生这样的情形：一个 CPU 或 thread 写入某些数据，另一个 CPU 或 thread 则会去读取这些数据，但却没有任何专门的 lock 来保护这些数据。 例如在 `fork` 中，parent process 会写入 child 的 user memory page，而 child（可能在另一个 thread 或 CPU 上）会去读取这些 page； 这些 page 没有被明确地使用 lock 保护

严格来说这不属于 locking 问题，因为 child 是在 parent 完成写入之后才开始执行的。 不过这会形成一个潜在的内存顺序问题（详见第六章），因为在没有 memory barrier 的情况下，不能保证某个 CPU 会看到另一个 CPU 的写入。 不过，由于 parent 会释放 lock，而 child 在启动时会获取 lock，因此 `acquire` 与 `release` 中隐含的 memory barrier 可保证 child 所在的 CPU 能够看到 parent 所写入的内容

## 9.4 Parallelism

Lock 的主要目的在于抑制并行性，以确保正确性。 然而，效能同样也很重要，因此内核设计者往往必须思考如何在使用 lock 的同时，既能达到正确性，也能保有一定程度的平行性。 虽然 xv6 并不以高效能为设计目标，但仍值得分析哪些 xv6 的操作能够并行执行，哪些操作会因 lock 发生冲突

xv6 中的 pipe 是一个平行性设计的还不错的例子。 每个 pipe 都有自己的 lock，因此不同的 process 可以在不同的 CPU 上并行地读写不同的 pipe。 然而，对于同一个 pipe，读写双方仍必须互相等待对方释放 lock； 他们无法同时对同一个 pipe 进行读或写。 此外，若读取一个空的 pipe（或写入一个已满的 pipe），其也会被阻塞，不过这是因为同步逻辑的设计，而非 lock 本身所造成的

context switch 是一个更复杂的例子。 如果两个 kernel thread 各自在自己的 CPU 上执行，可以同时调用 `yield`、`sched` 和 `swtch`，而这些调用能够并行执行。 每个 thread 都会持有一把锁，但这些锁彼此不同，因此他们不需互相等待。 不过一旦进入 `scheduler`，这两个 CPU 就可能会在搜索 `RUNNABLE` 的 process 时发生 lock 冲突。 换句话说，xv6 在 context switch 中可以从多核受益，但实际上得到的效能提升可能不如理想

另一个例子是不同 CPU 上的 process 同时调用 `fork`。 这些调用在某些地方需要互相等待，例如 `pid_lock`、`kmem.lock`，还有每个 process 自己的锁，这些都会在搜索 process table 中的 `UNUSED` process 时用到。 不过另一方面，这两个 fork 操作也可以完全平行地进行 user memory page 的复制与 page table 的初始化

上述各例中的 locking 设计，在某些情况下都牺牲了一定的平行效能。 不过每一种情况其实都可以通过更精密的设计来提升平行性。 至于是否值得这样做，则取决于许多细节：像是相关操作的调用频率、程序在持有受争用锁时花费的时间、有多少颗 CPU 同时在执行可能冲突的操作，以及是否有其他更严重的瓶颈存在。 要判断某个 locking 设计是否会造成效能问题，或一个新设计是否真的更好，往往并不容易，因此通常需要通过实际负载下的效能测量来决定

## 9.5 Exercises

1. 修改 xv6 的 pipe 实现，使得同一个 pipe 上的读与写可以在不同的 CPU 上平行进行
2. 修改 xv6 的 `scheduler()`，以减少当不同 CPU 同时搜索可执行 process 时所生成的 lock contention
3. 移除 xv6 中 `fork()` 实现里的一部分序列化操作
