---
title: "xv6 riscv book chapter 4：Traps and system calls"
description: "xv6 riscv book chapter 4：Traps and system calls"
publishDate: 2025-07-31
tags: [OS, xv6, risc-v]
category: "OS-xv6"
draft: false
comment: true
---
# xv6 riscv book chapter 4：Traps and system calls

有三种类型的事件会使 CPU 暂停正常的指令执行流程，并强制转移控制权到一段专门处理该事件的代码。 第一种情况是系统调用，当用户程序执行 `ecall` 指令时，会向 kernel 提出对应的要求。 第二种情况是例外（exception）：某条指令（无论是来自用户或 kernel ）执行了非法操作，例如除以零或使用无效的虚拟地址。 第三种情况是装置中断（interrupt），如某个装置发出信号表示它需要被处理，例如硬盘完成某次读写请求的时候

本书将上述这些情况统称为「trap」。 通常，发生 trap 时正在执行的代码之后需要能够继续执行，并且不应该察觉到任何特殊的事情发生了。 也就是说，我们通常希望 trap 是透明的； 这一点在处理装置中断时尤其重要，因为被中断的代码通常不会预期到被打断。 一般的处理流程是：trap 发生后控制权会转移到 kernel； kernel 会存储寄存器与其他状态，以便之后能够恢复执行； 接著 kernel 会执行对应的处理程序（例如系统调用的实现或装置驱动程序）； 然后 kernel 会还原先前存储的状态并从 trap 返回； 最后原本的代码会从中断处继续执行

xv6 在 kernel 中处理所有的 trap，trap 并不会交由用户程序处理。 将 trap 交由 kernel 处理对于系统调用来说是理所当然的。 而将中断交由 kernel 处理也是合理的，因为有隔离的需求，所以只有 kernel 能够操作装置，而且 kernel 也提供了一个便利的机制，能够让多个进程共享装置。 对于例外来说交由 kernel 处理也合理，因为 xv6 对于所有来自 user space 的例外都会以终止该程序作为响应

xv6 的 trap 处理流程分为四个阶段：第一阶段是 RISC-V CPU 执行的硬件动作； 第二阶段是一些汇编语言指令，用来为 kernel 的 C 代码做准备； 第三阶段是一个 C 函数，它决定该如何处理这个 trap； 第四阶段则是执行对应的系统调用或装置驱动服务常式

尽管这三种 trap 类型有不少共通性，理论上 kernel 可以用一条通用的路径来处理所有 trap，但实务上将其区分为两种情况会更方便：来自 user space 的 trap，与来自 kernel 空间的 trap。 负责处理 trap 的 kernel 代码（不论是组语或 C）通常被称为「handler」； 而最先执行的那几条 handler 指令通常以汇编语言撰写，有时会被称为「vector」

:::tip
这个「vector」也有一些别的名字，如「trap prologue」或「trap entry」等，不过这些应该是口语上的名称，而不是一个正式的名词。 但从另外两个名字你应该可以理解 xv6 中的「vector」就是一个统一的入口，发生 trap 时会先进入「vector」，然后再根据 trap 的种类去调用对应的 handler

具体而言，在 xv6 中有两个「vector」：`uservec` 与 `kernelvec`，它们都是用组语写的函数，两者都只会先将必要的信息存起来，然后调用对应的 C 函数 `usertrap` 与 `kerneltrap`，但这两个 trap handler 的设计不太一样，后面会再提到  
:::

## 4.1 RISC-V trap machinery

