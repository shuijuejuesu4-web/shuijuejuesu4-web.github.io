# 公告
* 今日议程
    * 深入讲解[上周一](http://web.stanford.edu/class/cs110/autumn-2017/lectures/16-networking-scrabble-api.html)发布的 Scrabble API 示例。
    * 讨论 MapReduce 编程模型：什么是 mapper，什么是 reducer，以及如何
      将它们链接在一起形成一个进程管道来分析和处理大型数据集。
        * 我们将展示与最经典的 MapReduce 作业（词频统计）相关联的 map 和 reduce 可执行文件。
          幻灯片中的代码使用 Python 编写，因为它非常简短且易于理解（即使你不懂 Python）。
          我的讲课不会聚焦于代码，而是聚焦于总体思想（我认为这相当直观）。
        * 我们将讨论如何将非常大的数据集划分为许多块文件，
          并通过成百上千台机器上同时执行的大量 map 和 reduce 作业进行处理。
        * 我们将讨论 `group-by-key` 算法（实际上非常简单直接），
          它对所有 mapper 输出文件的完整集合运行，以生成完整的 reducer 输入文件集合。

# MapReduce
* Mapper 是什么？
    * Mapper 是一个程序，它读取任意数据文件
      并输出一个键值对文件，每行一个键值对。
    * 以下是一个 Python 程序示例，它读取任意文本文件并
      输出形式为 "`<word>` 1" 的行。

    ```python
    #!/usr/bin/env python
    import sys
    import re
    import string

    pattern = re.compile("^[a-z]+$") # matches purely alphabetic words
    for line in sys.stdin:
        line = line.strip()
        tokens = line.split()
        for token in tokens:
            lowercaseword = token.lower()
            if pattern.match(lowercaseword):
                print '%s 1' % lowercaseword
    ```

    * 上述程序可以按如下方式调用：

    ```sh
    myth22> cat anna-karenina.txt | ./word-count-mapper.py
    ```

    * 这样做将产生以下（压缩的）输出：

    ```sh
    anna 1
    karenina 1
    by 1
    leo 1
    tolstoy 1
    ...
    i 1
    have 1
    the 1
    power 1
    to 1
    put 1
    into 1
    ```

::: tip 重难点解析
**Mapper 的核心职责**：Mapper 不关心全局统计，它只对"自己看到的那一块数据"负责。它的唯一任务就是把原始数据转换成 `(key, value)` 对。在词频统计这个经典例子中，每个单词被映射为 `(word, 1)`——这里的 `1` 是"我看到这个单词一次"的标记，而不是最终计数。你可以把 mapper 想象成一个"分拣员"：它把杂乱的输入分门别类地打上标签，为后续的汇总（reducer）做准备。

**为什么每行是 "word 1" 而不是直接计数？** 这体现了 MapReduce 的核心设计哲学：mapper 之间完全独立、无共享状态。每个 mapper 不需要知道其他 mapper 看到了什么，只需要诚实报告自己看到的内容即可。最终的聚合工作完全交给 reducer。这种"各司其职"的设计使得 mapper 可以大规模并行执行，而无需任何同步开销。
:::

::: tip 重难点解析
**MapReduce 的 Shuffle 阶段 —— 连接 Map 和 Reduce 的"隐形管道"**

在 CS110 的管道演示中，你看到的是 `sort | group-by-key.py | reducer` 的串联。但在分布式 MapReduce（如 Hadoop）中，"数据如何从 mapper 到达正确的 reducer"是一个完全不同的问题，这个阶段称为 **shuffle**。

**1. 分区 (Partitioning) 机制**

每个 mapper 输出 `(key, value)` 对后，必须决定"这个键值对应该发给哪个 reducer"。标准方法：**哈希分区**（hash partitioning）。

```
partiton = hash(key) % R    // R = number of reducers
```

例如，有 3 个 reducer（R=3），单词 "apple"（hash=123456）和 "banana"（hash=789012）会分别被路由到不同的 reducer。这保证了：所有相同 key 的键值对一定会到达同一个 reducer。

partitioning 发生在 mapper 端——每个 mapper 维护 R 个输出缓冲区（或 R 个输出文件），mapper 每产生一个键值对，就根据 hash 写入对应的分区。

**2. Shuffle 的两个子阶段**

- **Shuffle（传输阶段）**：Reducer 通过 HTTP 从各个 mapper 所在节点拉取属于自己的分区数据。如果有 M 个 mapper 和 R 个 reducer，最坏情况需要 M x R 次网络传输。
- **Merge/Sort（排序阶段）**：Reducer 收到来自不同 mapper 的数据片段（segment），需要将它们合并为一个整体已排序的流。这通过多路归并（K-way merge）完成。

**3. 为什么 shuffle 经常是瓶颈？**

Shuffle 阶段涉及大量网络传输和数据序列化/反序列化。在 Google 早期集群中，shuffle 阶段可以占总作业时间的 30%-70%。优化手段包括：
- **Combiner 函数**：在 mapper 端做局部预聚合（类似 reducer，但运行在 mapper 节点），减少需要传输的数据量。例如在词频统计中，mapper 可以先本地合并 `("the", 1, 1, 1, ...)` 为 `("the", 537)`。
- **数据压缩**：mapper 输出在发送前压缩（如 LZO、Snappy），reducer 接收后解压，以 CPU 换网络带宽。
:::

# 代码：Group By Key
* `group-by-key` 过程的前置条件
    * group-by-key 过程在所有 map-reduce 管道中都会使用，不仅仅是这个例子。
      该 group-by-key 过程假设 mapper 的输出已经按键排序，使得相同键的多个副本被分组在一起，如下所示：

    ```sh
    myth22> cat anna-karenina.txt | ./word-count-mapper.py | sort
    ```

    * 上述管道产生以下（压缩的）输出：

    ```sh
    a 1
    a 1
    a 1
    a 1
    a 1 // plus 6064 additional copies of this same line
    ...
    zeal 1
    zeal 1
    zeal 1
    zealously 1
    zest 1
    zhivahov 1
    zigzag 1
    zoological 1
    zoological 1
    zoology 1
    zu 1
    ```

::: tip 重难点解析
**为什么需要排序？** MapReduce 框架中，mapper 和 reducer 之间有一个隐式的"shuffle & sort"阶段。排序（`sort`）的作用是将所有 mapper 产生的、键相同的 `(key, value)` 对聚集在一起。排序之后，reducer 只需要一次顺序扫描就能拿到某个 key 的所有值，而不需要做随机查找。这就是为什么真实的 MapReduce（如 Hadoop）中的 shuffle 阶段是性能瓶颈之一：它需要将大量数据通过网络传输并按 key 重新分区排序。
:::

::: tip 重难点解析
**外部排序 (External Sorting) —— 当数据量超过内存时**

在 CS110 的管道演示中，`sort` 命令在单机上运行，数据量假设可以装入内存。但在分布式 MapReduce 中，数据量可能达到 TB 甚至 PB 级别，远超单机内存——此时必须使用**外部排序**。

**核心思想**：将大数据集划分为可装入内存的数据块（chunk），逐块排序后写回磁盘，最后通过多路归并（K-way merge）合并所有有序块。

**算法步骤**：
1. **分块 (Divide)**：读取尽可能大的数据块（例如 1GB），在内存中排序（使用 quicksort/heapsort），将排序后的块写入磁盘为临时文件（称为 run）
2. **归并 (Merge)**：同时打开所有临时文件，使用最小堆从每个文件的当前位置读取键值对，每次弹出键最小的条目写入最终输出，并从其来源文件补充下一行
3. **多遍归并 (Multi-pass)**：如果临时文件数量超过单次可打开的文件描述符上限，需要多轮归并（如第一轮将 100 个文件归并成 10 个，第二轮将 10 个归并成 1 个）

**Hadoop 中的优化**：
- 每个 mapper 输出数据量 `mapreduce.task.io.sort.mb`（默认 100MB）——超出后在 mapper 端先做归并排序到磁盘
- Reducer 端将各 mapper 分区下载后做归并，使用 `mapreduce.task.io.sort.factor`（默认 10）控制归并的并发流数
:::

# 代码：Group By Key（续）
* `group-by-key` 过程的后置条件
    * 以下 Python 脚本是一个简短（但紧凑）的程序，它从按键排序的键值对输入流中读取数据，
      并输出相同的内容，不同之处在于所有具有相同键的行被合并为一行，其中所有值
      被合并为一个向量表示形式：

    ```python
    #!/usr/bin/env python
    from itertools import groupby
    from operator import itemgetter
    import sys

    def read_mapper_output(file):
        for line in file:
            yield line.strip().split(' ')

    def main():
        data = read_mapper_output(sys.stdin)
        for key, keygroup in groupby(data, itemgetter(0)):
            values = ' '.join(sorted(v for k, v in keygroup))
            print "%s %s" % (key, values)

    if __name__ == "__main__":
        main()
    ```

    * 特定问题 mapper 的排序输出可以输入到上述脚本，如下所示：

    ```python
    myth22> more anna-karenina.txt | ./word-count-mapper.py | sort | ./group-by-key.py
    ```

    * 这样做会产生如下结果：

    ```python
    a 1 1 1 1 1 // plus 6064 more 1's on this same line
    abandon 1 1 1 1 1 1
    abandoned 1 1 1 1 1 1 1 1 1
    abandonment 1
    abashed 1 1
    abasing 1
    aber 1
    abilities 1
    ...
    zaraisky 1 1 1 1
    zeal 1 1 1
    zealously 1
    zest 1
    zhivahov 1
    zigzag 1
    zoological 1 1
    zoology 1
    zu 1
    ```

::: tip 重难点解析
**Group-by-key 的本质**：Group-by-key 是 mapper 和 reducer 之间的桥梁。它的输入是 `(key, value)` 行的排序流，输出是 `(key, [value1, value2, ...])` 的聚合行。注意在这个阶段，我们仍然不计算总和——只是把相同 key 的值收集到一起。这种"分层解耦"的设计（map -> group -> reduce）允许每一层只关注一件事：map 做转换、group 做聚合、reduce 做最终的汇总计算。在工业实践中，这一步常被称为 shuffle，是 MapReduce 框架自动完成而无需用户编写的。

**`itertools.groupby` 的巧妙运用**：这段 Python 代码虽然短小，但利用了 Python 标准库中一个强大的迭代器工具。`groupby` 将连续的、键相同的行分组，这正是"已排序"这个前置条件发挥作用的地方——如果没有排序，相同的 key 可能分散在不同位置，`groupby` 就无法正确分组。
:::

::: tip 重难点解析
**MapReduce 的工业实践 —— Google 的原始设计**

Jeff Dean 和 Sanjay Ghemawat 在 2004 年发表了 MapReduce 论文（OSDI'04），其动机是现实工程问题：

**1. Google 的原始用例：构建搜索的倒排索引**

Google 需要处理爬虫收集的数十亿网页，构建一个倒排索引（inverted index）：对于每个单词，记录它出现在哪些文档中。工作流程：
- **Map**：解析每个网页，输出 `(word, document_id)` 对
- **Shuffle**：按 word hash 分区，将相同 word 的数据路由到同 reducer
- **Reduce**：对每个 word，将所有 document_id 聚合成倒排列表 `(word, [doc1, doc2, ...])`

这一处理需要数千台机器共同完成。在 MapReduce 框架出现前，Google 工程师每次都需要从零编写分布式数据处理代码——处理机器故障、数据分区、负载均衡等。MapReduce 将这些通用问题抽象为框架逻辑，用户只需提供 `map()` 和 `reduce()` 两个函数。

**2. 故障处理 (Fault Tolerance)**

在大规模集群中，机器故障是常态而非例外。MapReduce 的容错策略：
- **Master 追踪所有 worker**：定期 ping，无响应则标记为 failed
- **Map task 重执行**：已完成的 map task 在 worker 故障后也需要重新执行（因为 mapper 输出存储在本地磁盘，故障后不可达）
- **Reduce task 重执行**：已完成但未写入最终输出文件的 reduce task 需要重执行（最终输出写入 GFS，已经持久化完成的则不需要重执行）
- **Map/Reduce 的幂等性要求**：框架假设 map 和 reduce 函数是确定性的（相同输入产生相同输出），这使得安全重执行成为可能。如果用户函数有副作用（如写入外部数据库），重执行会导致重复数据。

**3. 拖后腿任务 (Stragglers) 与推测执行**

在大规模作业中，总会有少数任务执行极慢（由于机器负载高、网络故障、磁盘错误等）。这些"straggler"会拖累整个作业的完成时间。MapReduce 的解决方案：**推测执行 (speculative execution)**——当作业接近完成时，master 调度备份任务在空闲 worker 上执行相同的 straggler 任务。先完成的那个结果被采用。

**4. 分布式文件系统 (GFS/HDFS) 的角色**

MapReduce 的高效性依赖于其底层分布式文件系统：
- **数据局部性 (data locality)**：GFS 将输入文件分块（通常 64MB/128MB 每块），每个块复制 3 份到不同节点。Master 在调度 map task 时，优先选择已经持有该输入块副本的节点——避免网络传输，直接从本地磁盘读取。这遵循"将计算移动到数据身边"的原则。
- **输出持久化**：Reducer 输出写入 GFS（3 副本），保证数据安全。Mapper 中间输出则仅存本地（不需要跨节点持久化，因为故障后可以重执行）。

**5. CS110 作业 8 的连接**

你在作业 8 中实现的 MapReduce 框架是对这些概念的教学级别实现：你会处理文件分区、map worker 和 reduce worker 的调度、shuffle 阶段的数据路由，以及容错的基本形式——所有这些都与上述工业实践直接对应。
:::

# 代码：Reducer
* Reducer 是什么？
    * Reducer 是一个特定问题的程序，它期望一个排序的输入文件，其中每行
      是由 `group-by-key` 脚本产生的键/值向量对。
    * 考虑以下代码：

    ```python
    #!/usr/bin/env python
    import sys

    def read_mapper_output(file):
        for line in file:
            yield line.strip().split(' ')

    def main():
        data = read_mapper_output(sys.stdin)
        for vec in data:
            word = vec[0]
            count = sum(int(number) for number in vec[1:])
            print "%s %d" % (word, count)

    if __name__ == "__main__":
        main()
    ```

    * 如果使用以下命令行管道链，上述 reducer 可以接收先前提供的 mapper
      的已排序、已按键分组的输出：

    ```python
    myth22> more anna-karenina.txt | ./word-count-mapper.py | sort | ./group-by-key.py | ./word-count-reducer.py
    ```

    * 上述管道可执行文件链产生如下结果：

    ```python
    a 6069
    abandon 6
    abandoned 9
    abandonment 1
    abashed 2
    abasing 1
    aber 1
    abilities 1
    ...
    zaraisky 4
    zeal 3
    zealously 1
    zest 1
    zhivahov 1
    zigzag 1
    zoological 2
    zoology 1
    zu 1
    ```

::: tip 重难点解析
**MapReduce 的完整数据流**：整个管道的处理流程可以概括为：

1. **Map**：`(原始文本)` -> `(word, 1)` 对，每个单词独立处理
2. **Sort**：将所有 `(word, 1)` 对按 key 排序，使得相同单词的行连在一起
3. **Group-by-key**：`(word, 1), (word, 1), ...` -> `(word, [1, 1, 1, ...])`，将同一单词的计数收集为列表
4. **Reduce**：`(word, [1, 1, 1, ...])` -> `(word, 6069)`，对列表求和得到最终词频

**为什么 reducer 可以并行化？** 在真实分布式环境中，不同的 reducer 处理不同范围的 key（例如按单词首字母分桶）。每个 reducer 只需要自己那部分经过排序和分组的数据，不同 reducer 之间完全没有依赖关系。这是 MapReduce 能够水平扩展到数千台机器的关键原因。你在作业 8 中将亲手实现这一框架。
:::
