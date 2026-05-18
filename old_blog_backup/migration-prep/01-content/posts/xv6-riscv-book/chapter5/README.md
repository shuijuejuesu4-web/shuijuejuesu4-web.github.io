---
title: xv6 riscv book chapter 5：Interrupts and device drivers
date: 2025-08-01
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---
# xv6 riscv book chapter 5：Interrupts and device drivers

driver 是操作系统中负责管理特定装置的代码：它会设置装置的硬件、命令装置执行操作、处理装置生成的中断，并与那些可能正在等待该装置 I/O 的 process 交互。 driver 的代码常常很棘手，因为它与其所管理的装置是并行执行的。 此外，driver 还必须理解装置的硬件接口，而这些接口可能很复杂，也可能没有良好文件说明

需要操作系统处理的装置通常可以将其设置成会生成 interrupt，kernel 的 trap handler 代码会辨认出装置发出的 interrupt，并调用对应 driver 的 interrupt handler； 在 xv6 里由 `devintr`（[kernel/trap.c:185](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)） 来去调用对应的 driver

许多 device driver 的程序会在两种情境下执行：一种是称为 top half 的部分，会在 process 的 kernel thread 里执行； 另一种是称为 bottom half 的部分，会在 interrupt 发生时执行。 top half 会被像是 read 和 write 这类要让装置进行 I/O 的 system call 调用。 top half 的程序可能会要求硬件开始某个操作（例如请硬盘读一个区块），然后等待操作完成，接著会生成一个 interrupt。 driver 的 interrupt handler，也就是 bottom half，会判断是哪个操作完成了，并在必要时唤醒等待的 process，然后告诉硬件可以开始执行下一个等待中的操作了

## 5.1 Code: Console input

console driver（[kernel/console.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c)）是一个简单展示 driver 结构的范例，其通过接在 RISC-V 上的 UART 序列埠硬件来接收人类输入的字元。 console driver 会一次累积一整行输入，并处理像 backspace 和 control-u 这类的特殊字元。 像 shell 这样的 user process，会通过 `read` 系统调用从 console 读取一整行的输入。 当你在 QEMU 中对 xv6 输入时，你的按键会经由 QEMU 模拟的 UART 硬件发送给 xv6