每个 RISC-V CPU 都有一组控制寄存器，kernel 会写入这些寄存器以告诉 CPU 该如何处理 trap，并且 kernel 也可以读取这些寄存器来得知 trap 的相关信息，RISC-V 的官方文件中有完整的说明<sup>[[1]](#1)</sup>。 riscv.h（[kernel/riscv.h:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L1)）中包含了 xv6 使用的相关定义。 以下是几个最重要的寄存器简介：

- `stvec`：  
  - kernel 会在这里写入 trap handler 的地址； 当发生 trap 时，RISC-V 会跳到 `stvec` 所指定的地址执行处理该 trap 的 handler
- `sepc`：  
  - 当发生 trap 时，RISC-V 会将当下的程序计数器（`pc`）存储在此处（因为 `pc` 随即会被 `stvec` 的值覆盖）。 `sret`（从 trap 返回的指令）会将 `sepc` 的内容复制回 `pc`。 kernel 也可以通过写入 `sepc` 来控制 `sret` 返回的位置
- `scause`：  
  - RISC-V 会在此处写入一个数值，描述这次 trap 的原因
- `sscratch`：  
  - trap handler 代码会使用 `sscratch` 来协助避免用户的寄存器尚未被存储前就被覆写
- `sstatus`：  
  - 此寄存器中的 SIE 位元控制装置中断是否启用。 如果 kernel 清除此位元，RISC-V 将会延后处理装置中断直到 kernel 再次设置它。 SPP 位元表示这次 trap 是从 user mode 还是 supervisor mode 进入的，并决定 `sret` 返回的模式

上述的这些都是 supervisor mode 下与处理 trap 有关的寄存器，并且在 user mode 中无法读写这些寄存器。 multi-core 晶片上的每个 CPU 都各有一组这些寄存器，并且任一时刻下都可能有多个 CPU 同时在处理 trap

当需要强制进入 trap 时，RISC-V 硬件会对所有 trap 类型执行以下动作：

1. 如果这次 trap 来自装置中断，且 `sstatus` 的 SIE 位元为清除状态，则不执行以下步骤
2. 清除 `sstatus` 中的 SIE 位元以关闭中断
3. 将 `pc` 的值复制到 `sepc`
4. 将当前的模式（user 或 supervisor）存储在 `sstatus` 的 SPP 位元中
5. 将 `scause` 设置为此次 trap 的原因
6. 将执行模式设为 supervisor
7. 将 `stvec` 的值复制到 pc
8. 从新的 `pc` 开始执行

请注意，CPU 不会在 trap 发生时自动切换到 kernel 的 page table，也不会切换到 kernel stack，除了 `pc` 外也不会存储任何其他寄存器。 这些任务必须由 kernel 的软件来执行，CPU 在处理 trap 时只做最少的工作，主要是为了让软件有更多弹性； 例如，有些操作系统会在特定情况下省略切换 page table，以提升 trap 的效能

值得思考的是我们能否省略上述步骤中的某些部分，以更快速的处理 trap。 虽然在某些情况下简化流程是可行的，但上方大多数的步骤若被省略会造成危险。 例如，假设 CPU 没有切换程序计数器，那么来自 user space 的 trap 就可能在仍执行用户指令的情况下进入 supervisor 模式。 这些用户指令可能会破坏用户与 kernel 之间的隔离，例如修改 `satp` 寄存器指向允许访问整个物理内存的 page table。 因此，CPU 切换到 kernel 所指定的指令地址（即 `stvec`）是非常重要的

## 4.2 Traps from user space

xv6 会根据 trap 发生时是在 kernel space 中还是 user space 中而采取不同的处理方式。 这段会讲述从 user code 发出的 trap 的流程； 至于 kernel code 发出的 trap，则会在第 4.5 节中说明

当线程正在 user space 执行时，如果 user program 发出了系统调用（通过 `ecall` 指令）、做了不合法的操作，或有装置中断发生，就可能会发生 trap。 从 user space 发出的 trap，其高阶的处理路径为：先进入 `uservec`（[kernel/trampoline.S:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L22)），接著进入 `usertrap`（[kernel/trap.c:37](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L37)）； 在处理完要返回 user space 时，会先经过 `usertrapret`（[kernel/trap.c:90](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L90)），最后再通过 `userret`（[kernel/trampoline.S:101](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L101)）回到 user program

xv6 的 trap 处理机制在设计上有个主要限制：RISC-V 硬件在触发 trap 时并不会自动切换 page table。 这表示 `stvec` 中指向的 trap handler 地址，必须在 user page table 中有一个有效的映射，因为 trap 发生时仍是使用 user 的 page table 来执行。 此外，xv6 的 trap handler 还需要切换到 kernel 的 page table； 而为了让 trap handler 在切换后能继续执行，kernel page table 也必须对 `stvec` 所指向的 handler 有一份映射

xv6 通过 trampoline page 来满足这些需求。 trampoline page 包含了 `uservec`，也就是 `stvec` 所指向的 trap handler。 xv6 会在每个 process 的 page table 中，将 trampoline page 映射到 `TRAMPOLINE` 这个地址； 这个地址在虚拟地址空间的最顶端，因此会高于 program 自己所使用的内存范围

同时，trampoline page 也会在 kernel 的 page table 中被映射到相同的 `TRAMPOLINE` 地址，详情可参考图 2.3 和图 3.3。 因为 trampoline page 有被映射进 user page table，且因为其在 kernel 的 page table 中也有映射，所以在发生 trap 而切换到 supervisor mode 时，handler 切换 page table 后仍能从该处继续执行

:::tip
这个被称为 trampoline 的共享 page 会被映射进所有 process 的 user page table 和 kernel page table，而且都是映射到同一个虚拟地址。 这样在 trap 发生时（使用 user page table）能进入 trampoline，而在 handler 中切换到 kernel page table 之后也不会失效，保证了执行的连贯性

Remark：

- root kernel page table 是全域唯一的，整个系统中只有一张
  - 任何 hart 只要处于 supervisor mode、且在执行 kernel 代码时，`satp` 就指到这张表
  - 使用 direct mapping（VA == PA）
- root user page table 则是每个 process 都自己有一张
  
:::

`uservec` 这段 trap handler 的代码写在 trampoline.S（[kernel/trampoline.S:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L22)）中。 当 `uservec` 开始执行时，32 个寄存器都还保留著被中断的 user code 的值。 这 32 个值需要被存到内存中，好让 kernel 在返回 user space 之前可以将它们还原。 但要把东西存到内存中，势必得有一个寄存器来存放内存的地址，然而此时却没有任何通用寄存器可以用，对此 RISC-V 提供了一个解法：`sscratch` 寄存器。 `uservec` 开头的 `csrw` 指令会先把 `a0` 存进 `sscratch`，这样 `uservec` 就可以暂时直接使用 `a0` 了

`uservec` 接下来要做的事，就是把 32 个用户寄存器全部存起来。 kernel 为每个 process 都分配了一个 page 给 `trapframe` 结构体，用来保存这些寄存器的值（[kernel/proc.h:43](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.h#L43)）。 由于现在 `satp` 还是指向 user page table，`uservec` 访问内存时，trapframe 必须要有对应的 user space 映射。 xv6 会在每个 process 的 user page table 中，把 `trapframe` 映射到 `TRAPFRAME` 这个虚拟地址，而这个地址就在 `TRAMPOLINE` 的正下方。 此外，process 的 `p->trapframe` 指针也会指向该 trapframe，不过是指向其实体地址，这样 kernel 就能通过 kernel page table 访问它

因此 `uservec` 会将 `TRAPFRAME` 的地址加载到 `a0` 中，并将所有 user 寄存器的值存到那个位置，其中也包含刚刚存入 `sscratch` 内的 user 的 `a0` 值。 `trapframe` 中会包含当前 process 的 kernel stack 地址、目前 CPU 的 hartid、`usertrap` 函数的地址，以及 kernel page table 的地址。 `uservec` 会从中读出这些信息，接著把 `satp` 切换成 kernel page table，然后跳到 `usertrap`

:::tip
`TRAMPOLINE` 和 `TRAPFRAME` 是两个已经被写死的 macro（[kernel/memlayout.h:44](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h#L44),[59](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h#L59)），换句话说每个 process 中 `TRAMPOLINE` 和 `TRAPFRAME` 的虚拟地址都是相同的。 区别在于对于不同的 process，`TRAMPOLINE` 都会对应到同一个 page frame，但对于不同的 process，`TRAPFRAME` 则会对应到不同的 page frame  
:::

`usertrap` 的工作是判断 trap 的原因、处理它，然后返回（[kernel/trap.c:37](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L37)）。 它一开始会修改 `stvec`，这样之后 kernel 里如果再次发生 trap，就会进入 `kernelvec` 而不是 `uservec`。 接著会存储 `sepc` 寄存器（也就是用户程序的 PC），因为 `usertrap` 有可能调用 `yield` 去切换到其他 process 的 kernel thread，而那个 process 在切换回 user space 时会改写 `sepc`

如果该 trap 是系统调用，`usertrap` 会调用 `syscall` 处理它； 如果是装置中断，就调用 `devintr`； 其他情况就是例外，kernel 会把出错的 process 给 kill 掉。 系统调用的情况下，还会将存储的 PC 加上 4，因为 RISC-V 的 `ecall` trap 发生后，`sepc` 仍会指向 `ecall` 那行指令，但 user code 恢复执行时需要从下一行继续执行。 处理完要离开时，`usertrap` 会检查这个 process 是否已经被 kill 了，或如果这次是 timer 中断的话，是否应该交出 CPU

要返回 user space 的第一步是调用 `usertrapret`（[kernel/trap.c:90](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L90)），这个函数会设置 RISC-V 的控制寄存器，为之后从 user space 发生的 trap 做准备：包含将 `stvec` 设为 `uservec`，以及准备好 `uservec` 会用到的 trapframe 栏位。 `usertrapret` 也会把 `sepc` 复原为先前存储的用户程序计数器。 最后，`usertrapret` 会调用 `userret`，`userret` 这段代码也位在 trampoline page 上，且因为 `userret` 的组语程序会切换 page table，所以其会同时映射在 user 和 kernel page table 中

`usertrapret` 调用 `userret` 时，会把 process 的 user page table 地址传入 `a0`（[kernel/trampoline.S:101](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L101)），`userret` 会将 `satp` 设成这份 user page table，记得 user page table 中的 kernel 区段只有 trampoline page 和 `TRAPFRAME` 会被映射，其他 kernel 区段都不会被映射

而 trampoline page 在 user 和 kernel page table 中拥有相同的虚拟地址，且映射到相同的实体地址，因此即使在这之后切换了 `satp`，`userret` 也还能继续执行，在这之后 `userret` 就只能访问寄存器内容与 trapframe 的内容，`userret` 会将 `TRAPFRAME` 地址加载到 `a0`，用它来还原先前存下的用户寄存器，还原用户的 `a0`，最后执行 `sret` 指令返回 user space

:::tip
`usertrapret` 调用 `userret` 的这段代码为：

```c
// tell trampoline.S the user page table to switch to.
uint64 satp = MAKE_SATP(p->pagetable);

// jump to userret in trampoline.S at the top of memory, which 
// switches to the user page table, restores user registers,
// and switches to user mode with sret.
uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64))trampoline_userret)(satp);
```

其中 `TRAMPOLINE` 为固定的 macro，值为 `0x3FFFFFF000`，这是一个固定的虚拟地址。 而 `userret` 与 `trampoline` 为定义在 trampoline.S 中的标签地址，可以通过 nm 或 objdump 来看到具体的值：

```
mes@MesDesktop:~/xv6-riscv$ nm kernel/kernel | grep -E "(trampoline|userret)"
0000000080006000 T _trampoline
0000000080006000 T trampoline
000000008000609c T userret
mes@MesDesktop:~/xv6-riscv$ objdump -t kernel/kernel | grep -E "(trampoline|userret)"
0000000080006000 g       .text  0000000000000000 trampoline
000000008000609c g       .text  0000000000000000 userret
0000000080006000 g       .text  0000000000000000 _trampoline
```

因此 `trampoline` 的值为 `0x80006000`，`userret` 的值为 `0x8000609c`，这两者也都为虚拟地址。 但注意这里准备把 `satp` 换掉了，用的是 user page table，因此在 `userret` 内是「无法使用 direct mapping」的，也因此无法直接使用 `0x8000609c` 这个地址，就算他就是实际上的实体地址，但 user page table 内搞不好根本就没有 `0x8000609c` 的这段映射，或是它可能会映射到其他 page frame

而前面有提到 trampoline page 会同时存在于 kernel 和 user 的 page table 中，这是 kernel page table 中加的 PTE：

```c
// map the trampoline for trap entry/exit to
// the highest virtual address in the kernel.
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
```

这是 user page table 中加的 PTE：

```c
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  ...
  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }
  ...
}
```

可以看到两者会把 `TRAMPOLINE` 这个虚拟地址映射到同一个实体地址（`trampoline`）。 所以这边用了 `TRAMPOLINE` 来走 Sv39 的路线将虚拟地址转实体地址。 通过 `(userret - trampoline)` 求出偏移量，再加上 `TRAMPOLINE`，就可以得到位于 trampoline page 内的 VM 了，其值为 `0x3FFFFFF000 + 0x9c = 0x3FFFFFF09C`，通过 Sv39 的转换，可以得到其值就为 `trampoline = 0x8000609c`

这边比较容易卡住的点是 trampoline page 在 kernel page table 中有另外一种路径是可以走 direct mapping 的。 换句话说如果 `satp` 指向的是 kernel page table，则 `trampoline` 和 `userret` 的值就同时代表了虚拟地址与实体地址，因为这两个地址处于 kernel RAM 区段（`0x80000000` 至 `0x88000000`，见图 3.3），因此在 kernel page table 中使用的是 direct mapping。 然而由于在 `userret` 的上下文忠 `satp` 指向的是 user page table，不能使用 direct mapping，所以才要绕这么大一圈去计算实体地址  
:::

## 4.3 Code: Calling system calls

第二章最后提到了 `initcode.S` 会调用 `exec` 这个系统调用（[user/initcode.S:11](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/initcode.S#L11)）。 现在我们来看看，这个来自 user space 的调用，是怎么一路传递到 kernel 中对应的 `exec` 实现的

`initcode.S` 会把传给 `exec` 的引数放进 `a0` 和 `a1` 寄存器中，并将系统调用的编号放进 `a7`。 系统调用编号会对应到 `syscalls` 数组中的条目，这个数组是一张函数指针的列表（[kernel/syscall.c:107](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L107)）。 接著 `ecall` 指令会触发 trap 进入 kernel，如前所述，这会依序执行 `uservec`、`usertrap` 与 `syscall`

`syscall`（[kernel/syscall.c:132）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L132)会从 `trapframe` 中存储的 `a7` 读出系统调用编号，并利用它去查找 `syscalls` 数组。 第一次的系统调用中，`a7` 里会放 `SYS_exec`（[kernel/syscall.h:8](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.h#L8)），因此会调用到对应的实现函数 `sys_exec`

当 `sys_exec` 执行完毕并返回时，`syscall` 会把它的返回值写进 `p->trapframe->a0`。 这样做的原因是，在 RISC-V 的 C 调用惯例中，返回值会放在 `a0`，因此这样可以让原本 user space 中的 `exec()` 调用收到正确的返回值。 惯例上，系统调用若发生错误会返回负数，成功则是 0 或正数。 若系统调用编号不合法，`syscall` 会印出错误消息，并返回 -1

## 4.4 Code: System call arguments

kernel 中的系统调用实现需要获取 user code 所传入的引数，由于 user code 会通过包装函数来使用系统调用，这些引数一开始会依据 RISC-V 的 C 调用惯例放在寄存器中。 kernel 的 trap 处理流程会将用户寄存器的内容存储到目前 process 的 trapframe 中，让 kernel code 之后可以从那里找到它。 kernel 提供了 `argint`、`argaddr` 和 `argfd` 等函数，分别可用来从 trapframe 中读取第 `n` 个系统调用的引数，并将其作为整数、指针或文件描述符使用。 这些函数内部都会调用 `argraw`，以获取对应的用户寄存器（[kernel/syscall.c:34](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L34)）

有些系统调用会将指针作为引数传入，而 kernel 必须使用这些指针去读写 user memory。 举例来说，`exec`系统调用会传给 kernel 一个数组，里面是指向 user space 中字串引数的指针。 这些指针会带来两个挑战：第一是 user program 中可能有 bug 或是恶意代码，也可能会传入一个无效的指针，甚至尝试诱导 kernel 访问 kernel memory 而不是 user memory； 第二是 xv6 的 kernel page table 映射方式与 user page table 不同，因此 kernel 不能直接用一般的指令去从 user 的地址读写数据

kernel 提供了几个函数，以安全地从 user 给的内存地址读写数据。 例如 `fetchstr`（[kernel/syscall.c:25](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L25)），像 `exec` 这样与文件相关的系统调用会用 `fetchstr` 从 user space 读取字串形式的档名引数。 `fetchstr` 本身会调用 `copyinstr` 来完成底层的复制工作

`copyinstr`（[kernel/vm.c:415)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L415)）会从 user page table `pagetable` 中的虚拟地址 `srcva` 复制最多 `max` 位元组的数据到 `dst`。 由于 `pagetable` 并不是目前使用中的 page table，`copyinstr` 会使用 `walkaddr`（它会调用 `walk`）去查询 `srcva` 在 `pagetable` 中对应的实体地址 `pa0`

由于 xv6 的 kernel page table 采用直接映射，这让 `copyinstr` 可以直接从 `pa0` 复制字串到 `dst`。 `walkaddr`（[kernel/vm.c:109](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L109)）也会检查用户给的虚拟地址是否真的属于该 process 的 user 地址空间，这样 kernel 就不会被诱骗去读其他内存。 另一个类似的函数是 `copyout`，它会把数据从 kernel 复制到 user 提供的地址

## 4.5 Traps from kernel space

xv6 处理来自 kernel code 的 trap 的方式与处理 user code 的 trap 不同。 当进入 kernel 时，`usertrap` 会将 `stvec` 设置为指向 `kernelvec` 的组语代码（[kernel/kernelvec.S:12](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kernelvec.S#L12)）。 由于 `kernelvec` 只有在 xv6 身处 kernel 状态时才会被执行，所以 `kernelvec` 可以假设 `satp` 已经指向了 kernel page table，并且 stack pointer 也已经指向了一个合法的 kernel stack。 `kernelvec` 会把 32 个寄存器的值全部存入 stack 内，之后再从中还原，这样就能让被中断的 kernel code 在不受干扰的情况下继续执行

`kernelvec` 会将寄存器内容存储在被中断的 kernel thread 的 stack 上，因为这些寄存器的值本来就属于该 thread。 这一点在 trap 导致切换到其他 thread 时特别重要，那种情况下 trap 结束后会从新 thread 的 stack 返回，而原本被中断的 thread 的寄存器内容就安全地保留在它自己的 stack 上

在将寄存器存入 stack 之后，`kernelvec` 会跳到 `kerneltrap`（[kernel/trap.c:135）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L135)。 `kerneltrap` 主要用来处理两种 trap：装置中断与例外状况。 它会调用 `devintr`（[kernel/trap.c:185](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)）来侦测并处理装置中断。 如果这个 trap 不是装置中断，那就表示是例外，而在 xv6 的 kernel 中，只要发生例外就一律视为致命错误，此时 kernel 会调用 `panic` 并停止执行

如果这次的 `kerneltrap` 是由 `timer` 中断触发的，而且当前执行的是某个 process 的 kernel thread（不是 scheduler thread），那 `kerneltrap` 就会调用 `yield`，让其他线程有机会被调度执行。 之后某个线程会再次调用 `yield`，使我们原本的线程与它的 `kerneltrap` 再度恢复执行。 `yield` 的详细行为会在第七章说明

当 `kerneltrap` 处理完毕后，它需要返回到原本被 trap 中断的那段代码。 由于 `yield` 可能已经修改了 `sepc` 和 `sstatus` 中的前一个模式，因此 `kerneltrap` 在开始时会先存储这些寄存器，然后将它们还原并回到 `kernelvec`（[kernel/kernelvec.S:38](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kernelvec.S#L38)）中。 `kernelvec` 会从 stack 中将原先存入的寄存器取出，然后执行 `sret` 指令，这会把 `sepc` 的值写回 `pc`，从而回到被中断的 kernel code

这边你可以想想看，如果 `kerneltrap` 是因为 timer 中断而调用了 `yield`，那 trap 是怎么完成返回的？

当某个 CPU 从 user space 进入 kernel 时，xv6 会把该 CPU 的 `stvec` 设置为 `kernelvec`； 你可以在 `usertrap` 中看到这段代码（[kernel/trap.c:29](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L29)）。 不过在 kernel 开始执行且 `stvec` 还没改成 `kernelvec` 的这段期间，`stvec` 仍指向 `uservec`，这段期间如果发生了装置中断就会有问题。 所幸 RISC-V 在进入 trap 时会自动关闭中断，而 `usertrap` 也会等到设完 `stvec` 才重新打开中断

## 4.6 Page-fault exceptions

xv6 对于例外状况的反应相当无趣：如果例外发生在 user space，kernel 就会把出错的 process 给 kill 掉； 如果例外发生在 kernel，kernel 则会直接 panic。 真正的操作系统通常会用更有趣的方式来处理这些状况

举个例子，许多 kernel 会利用 page fault 来实现 copy-on-write（COW）型的 `fork`。 为了说明这种 `fork`，让我们回到 xv6 的 `fork`，它在第三章内有被提到。 `fork` 会让 child process 初始的内存内容和 parent process 当下的内存内容相同。 xv6 用 `uvmcopy`（[kernel/vm.c:313](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L313)）来实现这个功能，它会为 child 分配物理内存，并把 parent 的内存内容复制过去。 如果能让 parent 和 child 共享 parent 的物理内存，效率会更高。 不过直接这样做是行不通的，因为他们会互相写入共享的 stack 和 heap，导致彼此的执行出错

只要搭配正确的 page table 权限设置与 page fault 机制，parent 和 child 是可以安全地共享物理内存的。 当某个虚拟地址没有对应的 page table 映射，或该映射的 `PTE_V` 位元没被设起来，或权限位元（`PTE_R`、`PTE_W`、`PTE_X`、`PTE_U`）不允许所尝试的操作时，CPU 就会生成 page-fault exception。 在 RISC-V 架构中，page fault 分为三种类型：load page fault（由 `load` 指令引起）、store page fault（由 `store` 指令引起）和 instruction page fault（由 instruction fetch 造成）。 `scause` 寄存器会指出是哪种 page fault，而 `stval` 则会记录无法被转换的地址

COW `fork` 的基本做法是，让 parent 和 child 一开始共享所有的 page frame，但他们各自都会将这些 page 设成唯读的（`PTE_W` 栏位清 0）。 parent 和 child 都可以读取这些共享内存，但如果任一方对某个 page 做了写入操作，RISC-V CPU 就会生成一个 page-fault exception。 kernel 的 trap handler 会处理这个例外：它会分配一张新的 page frame，并把发生 fault 时那个地址对应的 page frame 的内容复制过去。 然后 kernel 会更新发生 fault 的 process 的 page table，把对应的 PTE 改成指向新的 page frame，并允许读写

最后 kernel 会让该 process 从造成 fault 的那条指令重新执行。 而因为这时 PTE 已经允许写入了，所以这次执行就不会再触发 page fault。 copy-on-write 需要维护额外的信息来跟踪哪些 page frame 可以被释放，因为每张 page 可能会被多张 page table 所引用，这些引用会随著 fork、page fault、exec 和 exit 而改变。 这样的记录机制还带来一个重要的最佳化：如果某个 process 发生 store page fault，但该 page frame 只有被它自己的 page table 引用，那其实就不需要做复制

copy-on-write 可以让 `fork` 更快，因为在 fork 的当下不需要复制内存。 虽然之后在写入时还是可能得做复制，但实际上大多数内存都不需要真的被复制。 常见的一个例子是 `fork` 后马上做 `exec`：在 `fork` 之后可能只有少数的 page 被写入，而 child 的 `exec` 又会释放掉大部分从 parent 继承来的内存。 copy-on-write `fork` 可以避免复制这些内存。 此外，COW `fork` 是透明的，其不需要对应用程序做任何修改，它们就能自动受益

page table 和 page fault 的组合，除了能实现 COW `fork` 以外，还能支持很多有趣的功能。 其中一个被广泛使用的机制是 lazy allocation，它包含两个步骤。 第一步，当应用程序调用 `sbrk` 向系统要求更多内存时，kernel 会纪录其要增加的大小，但不会马上分配物理内存，也不会为这段新的虚拟地址区间创建 PTE。 第二步，当某个新地址上发生 page fault 时，kernel 才会真正分配一张 page frame，并把它映射进 page table。 就像 COW `fork` 一样，lazy allocation 对应用程序来说也是透明的

由于应用程序请求的内存通常会比实际需要的还多，因此 lazy allocation 在这种情况下就非常有效：对于那些应用程序从未实际使用的 page，kernel 完全不需要做任何处理。 此外，如果应用程序一次请求大量的地址空间，而没有 lazy allocation 的话，`sbrk` 的成本会非常高：例如应用程序要求 1GB 的内存时，kernel 必须分配并清零 262,144 个 4096-byte 的 page。 而 lazy allocation 能使这笔成本随时间摊平

然而，lazy allocation 也会带来额外的 page fault 开销，因为每次 page fault 都会牵涉一次 user/kernel 的切换。 为了降低这项成本，操作系统可以在每次 page fault 时一次分配多个连续的 page，而不是只分配一个，并且还可以为这种 page fault 特化 kernel 的进出路径

另一个广泛使用且仰赖 page fault 的功能是 demand paging。 在 xv6 中，当执行 `exec` 时，它会在启动应用程序前就把应用程序的 text 和 data 段全部加载内存。 由于应用程序可能很大，而从硬盘读数据又很耗时，这个启动成本对用户而言可能会很明显

为了缩短启动时间，现代的 kernel 一开始并不会把可执行档加载内存，而是创建一份 user page table，并将其中所有 PTE 标成 invalid。 kernel 启动程序后，每当程序第一次使用某个 page，就会发生 page fault，然后 kernel 根据这个 fault 去从硬盘读入该 page 的内容，并将它映射到 user 的地址空间。 就像 COW fork 和 lazy allocation 一样，这个机制对应用程序来说是透明的

电脑上执行的程序还可能会需要超过物理内存容量的内存。 为了优雅地处理这种情况，操作系统可能会实现 swapping 的机制。 它的基本想法是：只在内存中保留部分用户的 page，剩下的则存储在硬盘中的 swap space。 kernel 会把那些对应到硬盘中的 swap space 的内存，其对应的 PTE 标为 invalid

接著如果应用程序试图使用某个已经被 swap out 到硬盘的 page，便会触发 page fault，此时该 page 必须被 swap in：kernel 的 trap handler 会分配一张物理内存，将对应的数据从硬盘读回 RAM，然后更新对应的 PTE，让它指向这张新的 page frame

如果某个 page 需要被 swap in，但当下已经没有任何可用的物理内存了，这种情况下 kernel 必须先释放出一张 page frame，方法是将其中的一个 page 做 swap out，也就是把它「搬移」到硬盘上的 swap space，并将所有引用该 page 的 PTE 标记为 invalid

不过「搬移」的成本很高，因此在它不常发生的情况下 paging 的表现较好，这代表应用程序只会使用其分配内存中的一小部分，而且这些常用 page 的总能被放在内存里。 这种特性通常被称为良好的 locality of reference。 就像其他许多虚拟内存技术一样，kernel 通常会让 swapping 对应用程序来说是透明的

:::tip
这边将原文的用词改成了更常见的用词：

- swapping：原文为 paging to disk
- swap out：原文为 paged out
- swap in：原文为 paged in

主要是因为对于「page」相关的词我已经选择保留原文了，这几个再加进来会有些杂乱，导致不好阅读  
:::

即使硬件提供了大量的内存，电脑在实际运行时仍经常处于几乎没有「空闲（free）」物理内存的状态。 例如，云端服务提供者通常会在单一机器上同时运行许多客户的应用程序，以达到硬件资源的最大利用率。 再例如，用户会在只有少量物理内存的智慧型手机上同时执行多个应用程序。 在这些情况下，每次分配一张新 page 前都可能需要先将某张现有的 page swap out。 因此，当物理内存资源紧张时，分配内存的成本会较高

在可用内存紧张、而程序实际上只使用其分配内存的一部分时，lazy allocation 和 demand paging 特别具有优势。 这些技术还能避免某些情况下的资源浪费，例如：某个 page 被分配或从硬盘加载，但却从未被实际使用，或甚至在使用前就被 swap out 了

还有一些其他功能同样结合了 paging 和 page fault exception，例如自动延展的 stack 以及 memory-mapped file。 memory-mapped file 是指程序通过 `mmap` 系统调用把文件映射进自己的地址空间，这样程序就可以直接用 `load` 和 `store` 指令来读写这些文件了

## 4.7 Real world

trampoline 与 trapframe 的设计看起来可能过于复杂。 背后的主要原因是 RISC-V 在触发 trap 时会刻意地不做太多事，这样可以让 trap handler 的执行速度更快，而这点在实现上是非常重要的。 结果就是 kernel 的 trap handler 的前几条指令必须在 user environment 下执行：使用的是 user page table，还有 user 的寄存器内容。 而且 trap handler 起初也不知道像「目前执行的 process 是谁」或「kernel page table 的地址」这些有用的信息

这些问题之所以有解，是因为 RISC-V 提供了一些受保护的区域让 kernel 可以在进入 user space 前先存储信息，例如 `sscratch` 寄存器，还有一些指向 kernel memory 的 user page table entry，但这些 entry 并没有设 `PTE_U` 权限来保护。 xv6 的 trampoline 与 trapframe 就是善用了这些 RISC-V 的特性

如果 kernel memory 会被映射到每个 process 的 user page table 中（但不设 `PTE_U` 权限），那就不需要额外的 trampoline page 了。 这样一来，从 user space trap 进 kernel 时也就不需要切换 page table。 这又让 kernel 在实现系统调用时可以直接访问 user memory，因为这些内存已经被映射到了目前的 page table 中。 许多操作系统都会这样设计来提升效率。 不过 xv6 为了避免 kernel 不小心使用 user pointer 而生成安全漏洞，也为了简化 user 与 kernel 的地址空间不重栈所需要的处理，因此选择不使用这种设计

真正的操作系统会实现像是 copy-on-write `fork`、lazy allocation、demand paging、paging to disk、memory-mapped file 等等机制。 此外，这些系统也会尽量让整个物理内存都有用处，通常会拿来缓存那些不属于任何 process 的文件内容

真正的操作系统也会提供一些系统调用让应用程序来管理自己的地址空间，或是让它自己处理 page fault，例如 `mmap`、`munmap`、`sigaction` 这些调用，也会提供像 `mlock` 这样的调用来让应用程序使用的 page 固定在内存里而不被 swap out，或是像 `madvise` 这样的调用让应用程序告诉 kernel 它打算怎么使用这块内存

## 4.8 Exercises

1. `copyin` 和 `copyinstr` 会通过软件的方式走访 user page table。 设置 kernel 的 page table，让 kernel 能直接映射 user program，这样 `copyin` 和 `copyinstr` 就可以改用 `memcpy` 把系统调用的引数复制到 kernel space，而不用自己做 page table walk 了
2. 实现 lazy memory allocation
3. 实现 COW fork
4. 有没有方法可以去掉每个 user address space 中的 `TRAPFRAME` page 映射？ 例如，`uservec` 是否可以改成直接把 32 个 user 寄存器 push 到 kernel stack，或是存在 `proc` 结构中？
5. 能不能改写 xv6，让它不需要 `TRAMPOLINE` page 的映射？
6. 实现 `mmap`

## Bibliography

- <a id="1">[1]</a>：The RISC-V instruction set manual Volume II: privileged specification. https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link, 2024
