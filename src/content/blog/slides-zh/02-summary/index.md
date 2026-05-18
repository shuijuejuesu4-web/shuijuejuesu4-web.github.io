---
title: "CS110：UNIX文件系统API编程"
description: "CS110：UNIX文件系统API编程"
publishDate: 2024-01-01
tags: [CS110, 课件]
category: "CS110-课件"
draft: false
comment: true
---
# CS110：针对 UNIX 文件系统的 API 编程

* 硬件上层的软件、文件系统 API 调用
    * 首先，我们将初步了解磁盘驱动器的物理硬件如何被构建成看起来像存储传统文件的软件。我会省略一些细节，但会提供足够的细节来清楚地说明大小差异巨大的普通文件如何存储在磁盘上，并通过由 `FILE *`、`ifstream` 和 `ofstream` 等数据类型管理的文件会话来检索。
    * 我们将学习程序员如何通过**系统调用**与文件系统交互（直接交互，或通过 `FILE *` 和 `[io]stream` 实现间接交互），系统调用是一组驻留在内核中的函数，用户程序必须通过这些函数来访问和操作系统资源。打开文件、读取文件、扩展堆等请求最终都通过系统调用来完成，它们是唯一可以信任来接触系统的函数。

::: tip 重难点解析
**系统调用（System Call）的本质**：系统调用是用户程序访问操作系统内核服务的唯一入口。当你的程序调用 `open()`、`read()`、`write()` 这些函数时，实际触发的是从用户态（user mode）到内核态（kernel mode）的切换——CPU 执行 `syscall` 指令（x86-64），陷入内核，由内核在受保护的环境中完成文件操作，然后返回结果。这种设计是为了保护系统资源不被任意用户程序破坏。与之对比，普通的库函数（如 `printf`、`strlen`）完全在用户态执行，不能直接访问硬件。