这个 driver 所操作的 UART 硬件，是由 QEMU 模拟出来的 16550 晶片<sup>[[1]](#1)</sup>。 在真实的电脑上，16550 晶片通常用来控制 RS232 序列连线，连接到终端机或另一台电脑。 而在执行 QEMU 时，它则连接到你的键盘与显示器

以软件的角度来看，UART 硬件是由 memory-mapped 的控制寄存器组成的。 也就是说，RISC-V 硬件会将某些实体地址对应到 UART 装置，使得对那些地址的 load 与 store 操作实际上是与硬件交互，而不会访问 RAM。 UART 的 memory-mapped 地址从 `0x10000000` 开始，也就是 `UART0`（[kernel/memlayout.h:21](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/memlayout.h#L21)）

UART 有一些控制寄存器，每个寄存器的宽度都是一个 byte，它们相对于 `UART0` 的位移定义在（[kernel/uart.c:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L22)）里面。 例如 `LSR` 寄存器里的某些 bit 表示是否有输入字元等著被软件读取，这些字元（如果有的话）可以从 RHR 寄存器读出。 每次读取后，UART 硬件会从它内部的 FIFO 中删除该字元，当 FIFO 清空后，其会一并清除 `LSR` 中的 ready bit。 UART 的发送逻辑与接收逻辑几乎是独立的，如果软件写入一个 byte 到 `THR`，UART 就会发送该 byte

xv6 的 `main` 会调用 `consoleinit`（[kernel/console.c:182](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L182)）来初始化 UART 硬件。 这段程序会设置 UART，让它在接收到每个输入 byte 时生成接收中断（receive interrupt），以及在每个输出 byte 发送完成时生成发送完成中断（transmit complete interrupt）（[kernel/uart.c:53](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L53)）

xv6 的 shell 会通过一个由 init.c 所打开的 file descriptor（[user/init.c:19](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/init.c#L19)）来从 console 读取数据。 对 `read` system call 的调用会一路进入 kernel，最后到达 `consoleread`（[kernel/console.c:80](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L80)）。 `consoleread` 会等待输入通过中断抵达，并将输入暂存到 `cons.buf` 中，然后将输入复制到 user space，并在整行输入完成后才返回给 user process。 若用户尚未输入完整的一行，任何调用 `read` 的 process 都会停在 `sleep`（[kernel/console.c:96](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L96)）调用中，第七章节会再详细说明 `sleep` 的运行

::: tip  
`read` 系统调用的实现为 `sys_read`，内部最后会调用 `fileread`，而 `fileread` 会根据 `file` 这个结构体内的成员 `major` 来判断要怎么读取这个 file descriptor：

```c
// Read from file f.
// addr is a user virtual address.
int
fileread(struct file *f, uint64 addr, int n)
{
  ...
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(1, addr, n);
  ...
  return r;
}
```

而在 `consoleinit` 中会将它接到 `consoleread`：

```c
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```

而前面有说 init.c 会开一个 file descriptor 来从 console 读取数据，其底层就是对应到 `CONSOLE`。 因此可知读取 console 的流程为：`read` → `fileread` → `consoleread`  
:::

当用户输入一个字元时，UART 硬件会要求 RISC-V CPU 生成一个中断，这会启动 xv6 的 trap handler。 trap handler 接著会调用 `devintr`（[kernel/trap.c:185](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)），它会查看 RISC-V 的 `scause` 寄存器，以判断该中断是否来自外部装置。 接著，它会向名为 PLIC 的硬件单元查询是由哪个装置生成的中断（[kernel/trap.c:193](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L193)）； 对于 UART 生成的中断，`devintr` 会调用 `uartintr`

`uartintr`（[kernel/uart.c:177](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L177)）会从 UART 硬件中读取所有尚未处理的输入字元，并将这些字元交给 `consoleintr`（[kernel/console.c:136](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L177)）； 它本身不会等待更多输入，因为未来有新输入时还会再次触发新的中断。 `consoleintr` 的工作是将这些输入字元累积到 `cons.buf` 中，直到整行输入完成为止。 `consoleintr` 会特别处理 backspace 与其他一些特殊字元

当输入遇到 newline 时，`consoleintr` 会唤醒等待中的 `consoleread`（如果有的话）。 一旦被唤醒，`consoleread` 就会发现 `cons.buf` 中已经有一整行输入，接著它会把这行数据复制到 user space，然后通过系统调用的机制将控制权返回给 user space

## 5.2 Code: Console output

对已连接到 console 的 file descriptor 所做的 `write` 系统调用，最终会到达 `uartputc`（[kernel/uart.c:87](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L87)）。 driver 维护了一个输出缓冲区（`uart_tx_buf`），使得执行写入的 process 不需要等待 UART 发送完毕； 相反地，`uartputc` 会将每个字元加入缓冲区，然后调用 `uartstart` 来启动 UART 的发送（如果尚未开始的话），接著就直接返回。 只有当缓冲区满了的情况下 `uartputc` 才会停下来等待

每当 UART 发送完一个 byte，它就会生成一个中断。 `uartintr` 会调用 `uartstart`，这个函数会检查 UART 是否真的完成发送，然后把下一个缓冲区中的输出字元交给 UART 发送。 因此，如果某个 process 一次写入多个 byte 到 console，通常第一个 byte 是由 `uartputc` 调用 `uartstart` 发送出去的，其余在缓冲区里的 byte 则会由 `uartintr` 在每次中断到来时持续调用 `uartstart` 发送出去

通常我们会利用「缓冲区」与「中断」来让「装置活动」与「process 活动」解耦。 即使没有 process 正等著要读数据，console driver 也能先处理输入，因此之后的 `read` 调用仍然能读到这些数据。 同样地，process 也能够直接输出数据，而不用等待装置完成发送。 这种解耦能提升效能，因为它允许 process 在进行装置 I/O 的同时继续执行，而当装置速度很慢（像 UART）或需要即时反应（像 echo 指定字元）时，这点特别重要。 这种设计理念有时被称为 I/O 并行（I/O concurrency）

::: tip  
当我们在 shell 中敲下 `a` 键，其流程大概如下：

1. shell 内的 `getcmd` 印出 `$` 提示后会调用 `gets`
    - 其底层会调用 `consoleread`，由于 FIFO 为空，因此会进入 sleep 状态
2. QEMU 把 `a` 推入 UART FIFO
3. UART 设置 `LSR_RX_READY` bit → PLIC 送 IRQ
4. CPU trap → trap vector → trap handler → `devintr()`
5. `devintr()` → `uartintr()` → 读 FIFO（通过 `uartgetc()`）
6. `uartintr()` 调用 `consoleintr('a')`
7. `consoleintr()` 把 `'a'` 放入 `cons.buf` 内
    - 由于没有 `'\n'`，所以不会唤醒 `consoleread()`，立即 return
8. 回到被抢占前正在执行的进程

其中：

- 第 2、3 步都是硬件（QEMU）处理的
- 第三步是要走 `kernelvec` 还是 `uservec`，取决于发生 trap 时 CPU 正在哪个 mode 下
  - 例如如果当时刚好 shell 调用了 `read()` 并正睡在 `sleep()` 中，那 CPU 就还处在 kernel space，因此会走 `kernelvec`
  - 而如果 CPU 已经切去另一个 user process 了，那就会走 `uservec`
  - 但不管走哪个，最后都会执行到 `devintr`  
:::

## 5.3 Concurrency in drivers

你可能已经注意到在 `consoleread` 和 `consoleintr` 中有调用 `acquire`，这些调用会获取锁，以保护 console driver 的数据结构不被并行访问。 在这里有三种并行风险：第一是两个不同 CPU 上的 process 可能同时调用 `consoleread`； 第二是硬件可能在某个 CPU 执行 `consoleread` 时触发 console（实际上是 UART）中断，打断该 CPU； 最后是中断可能发生在另一个 CPU 上，而此时某个 CPU 正在执行 `consoleread`。 第六章会说明如何使用锁来避免这些情况导致错误结果

driver 在处理并行时还有另一个需要注意的情况，那就是某个 process 可能正在等待某个装置的输入，但用来通知输入到达的中断可能是在另一个 process（或根本没有 process）执行时生成的。 因此，interrupt handler 无法预期它中断的 process 或代码是什么，例如 interrupt handler 无法直接使用当前 process 的 page table 去调用 `copyout`。 通常 interrupt handler 只会做一些简单的工作（例如将输入数据复制到缓冲区），然后唤醒 top-half 的代码来完成后续处理

::: tip  
interrupt handler 不能依赖于当前 CPU 所执行的 context，因为它可能与触发中断的真正程序无关。 这是因为：

- 中断可能在任意时间点、任意 CPU 上发生
- 没有保证当前在执行的就是那个等待输入的 process。 所以 interrupt handler 常只做 minimal 的事（如复制数据进 buffer），然后唤醒后续能正确处理的代码。这就是 top-half / bottom-half 设计的意义  
:::

## 5.4 Timer interrupts

xv6 使用 timer interrupt 来维持系统对「目前时间」的认知，并在多个 compute-bound 的 process 之间进行切换。 timer interrupt 由接在每个 RISC-V CPU 上的时钟硬件所生成，xv6 会设置每个 CPU 的时钟硬件，让它定期对该 CPU 生成中断

start.c 中的代码（[kernel/start.c:53](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/start.c#L53)）会设置一些控制用的 bit，使 supervisor mode 可以访问 timer 控制寄存器，接著会请求第一个 timer interrupt。 `time` 控制寄存器会以固定速率自动递增，这提供了「目前时间」的概念。 `stimecmp` 寄存器中则存放一个时间点，当 `time` 递增到该时间点时，CPU 就会生成 timer interrupt。 换句话说，如果将 `stimecmp` 设为 `time` 加上某个值 `x`，就表示在 `x` 个时间单位之后会生成一个 interrupt。 对于 QEMU 的 RISC-V 模拟器来说，1000000 个时间单位大约等于 0.1 秒

timer interrupt 会像其他装置中断一样，经由 `usertrap` 或 `kerneltrap` 和 `devintr` 发送进来。 timer interrupt 发生时，`scause` 的低位元会被设为 5； trap.c 中的 `devintr` 侦测到这种情况时，会调用 `clockintr`（[kernel/trap.c:164](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L164)）。 `clockintr` 会将 ticks 递增，以让 kernel 能够跟踪时间流逝，这个递增动作只会在其中一个 CPU 上执行，以避免多个 CPU 同时让时间变快。 `clockintr` 也会唤醒那些正在 `sleep` 系统调用中等待的 process，并写入新的 `stimecmp` 值来调度下一次的 timer interrupt

`devintr` 若遇到 timer interrupt，会返回 2，以通知 `kerneltrap` 或 `usertrap` 应该调用 `yield`，从而让 CPU 能够在多个可执行的 process 之间进行切换

kernel 代码可能会在执行期间被 timer interrupt 打断，并因为 `yield` 而进行 context switch，这也是为什么 `usertrap` 初期的代码会小心地先存储像 `sepc` 这类的状态，然后才打开中断。 这也意味著在撰写 kernel 代码时必须考虑到会有这类 context switch 发生，它可能会在没有预警的情况下从某个 CPU 被切换到另一个 CPU 上执行

## 5.5 Real world

和许多操作系统一样，xv6 在 kernel 执行期间也允许中断发生，甚至允许通过 `yield` 进行 context switch。 这么做的原因是希望即使在执行一些耗时且复杂的系统调用时，也能维持良好的反应速度。 不过，如前所述，在 kernel 中允许中断也会导入一些额外的复杂性； 因此，有些操作系统选择只在执行 user code 时才允许中断

要完整支持一台典型电脑上所有装置，是一件非常繁琐的事，因为装置种类繁多，每个装置又有很多功能，而且装置与 driver 之间的通讯协定往往很复杂，甚至缺乏良好文件。 在许多操作系统中，driver 的代码量往往超过了 kernel 本身

UART driver 是通过读取 UART 控制寄存器，每次取回一个 byte 的方式来获取数据的； 因为是由软件主动驱动数据搬移的，所以这种模式称为「programmed I/O」。 programmed I/O 实现简单，但速度太慢，不适合高数据速率的应用。 需要高速搬移大量数据的装置通常会使用 direct memory access（DMA）的方式，例如现代的硬盘与网络装置都使用 DMA。 DMA 装置硬件会直接把接收到的数据写入 RAM，也会从 RAM 中读取要发送的数据。 对于 DMA 装置而言，driver 会先在 RAM 中准备好数据，然后只需写一次控制寄存器来告诉装置处理这些数据即可

当无法预测装置需要处理的时间点，但其频率又不太高时，使用中断是合理的。 但中断的 CPU 成本很高，因此像网络或硬盘这种高速装置会使用一些技巧来减少中断需求。 一种技巧是针对一整批收送请求只生成一次中断。 另一种技巧则是完全关掉中断，由 driver 定期检查装置是否需要处理，这种技巧称为 polling。 polling 适用于装置操作频率很高的情况，但若装置大多时间空闲，polling 会浪费大量 CPU 资源。 有些 driver 会根据装置目前的负载，动态地在 polling 和中断两种模式之间切换

UART driver 会先将收到的数据复制到 kernel 的缓冲区，再复制到 user space。 这样的作法在低数据速率的环境下是合理的，但对于生成或消耗数据速度很快的装置，这样的两次复制会显著影响效能。 有些操作系统能够直接在 user-space buffer 和装置硬件之间搬移数据，这通常需要通过 DMA 来完成

如第一章所述，console 在应用程序看来就像是一般的文件，应用程序会使用 `read` 和 `write` 系统调用来进行输入与输出。 不过，有些装置功能无法通过标准的文件系统调用来表示（例如，打开或关闭 console driver 的 line buffer）。 Unix 操作系统会提供 `ioctl` 系统调用来处理这类情况

有些电脑用途要求系统必须在固定时间内做出反应。 例如在重视安全性的系统中，错过期限可能会导致灾难。 xv6 不适合用在硬即时（hard real-time）的情境，硬即时系统的操作系统通常会以函数库的形式与应用程序链接，让系统能分析出最坏情况的反应时间 。xv6 也不适合用于软即时（soft real-time）应用，也就是偶尔错过期限可以接受的情境，因为 xv6 的调度器太过简单，而且有些 kernel 的程序路径会在长时间内关闭中断

::: tip  
硬即时系统（例如医疗、飞控）要求绝不能 miss deadline，而软即时系统（例如影音播放）容许偶尔 miss，但仍要求反应快。 但 xv6 的调度器过于简单，且某些 kernel 区段会 disable interrupt 太久，这让它无法保证 deadline，故两种都不适合  
:::

## 5.6 Exercises

1. 修改 uart.c，让它完全不使用中断。 你可能也需要修改 console.c
2. 为网卡新增一个 driver

## Bibliography

- <a id="1">[1]</a>：Martin Michael and Daniel Durich. The NS16550A: UART design and application considerations. http://bitsavers.trailing-edge.com/components/national/_appNotes/AN-0491.pdf, 1987.