**用户态与内核态**：这是现代操作系统安全模型的基石。用户态代码只能访问受限的地址空间和有限的 CPU 指令；内核态代码可以访问全部内存和所有硬件资源。系统调用本质上是一次"受控的门"——用户程序敲门请求服务，内核在验证权限后执行操作并返回。CS111 课程会进一步讨论系统调用的具体实现细节和上下文切换的开销。
:::

    * 今天的讲座示例位于 `/usr/class/cs110/lecture-examples/spring-2017/filesystems`。
    * `/usr/class/cs110/lecture-examples/spring-2017` 目录是一个 Mercurial 仓库，随着学期的推进，将会更新更多示例。
        * 开始使用时，在命令提示符下输入 `hg clone /usr/class/cs110/lecture-examples/autumn-2017 cs110-lecture-examples` 来创建主仓库的本地副本。
        * 每次我提到有新示例时，进入你的本地副本并输入 `hg pull && hg update`。
    * 更重要的是，请阅读 Saltzer 和 Kaashoek 在线教材的[第 1 至 5 节](http://www.sciencedirect.com/science/article/pii/B9780123749574000116)，特别注意第 5 节的细节，这将帮助你完成第一次作业（周五发布）。

# 文件系统 API：实现 `copy` 以模拟 `cp`

* `copy` 的实现
    * `copy` 的实现（旨在模仿 `cp` 的行为）说明了如何使用 `open`、`read`、`write`、`close`、`stat`。它还引入了文件描述符的概念。

::: tip 重难点解析
**文件描述符（File Descriptor）**：文件描述符是一个整数——就是这张幻灯片中 `open()` 返回的那个 `int`。但它本质上是一个索引，指向内核为每个进程维护的"打开文件表"中的一个条目。文件描述符 0、1、2 分别预留给标准输入、标准输出和标准错误。每当你调用 `open()` 成功时，内核返回当前可用的最小整数作为新的文件描述符。这个看似简单的整数抽象是 Unix "一切皆文件"哲学的基石——无论是磁盘文件、管道、套接字还是设备，都可以通过同一个 `read(fd, ...)` / `write(fd, ...)` 接口操作。
:::

    * 所有这些函数都有 `man` 手册页（例如 `man 2 open`、`man 2 read` 等）
    * 我们自己的 `copy` 可执行文件的完整实现在[这里](http://cs110.stanford.edu/autumn-2017/examples/filesystems/copy.c)。

* 文件描述符相比 `FILE` 指针和 C++ `iostream` 的优缺点
    * 文件描述符抽象提供了对数据流的直接、底层访问，无需数据结构或对象的繁琐处理。它肯定不会更慢，而且根据你在做什么，甚至可能更快。
    * `FILE` 指针和 C++ `iostream` 在你知道自己在标准输出、标准输入和本地文件之上工作时表现出色。但当字节流与网络连接相关联时，它们就不太好用了。（`FILE` 指针和 C++ `iostream` 假设它们可以来回自由地倒带和移动文件指针，但对于与网络连接关联的文件描述符来说，这是不可行的）。
    * 然而，文件描述符只能与 `read` 和 `write` 配合使用，不能做其他事情。C 语言的 `FILE` 指针和 C++ 流提供了自动缓冲和更复杂的格式化选项。

# 文件系统 API（续）

* `copy` 的实现

    ```c
    int main(int argc, char *argv[]) {
      if (argc != 3) {
        fprintf(stderr, "%s <source-file> <destination-file>.\n", argv[0]);
        return kWrongArgumentCount;
      }
    
      int fdin = open(argv[1], /* flags = */ O_RDONLY);
      if (fdin == -1) {
        fprintf(stderr, "%s: source file could not be opened.\n", argv[1]);
        return kSourceFileNonExistent;
      }
    
      int fdout = open(argv[2], /* flags = */ O_WRONLY | O_CREAT | O_EXCL, 0644);
      if (fdout == -1) {
        switch (errno) {
          case EEXIST:
            fprintf(stderr, "%s: destination file already exists.\n", argv[2]);
            break;
          default:
            fprintf(stderr, "%s: destination file could not be created.\n", argv[2]);
            break;
        }   
        return kDestinationFileOpenFailure;
      }
    ```

::: tip 重难点解析
**open() 的标志位与位掩码技巧**：注意 `O_WRONLY | O_CREAT | O_EXCL` 这种写法。这些宏本质上是不同的比特位（如 `O_WRONLY = 0x01`、`O_CREAT = 0x40`、`O_EXCL = 0x80`），通过按位或（`|`）组合在一起。这是一种经典的 C 语言 API 设计模式——用一个整数同时传递多个布尔选项，内核通过按位与（`&`）检查每个标志是否被设置。这种设计在系统编程中极为常见（`mmap`、`socket`、`waitpid` 的参数都使用了类似的技巧）。

**O_EXCL 的原子性保证**：`O_CREAT | O_EXCL` 的组合保证：如果文件已存在，`open` 将失败并返回 -1 且设置 `errno` 为 `EEXIST`。关键的是，这个检查和创建是原子操作——不会出现两个进程同时检测到文件不存在、然后都尝试创建而产生的竞争条件（race condition）。这是文件系统层面解决并发问题的一个经典案例。
:::

::: tip 重难点解析
**文件描述符的三级内核表结构**：文件描述符（一个 `int`）背后是内核维护的三级表格，理解这三层结构是掌握 Unix I/O 的关键：

```
进程级:  fd table (per-process)            系统级:  open file table               系统级:  inode table
┌─────┐     ┌──────────────────┐          ┌──────────────────────┐          ┌──────────────────┐
│ fd  │────>│ struct file *    │─────────>│ struct file {        │          │ struct inode {   │
│  0  │     │ close_on_exec    │          │   f_mode (R/W)       │          │   st_mode        │
│  1  │     └──────────────────┘          │   f_pos (偏移量)      │─────────>│   st_size        │
│  2  │                                   │   f_count (引用计数)   │          │   st_ino         │
│  3  │                                   │   *f_op (操作表)       │          │   数据块指针[]    │
│ ... │                                   └──────────────────────┘          └──────────────────┘
└─────┘
```

**第一层：进程的 fd table**（`files_struct`）——每个进程独有。`fd` 是这张表的索引。表中的条目是一个指向系统级 `struct file` 的指针。`close_on_exec` 标志控制该 fd 在 `exec` 后是否保留（参见 `FD_CLOEXEC`）。

**第二层：系统级 open file table**——内核全局的 `struct file` 实例。关键字段：`f_mode`（读/写模式）、`f_pos`（当前文件偏移量，`read`/`write` 在此位置进行并从这里继续）、`f_count`（引用计数，多少个 fd 指向它）。`dup2` 和 `fork` 会创建新的 fd 但共享同一个 `struct file`——这就是父子进程共享文件偏移量的原因。

**第三层：inode table**——存储文件的元数据和数据块指针。多个 `struct file` 可以指向同一个 inode（例如同一个文件被多次 `open`）。

**为什么需要三层？** 核心原因：分离"打开实例"和"文件本身"。`f_pos`（文件偏移量）属于"打开实例"而非"文件本身"——同一个文件被 `open` 两次产生两个独立的偏移量。而 inode 信息（大小、权限）属于文件本身。引用计数 `f_count` 确保在最后一个打开实例关闭后才释放资源。
:::

# 文件系统 API（续）

* `copy` 的实现，续

    ```c
      char buffer[1024];
      while (true) {
        ssize_t bytesRead = read(fdin, buffer, sizeof(buffer));
        if (bytesRead == 0) break;
        if (bytesRead == -1) {
          fprintf(stderr, "%s: lost access to file while reading.\n", argv[1]);
          return kReadFailure;
        }
    
        size_t bytesWritten = 0;
        while (bytesWritten < bytesRead) {
          ssize_t count = write(fdout, buffer + bytesWritten, bytesRead - bytesWritten);
          if (count == -1) {
            fprintf(stderr, "%s: lost access to file while writing.\n", argv[2]);
            return kWriteFailure;
          }
          bytesWritten += count;
        }
      }
    
      if (close(fdin) == -1) fprintf(stderr, "%s: had trouble closing file.\n", argv[1]);
      if (close(fdout) == -1) fprintf(stderr, "%s: had trouble closing file.\n", argv[2]);
      return 0;
    }
    ```

::: warning 注意事项
**`write()` 不一定一次性写完所有数据**：注意代码中外层 `while` 和内层 `while` 的双重循环结构。`write(fd, buf, n)` 承诺尝试写入最多 `n` 个字节，但实际写入的字节数可能小于 `n`（例如磁盘空间不足、信号中断等）。因此，健壮的程序必须在循环中持续调用 `write`，每次从上次结束的位置继续，直到所有数据写完为止。这种"短写"（short write）现象在系统编程中是一个常见陷阱，也是 `writeall()` 辅助函数之所以存在的原因。
:::

# 文件系统 API：实现 `t` 以模拟 `tee`

* `tee` 概述
    * `tee` 用户程序将标准输入的所有内容复制到标准输出，并在作为用户程序参数提供的命名文件中生成零个或多个额外副本。例如，如果文件 `alphabet.txt` 包含 27 个字节——26 个英文字母后跟一个换行符，那么以下命令会将字母表打印到标准输出以及三个名为 `one.txt`、`two.txt` 和 `three.txt` 的文件中。

    ```sh
    myth4> cat alphabet.txt | tee one.txt two.txt three.txt
    abcdefghijklmnopqrstuvwxyz
    myth4> cat one.txt 
    abcdefghijklmnopqrstuvwxyz
    myth4> cat two.txt
    abcdefghijklmnopqrstuvwxyz
    myth4> diff one.txt two.txt
    myth4> diff one.txt three.txt
    myth4>
    ```

    * 如果文件 `vowels.txt` 包含五个元音和换行符，并且 `tee` 按如下方式调用，`one.txt` 将被重写为只包含英文元音，但 `two.txt` 和 `three.txt` 将保持不变。

    ```sh
    myth4> more vowels.txt | tee one.txt
    aeiou
    myth4> more one.txt 
    aeiou
    myth4> more two.txt 
    abcdefghijklmnopqrstuvwxyz
    myth4>
    ```

    * 我们自己的 `t` 可执行文件的完整实现在[这里](http://cs110.stanford.edu/autumn-2017/examples/filesystems/t.c)。
    * 这里的实现复制了 `copy.c` 所做的大部分内容，但它说明了如何使用底层 I/O 管理多个文件的多个会话。这里的实现错误检查较少，因为我希望你关注底层 I/O 及其成功方式，而不是其失败方式。

# 文件系统 API（续）

* `t` 的实现

    ```c
    static void writeall(int fd, const char buffer[], size_t len) {
      size_t numWritten = 0;
      while (numWritten < len) {
        numWritten += write(fd, buffer + numWritten, len - numWritten);
      }
    }
    
    int main(int argc, char *argv[]) {
      int fds[argc];
      fds[0] = STDOUT_FILENO;
      for (size_t i = 1; i < argc; i++)
        fds[i] = open(argv[i], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    
      char buffer[2048];
      while (true) {
        ssize_t numRead = read(STDIN_FILENO, buffer, sizeof(buffer));
        if (numRead == 0) break;
        for (size_t i = 0; i < argc; i++)
          writeall(fds[i], buffer, numRead);
      }
    
      for (size_t i = 1; i < argc; i++) close(fds[i]);
      return 0;
    }
    ```

    * 特征 1：请注意，`argc` 恰好提供了需要写入的描述符数量。
    * 特征 2：`STDIN_FILENO` 是数字 0 的内置常量，它是通常绑定到标准输入的描述符。`STDOUT_FILENO` 是数字 1 的常量，它是绑定到标准输出的默认描述符。
    * 特征 3：我假设这里所有系统调用都成功。我保证我不是偷懒。我只是试图让示例尽可能清晰和紧凑。

# 文件系统 API：使用 `stat` 和 `lstat`

* `stat` 和 `lstat`
    * `stat` 是一个函数，用于将某个命名文件（普通文件、目录、链接）的信息填充到 `struct stat` 中。
    * `stat` 和 `lstat` 的操作方式完全相同，区别在于当命名文件是一个符号链接时，`stat` 返回的是链接所引用文件的信息，而 `lstat` 返回的是链接本身的信息。

::: tip 重难点解析
**stat vs lstat：符号链接的追踪与不追踪**：这是文件系统 API 中一个微妙但重要的区别。`stat` 会"追踪"（follow）符号链接，返回目标文件的元数据；`lstat` 不追踪，返回符号链接自身的元数据。这两者的区别在实际开发中影响深远——例如 `find` 命令默认使用 `lstat` 来避免陷入符号链接的循环，而 `ls -l` 使用 `lstat` 显示链接本身的权限和信息（而不是目标文件的信息）。如果你想检查一个路径是否是符号链接，必须使用 `lstat` 配合 `S_ISLNK` 宏——用 `stat` 永远检测不到符号链接，因为它已经追踪过去了。
:::

~~~sh
* 这两个函数都有手册（`man`）页（例如 `man 2 stat`、`man 2 lstat` 等）
* `struct stat` 包含以下字段（[来源](http://pubs.opengroup.org/onlinepubs/7908799/xsh/sysstat.h.html)）

```sh
dev_t     st_dev     包含文件的设备 ID
ino_t     st_ino     文件序列号
mode_t    st_mode    文件模式
nlink_t   st_nlink   指向文件的链接数
uid_t     st_uid     文件的用户 ID
gid_t     st_gid     文件的组 ID
dev_t     st_rdev    设备 ID（如果文件是字符或块特殊文件）
off_t     st_size    文件大小（以字节为单位，如果文件是普通文件）
time_t    st_atime   最后访问时间
time_t    st_mtime   最后数据修改时间
time_t    st_ctime   最后状态变更时间
blksize_t st_blksize 文件系统为此对象指定的首选 I/O 块大小。
                     在某些文件系统类型中，这可能
                     因文件而异
blkcnt_t  st_blocks  为此对象分配的块数
```

* `st_mode` 字段与其说是一个单一的值，不如说是一个编码了文件类型和权限的多条信息的位集合。
* 可以使用一组位掩码和宏从 `st_mode` 字段中提取信息。
* 接下来的两个示例——在以下[两](02-filesystems-search.html) [个幻灯片](02-filesystems-list.html)中展示——说明了如何使用 `stat` 和 `lstat` 函数在文件系统中遍历和操作文件树。
~~~

::: tip 重难点解析
**`st_mode` 位字段的真面目——八进制与按位运算**：`st_mode` 是一个 16 位的 `mode_t` 类型，其各位的实际分配如下（以八进制表示）：

```c
八进制位值:     0170000 (高 4 位 = 文件类型)
              0004000 (setuid)
              0002000 (setgid)
              0001000 (sticky bit)
              0000700 (owner 的 rwx)
              0000070 (group 的 rwx)
              0000007 (other 的 rwx)
```

**文件类型判断的宏展开**——以 `S_ISDIR` 为例：

```c
#define S_IFMT   0170000   // 位掩码: 1111 000 000 000 000 (二进制)
#define S_IFREG  0100000   // 普通文件: 1000 000 000 000 000
#define S_IFDIR  0040000   // 目录:     0100 000 000 000 000
#define S_IFLNK  0120000   // 符号链接: 1010 000 000 000 000

#define S_ISDIR(m)  (((m) & S_IFMT) == S_IFDIR)
// 等价于: ((m & 0170000) == 0040000)
```

实际检测过程：`st_mode` 与 `S_IFMT`（0170000）按位 AND，只保留高 4 位（文件类型部分），然后与 `S_IFDIR`（0040000）比较。如果高 4 位恰好是 `0100`（八进制 4），则是目录。同理，`S_ISREG` 检查高 4 位是否为 `1000`（八进制 10）。

**权限位的提取**：权限不再是"与掩码后判等"，而是"与掩码后判非零"：
```c
#define S_IRUSR  0000400   // owner read:   100 000 000 (二进制)
#define S_IWUSR  0000200   // owner write:   10 000 000
if (st.st_mode & S_IRUSR) printf("r"); else printf("-");
```
这里 `st_mode & S_IRUSR` 提取该位，非零则权限存在。注意文件类型用 `==` 比较（精确匹配），权限位用 `!= 0` 判断（存在性检测）——这是两种不同的位运算使用模式，区别在于：类型是多位编码（需精确匹配值），权限是单位标志（只需判断该位是否为 1）。

**`st_nlink` 与文件删除**：`st_nlink` 记录硬链接计数。当 `st_nlink` 降为 0 且没有进程持有该文件的 fd 时，文件的数据块才被真正释放。这是 `rm` 不立刻释放空间的底层原因——`rm` 只是 `unlink()`，减少链接计数，数据只有在最后一个引用消失后才删除。
:::

# 文件系统 API：实现 `search`

* `search` 的实现
* `search` 是我们自己简化实现的 `find` 内置命令。

::: tip 重难点解析
**深度优先遍历与递归**：`search` 程序展示了文件系统遍历的经典模式——深度优先搜索（DFS）。对于每个目录，先处理其中的普通文件，再递归进入子目录。这种遍历方式自然地映射到递归实现：`listMatches` 在处理目录时调用自身。需要注意两点：避免无限递归（跳过 `.` 和 `..` 条目），以及注意递归深度——极端情况下，文件系统嵌套过深可能导致栈溢出。这也是为什么工业级的 `find` 命令会使用迭代而非递归实现。
:::

~~~c
```c
static void exitUnless(bool test, FILE *stream, int code, const char *control, ...) {
  if (test) return;
  va_list arglist;
  va_start(arglist, control);
  vfprintf(stream, control, arglist);
  va_end(arglist);
  exit(code);
}

int main(int argc, char *argv[]) {
  exitUnless(argc == 3, stderr, kWrongArgumentCount,
             "Usage: %s <directory> <pattern>\n", argv[0]);
  struct stat st;
  const char *directory = argv[1];
  stat(directory, &st);
  exitUnless(S_ISDIR(st.st_mode), stderr, kDirectoryNeeded,
             "<directory> must be an actual directory, %s is not", directory);
  size_t length = strlen(directory);
  if (length > kMaxPath) return 0;

  const char *pattern = argv[2];
  char path[kMaxPath + 1];
  strcpy(path, directory); // no buffer overflow because of above check                  
  listMatches(path, length, pattern);
  return 0;
}
```
~~~

# 文件系统 API：`search`（续）

* 实现细节
    * 除非你好奇，否则不要担心 `va_list` 的技巧。我只是想将错误检查统一到一个辅助函数中。我的 `exitUnless` 基本上是 `assert` 宏的一个更炫酷的版本。
    * 对我们来说，第一个新事物是调用 `stat`，它从文件系统中提取有关命名文件的大量信息，并将 `st` 填充这些信息。
    * 你还会注意到使用了 `S_ISDIR` 宏，它检查 `st_mode` 字段的高四位，以确定命名文件是否为目录（或指向目录的链接）。
    * `S_ISDIR` 有几个同类宏：`S_ISREG` 决定文件是否是普通文件，`S_ISLNK` 决定文件是否为链接。（我们将在下一个示例中使用所有这些宏）。
    * 大部分有趣的内容由 `listMatches` 函数管理，该函数对文件系统进行深度优先遍历，查看哪些文件恰好包含指定的 `pattern` 作为子字符串。

# 文件系统 API：`search`（续）

* `listMatches` 的实现

    ```c
    static void listMatches(char path[], size_t length, const char *pattern) {
      DIR *dir = opendir(path);
      if (dir == NULL) return; // path isn't a directory
      strcpy(path + length, "/");
      while (true) {
        struct dirent *de = readdir(dir);
        if (de == NULL) break;
        if (strcmp(de->d_name, ".") == 0 || strcmp(de->d_name, "..") == 0) continue;
        if (length + strlen(de->d_name) + 1 > kMaxPath) continue;
        strcpy(path + length + 1, de->d_name);
        struct stat st;
        lstat(path, &st);
        if (S_ISREG(st.st_mode)) { 
          if (strstr(de->d_name, pattern) != NULL) {
            printf("%s\n", path);
          }
        } else if (S_ISDIR(st.st_mode)) {
          listMatches(path, length + 1 + strlen(de->d_name), pattern);
        }
      }
    
      closedir(dir);
    }
    ```

# `listMatches` 的实现（续）

* 实现细节
    * 我的实现依赖于 `opendir`，它接受一个应该是目录的参数。它返回一个指向不透明可迭代对象的指针，该对象通过一系列 `readdir` 调用呈现一系列 `struct dirent`。
    * 如果 `opendir` 的参数不是目录，它将返回 `NULL`。
    * 当 `DIR` 已经呈现了其所有条目时，`readdir` 返回 `NULL`。返回 `NULL` 表示遍历结束。
    * `struct dirent` 只被*保证*包含一个 `d_name` 字段，该字段是目录条目名称的 C 字符串表达式。`.` 和 `..` 也包含在命名条目序列中，但我忽略它们，以避免对任何单一目录循环遍历超过一次。
    * 我使用 `lstat` 而不是 `stat`，这样我知道条目是否真的是链接。
    * 如果状态明确标识一个条目为普通文件，那么当且仅当其包含我们感兴趣的 `pattern` 时，我才打印完整路径。
    * 如果状态标识一个条目为目录，那么我递归地进入其中查看它的任何命名条目是否匹配我们要查找的模式。
    * `opendir` 返回对一个记录的访问权限，该记录最终必须通过调用 `closedir` 来释放。这就是为什么我的实现以它结尾。

::: tip 重难点解析
**`struct dirent` 与 `d_type`——为何它只是一个"提示"**：POSIX 标准只保证 `struct dirent` 包含 `d_name` 字段（目录条目名称）和 `d_ino`（inode 编号）。`d_type` 字段是 Linux 扩展，用于指示条目类型而不需要额外的 `stat` 调用：

```c
struct dirent {
    ino_t          d_ino;       // inode 编号
    off_t          d_off;       // 到下一个 dirent 的偏移
    unsigned short d_reclen;    // 此记录的长度
    unsigned char  d_type;      // 文件类型（DT_REG, DT_DIR, DT_LNK 等）
    char           d_name[];    // 文件名（以 '\0' 结尾，灵活数组成员）
};

// d_type 的可能值（与 S_IFxxx 对应）:
#define DT_UNKNOWN  0   // 文件系统不支持 d_type
#define DT_REG      8   // 普通文件（对应 S_IFREG = 0100000）
#define DT_DIR      4   // 目录（对应 S_IFDIR = 0040000）
#define DT_LNK     10   // 符号链接（对应 S_IFLNK = 0120000）
```

**为什么 `d_type` 不可靠？** `d_type` 由底层文件系统的 `readdir` 实现填充，并非所有文件系统都支持它：
- **ext4、tmpfs**：始终填充 `d_type`，可以信任。
- **XFS**：早期版本不填充 `d_type`，始终返回 `DT_UNKNOWN`。较新版本可配置。
- **NFS**（某些版本）：可能返回 `DT_UNKNOWN`，因为网络文件系统不总是传递类型信息。
- **FUSE（用户态文件系统）**：取决于具体实现。

因此，**生产级代码**的健壮写法是：先用 `d_type` 作为快速路径，但如果 `d_type == DT_UNKNOWN`，则回退到 `lstat` 获取确切类型。这正是 `find` 命令的实现方式——它在支持 `d_type` 的文件系统上可以避免对每个文件调用 `stat`，大幅提升性能。

**`d_off` 与 telldir/seekdir**：`d_off` 是目录流中的一个"cookie"值，允许通过 `seekdir()` 跳转到之前通过 `telldir()` 保存的位置。不过 POSIX 未标准化此行为，依赖它可能导致跨平台问题。
:::

# 文件系统 API：`list`

* `list` 实现细节
    * 我还展示了 `list` 的实现，它模拟了 `ls` 的功能（特别是 `ls -lUa`）。
    * `list` 和 `search` 的实现有很多共同点，但 `list` 的实现要长得多。
    * 完整的 `list` 可执行文件的实现在[这里](http://cs110.stanford.edu/autumn-2017/examples/filesystems/list.c)。
    * 示例输出（注意这是我自己的 `list`，不是 `ls`！）：

    ```sh
    myth7> list /usr/class/cs110/WWW
    drwxr-xr-x  8    70296 root       2048 Sep 24 15:07 .
    drwxr-xr-x >9 root     root       2048 Sep 25 12:46 ..
    drwxr-xr-x  2    70296 root       2048 Sep 24 09:23 restricted
    drwx------  2 poohbear operator   2048 Sep 24 09:23 repos
    drwx------ >9 poohbear operator   2048 Sep 25 16:29 autumn-2017
    -rw-------  1 poohbear operator     89 Sep 24 15:03 index.html
    ```

    * 我不展示完整的实现。我只展示一个关键函数：那个知道如何打印任意条目权限信息的函数。

# 文件系统 API：`list`

* `list` 的 `listPermissions` 实现：

    ```c
    static inline void updatePermissionsBit(bool flag, char permissions[], size_t column, char ch) {
      if (flag) permissions[column] = ch;
    }
    
    static const size_t kNumPermissionColumns = 10;
    static const char kPermissionChars[] = {'r', 'w', 'x'};
    static const size_t kNumPermissionChars = sizeof(kPermissionChars);
    static const mode_t kPermissionFlags[] = {
      S_IRUSR, S_IWUSR, S_IXUSR, // user flags
      S_IRGRP, S_IWGRP, S_IXGRP, // group flags
      S_IROTH, S_IWOTH, S_IXOTH  // everyone (other) flags
    };
    static const size_t kNumPermissionFlags = sizeof(kPermissionFlags)/sizeof(kPermissionFlags[0]);
    
    static void listPermissions(mode_t mode) {
      char permissions[kNumPermissionColumns + 1];
      memset(permissions, '-', sizeof(permissions));
      permissions[kNumPermissionColumns] = '\0';
      updatePermissionsBit(S_ISDIR(mode), permissions, 0, 'd');
      updatePermissionsBit(S_ISLNK(mode), permissions, 0, 'l');
      for (size_t i = 0; i < kNumPermissionFlags; i++) {
        updatePermissionsBit(mode & kPermissionFlags[i], permissions, i + 1,
                             kPermissionChars[i % kNumPermissionChars]);
      }
      printf("%s ", permissions);
    }
    ```

    * `list` 的完整实现在 [list.c](http://cs110.stanford.edu/autumn-2017/examples/filesystems/list.c)。

::: tip 重难点解析
**`st_mode` 位字段与权限模型**：Unix 权限用 12 个比特位编码（实际上更多，但常用的为低 12 位）。从高位到低位依次是：文件类型（4 位，如目录 `d`、普通文件 `-`、符号链接 `l`）、SUID/SGID/sticky 位（3 位）、所有者权限 rwx（3 位）、所属组权限 rwx（3 位）、其他用户权限 rwx（3 位）。`listPermissions` 函数中的 `mode & kPermissionFlags[i]` 操作正是利用位掩码逐位提取权限信息。这种位运算技巧在系统编程中无处不在——从文件权限到网络协议标志位，再到 `mmap` 的保护标志。
:::

::: tip 重难点解析
**硬链接 vs 符号链接——inode 层面的本质区别**：

**硬链接（hard link）**：`link("target", "newlink")` 创建的是一个**指向相同 inode 的新目录条目**。两个文件名完全平等（不存在"原始"和"复制"的区别），共享同一个 inode。通过 `ls -li` 可以看到相同的 inode 编号：
```sh
$ echo "hello" > original.txt
$ ln original.txt hardlink.txt
$ ls -li original.txt hardlink.txt
1298477 -rw-r--r-- 2 user group 6 May 17 10:00 hardlink.txt
1298477 -rw-r--r-- 2 user group 6 May 17 10:00 original.txt
#        ^ st_nlink = 2 (两个目录条目指向同一个 inode 1298477)
```
修改任一个文件（通过任一名称）会影响另一个（因为它们是同一份数据）。`rm original.txt` 只是减少 `st_nlink` 计数，数据在 `st_nlink` 降至 0 时才释放。

**符号链接（symbolic link / soft link）**：`symlink("target", "linkname")` 创建的是一个包含目标路径字符串的**特殊类型文件**（`S_IFLNK = 0120000`）。它有自己的 inode 和数据块（存储目标路径）：
```sh
$ ln -s original.txt symlink.txt
$ ls -li original.txt symlink.txt
1298477 -rw-r--r-- 1 user group 6 May 17 10:00 original.txt
1298478 lrwxrwxrwx 1 user group 12 May 17 10:01 symlink.txt -> original.txt
#        ^ 符号链接有自己的 inode (1298478)，类型为 l
```

**硬链接的两大限制（及其原因）**：
1. **不能跨文件系统**：硬链接本质是在目录中增加一个 inode 条目，而 inode 编号只在单个文件系统内唯一。不同文件系统（如 ext4 的 `/home` 和 tmpfs 的 `/tmp`）有各自独立的 inode 空间，无法创建跨文件系统的硬链接。
2. **不能链接到目录**（root 也不行）：如果允许目录的硬链接，会创建目录图中的环。考虑 `ln /a /a/b/c`——遍历 `/a/b/c/..` 时可能遇到无限循环。`find`、`du` 等工具依赖目录结构为 DAG（有向无环图）来避免无限遍历。Linux 内核直接禁止此操作，返回 `EPERM`。

**`stat` vs `lstat` 在链接上的行为**：`stat` 追踪符号链接（返回目标文件的元数据），`lstat` 不追踪（返回链接自身的元数据）。对于硬链接，两者行为完全相同——因为没有"链接自身"的概念，两个名称都直接指向 inode。
:::
