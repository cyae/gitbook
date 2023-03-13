## MapReduce 解决了一个什么问题?

2004 年谷歌提出了 MapReduce, 在此之前谷歌程序员面对的大规模数据集,常常需要编程实现:

1. 统计某个关键词的现的频率,计算 pageRank
2. 对大规模数据按词频排序
3. 对多台机器上的文件进行 grep 等

这些工作不可能在一台机器上完成(否则也不能称之为大规模),因此谷歌的程序员每次编写代码都需要处理,多机并行协同,网络通信,处理错误,提高执行效率等问题。

这些问题使得开发效率严重降低,因此为了治理这一现象导致的复杂度,Jeff Dean,设计了一种新的编程模型 MapReduce。

所以 MapReduce 就是为了编写在普通机器上运行的大规模并行数据处理程序而抽象出来的编程模型,为解决多机并行协同,网络通信,处理错误,提高执行效率 等通用性问题的一个编程框架。

## MapReduce 究竟是什么?

![[image 1.png]]

来自于对 Lisp 语言中 map/reduce 原语的借鉴,经过谷歌大量重复的编写数据处理类的程序,发现所有数据处理的程序都有类似的过程:

> 将一组输入的数据应用 map 函数返回一个 k/v 对的结构作为中间数据集,并将具有相同 key 的数据输入到一个 reduce 函数中执行,最终返回处理后的结果。

这一计算模型的优势在于非常利于并行化,map 的过程可以在多台机器上而机器之间不需要相互协调。reduce 的执行同样不需要协调,没有相互依赖可并行执行。 所有的依赖性和协调过程都被隐藏在 map 与 reduce 函数的数据分发之间。因此这里有一个很重要的细节就是 map 和 reduce 都是可以各自并行执行,但 reduce 执行的前提条件是所有的 map 函数执行完毕。

作为一个编程模型，mapreduce 如此工作 但是这一过程想要落地就必须要考虑工程上的永恒问题 可靠性 与 性能。 大规模的机器产生局部性的失败是一种必然现象,因此如果 mapreduce 分散在多台机器上执行,其中一个 map 任务失败导致整个处理过程都需要重新计算这是不可接受的。 因此 对于大规模分布式程序来说 能够应对局部性失败的容错性 与性能同等重要。这是一个必要的问题,分布式的容错性本质上就是如何在不可靠的硬件之上构建可靠的软件。

所以 作为一个成熟的工业级实现 MapReduce 就是一个 利用普通机器组成的大规模计算集群进行并行的,高容错,高性能的数据处理函数框架。

```java
map(String key,String value):
    // key: 文档名
    // value: 文档内容
    for each word w in value:
        EmitIntermediate(w,"1");

reduce(Stringkey, Iterator values):
    // key: 一个单词
    // value: 计数值列表
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
```

## MapReduce 的应用场景是什么?

这里例举了一些有趣的程序，它们都可以很轻松的用 MapReduce 模型表达。

- **分布式 Grep**：map 函数在匹配到给定的 pattern 时输出一行。reduce 函数只是将给定的中间数据复制到输出上。
- **URL 访问频次统计**：map 函数处理网页请求的日志，对每个 URL 输出〈URL, 1〉。reduce 函数将相同 URL 的所有值相加并输出〈URL, 总次数〉对。
- **倒转 Web 链接图**：map 函数在 source 页面中针对每个指向 target 的链接都输出一个〈target, source〉对。reduce 函数将与某个给定的 target 相关联的所有 source 链接合并为一个列表，并输出〈target, list(source)〉对。
- **每个主机的关键词向量**：关键词向量是对出现在一个文档或一组文档中的最重要的单词的概要，其形式为〈单词, 频率〉对。map 函数针对每个输入文档（其主机名可从文档 URL 中提取到）输出一个〈主机名, 关键词向量〉对。给定主机的所有文档的关键词向量都被传递给 reduce 函数。reduce 函数将这些关键词向量相加，去掉其中频率最低的关键词，然后输出最终的〈主机名, 关键词向量〉对。
- **倒排索引**：map 函数解析每个文档，并输出一系列〈单词, 文档 ID〉对。reduce 函数接受给定单词的所有中间对，将它们按文档 ID 排序，再输出〈单词, list(文档 ID)〉对。所有输出对的集合组成了一个简单的倒排索引。用户可以很轻松的扩展这个过程来跟踪单词的位置。
- **分布式排序**：map 函数从每条记录中提取出 key，并输出〈key, 记录〉对。reduce 函数不改变这些中间对，直接输出。这个过程依赖于 4.1 节介绍的划分机制和 4.2 节介绍的排序性质。

## MapReduce 是如何实现的?

![[501ba643-7318-4317-8c47-a5eb2be534c1.svg]]

1. 用户程序中的 MapReduce 库首先将输入文件切分为 M 块，每块的大小从 16MB 到 64MB（用户可通过一个可选参数控制此大小）。然后 MapReduce 库会在一个集群的若干台机器上启动程序的多个副本。
2. 程序的各个副本中有一个是特殊的——主节点，其它的则是工作节点。主节点将 M 个 map 任务和 R 个 reduce 任务分配给空闲的工作节点，每个节点一项任务。
3. 被分配 map 任务的工作节点读取对应的输入区块内容。它从输入数据中解析出 key/value 对，然后将每个对传递给用户定义的 map 函数。由 map 函数产生的中间 key/value 对都缓存在内存中。
4. 缓存的数据对会被周期性的由划分函数分成 R 块，并写入本地磁盘中。这些缓存对在本地磁盘中的位置会被传回给主节点，主节点负责将这些位置再传给 reduce 工作节点。
5. 当一个 reduce 工作节点得到了主节点的这些位置通知后，它使用 RPC 调用去读 map 工作节点的本地磁盘中的缓存数据。当 reduce 工作节点读取完了所有的中间数据，它会将这些数据按中间 key 排序，这样相同 key 的数据就被排列在一起了。同一个 reduce 任务经常会分到有着不同 key 的数据，因此这个排序很有必要。如果中间数据数量过多，不能全部载入内存，则会使用外部排序。
6. reduce 工作节点遍历排序好的中间数据，并将遇到的每个中间 key 和与它关联的一组中间 value 传递给用户的 reduce 函数。reduce 函数的输出会写到由 reduce 划分过程划分出来的最终输出文件的末尾。
7. 当所有的 map 和 reduce 任务都完成后，主节点唤醒用户程序。此时，用户程序中的 MapReduce 调用返回到用户代码中。
8. 主节点维持多种数据结构。它会存储每个 map 和 reduce 任务的状态（空闲、处理中、完成），和每台工作机器的 ID（对应非空闲的任务）。
9. 主节点是将 map 任务产生的中间文件的位置传递给 reduce 任务的通道。因此，主节点要存储每个已完成的 map 任务产生的 R 个中间文件的位置和大小。位置和大小信息的更新情况会在 map 任务完成时接收到。这些信息会被逐步发送到正在处理中的 reduce 任务节点处。

## 这会存在什么问题?该如何改进?

### 大规模的集群如何容错?

1. **执行 map 任务的节点** 通过周期性的 ping-pong 心跳机制 main 节点感知到 map 节点的状态,如果心跳超时认为节点失败。 重新将当前 worker 上的 map task 传递给其他 worker 节点完成。主节点要维护一个任务队列来记录哪些任务还未完成,同时记录已经完成的任务的状态,来感知到当前 mapreduce 任务处于一个怎样的生命周期。
2. **执行 reduce 任务的节点** reduce 产出的文件是一个持久性的文件,存在副作用,因此每次 reduce 被重新分配后要重命名一个新的文件,防止与损坏的文件冲突。
3. **主节点** 如果主节点失败了怎么办?对于任务执行的元数据产生的中间态可以保持在一个恢复点文件中,当节点崩溃重启后可以从最近的一个恢复点重新执行 mapreduce 任务。
4. **副作用** 对于所有持久化的操作，不可避免的会产生副作用。导致这种副作用的根本原因在于不能原子的提交状态。 因此解决方案就是保证 map 与 reduce 产生的文件在执行过程中先存储在临时文件中(以时间戳命名) 等到提交文件时 将其原子的重命名为最终文件(linux 内核中 重命名操作是有原子性的保证的)。

### 如何加速并行处理的过程?

#### 利用局部性

在分布式系统中，网络带宽通常是稀缺资源，很容易成为系统瓶颈。 在 MapReduce 的任务中 至少需要 M\*R 次的网络传输，才能将中间文件发送给 reduce 所在的 worker 节点上。同时把输入文件发送给 map 任务所在的 worker 也是非常消耗网络带宽的事情。 但是我们可以通过仅将 map 任务分配给本来就有所有输入文件的节点上,来减少一次网络调用使得性能得到提升。 同时还可以使用一些流处理的思路优化 shuffle 的过程,那就是 当一个 map 任务完成后通知 main 进程后,main 进程立即通知 reduce 任务拉取其中一份文件,而不必等到所有 map 任务全部执行完毕后进行网络传输而提高了并行性。

#### 任务的粒度

应该配置多少个任务执行 map，多少个任务执行 reduce？任务拆分过多加剧网络传输的负担。 而任务拆分的过少又会导致并行度不够而降低整体的执行效率。

为此 一些经验性的配置是 map 任务通常为输入文件总大小除以 64M 的值(这源于底层的分布式文件系统是以 64m 为一个 chuck 进行存储的),reduce 的数量通常是 map 任务的一半。同时 为了发挥机器本身的多核特性,一台机器上可以指定多个 map reduce 任务来执行 通常是任务总数的百分之一。

#### 备用任务

如果最后的几个任务执行时间过长怎么办?存在这种 case,10 个任务用 5 分钟完成了其中 9 个,但最后一个任务因为当前机器的负载过高花费了 20 分钟执行完毕,这么整个任务的执行周期就是 20 分钟。 如何能应对这一问题呢?

当仅剩下 1%的任务时,可以启动备用任务,即同时在两个节点上执行相同的任务。这样只要其中一个先返回即可结束整个任务,同时释放未完成的任务所占用的资源。

## 一些有价值的技巧

### 划分函数

按 map 函数输出的 key 决定当前的 kv 被划分到哪一个中间文件中。 工程经验上看,我们有时不仅仅要求其平衡划分，我们还可能要求其可以按照某种 kv 的元数据信息进行分区。 比如将一个主机下的文件划分到一起等定制的能力。

### 顺序保证

有时我们需要支持对数据文件按 key 进行随机访问,或者有序输出,并且为了减少 reduce 任务的负担。 输出的每个 map 任务的中间文件都是保证按 key 递增有序的。

### 合并函数

如果 map 函数输出的同一种 key 的内容过多怎么办?在词频统计中 一些常见的谓词频率非常高。这就会产生非常多的类似<are, 1>这样的 kv 对,导致中间文件的大小严重不均衡。网络传输带宽加剧。

解决这一问题的方案在于对传输的中间文件进行预处理,在 shuffle 之前对其进行一次 reduce 操作,将这种<are,1>进行合并变为<are,100000>，减少需要传输数据的大小节省网络带宽。

### 输入和输出类型

有时 map 、reduce 函数的输入类型并不是纯文本的，这种情况下如果能输入输出结构化类型是一种理想的情况。这需要在 map reduce 函数之前实现一个类型适配器的组件,同时也可以实现 read 接口不仅仅从文件中读取数据也可以从数据库等数据源来读取。

### 确定性问题

如果 mapreduce 执行过程是不确定那就会产生问题,也就说 mapreduce 执行的前提假设是输入文件是静态的,是有界的,不能在执行过程中发生改变。如果发生改变那么 mapreduce 的计算结果并不保证其正确性。

### 略过坏记录

数据集并不能保证完全是正确的,如果有一行记录是错误的导致 map 任务崩溃,不断的重试最终使得整个程序不能结束。因此必然需要跳过这一段错误的记录。 如何跳过呢？

每个 map 任务要捕获异常,通过安装信号的方式,在程序退出之前执行安装的信息函数把执行到的文件的行号 offset 等信息发送给主节点。 主节点在下次调度的时候 将这些 offset 处的记录作为黑名单列表传递给新的 map 任务，执行时会对此处的记录跳过执行。

### 本地执行

如何调试分布式的程序?MapReduce 通常在几千台机器上执行,这其中如果存在逻辑问题非常难以调试,因此为 MapReduce 提供单机多进程的实现可以有效的进行本地 debug 发现问题。

### 状态信息

分布式系统的可观测性尤为重要, 通过在 ping-pong 过程中将每个 worker 节点的工作状态进行上报,存储在 main 进程中，并提供可访问的网页来展示系统运行的信息。即可实现可解释性。

### 计数器

如何在 map reduce 的处理进行埋点统计？以实现用户自定义的指标监控? 需要创建一个原子计数器的对象,由用户在 map 和 reduce 的函数中 在发生某些事件时 对其进行累加。 并通过 ping-pong 的心跳中将数据携带回 main 进程 累加进去，但要注意每次消息的幂等性来保证不会导致重复累加或者少累加了计数的情况。

```c++
#include "mapreduce/mapreduce.h"

// User's map fuction
class WordCounter: public Mapper {
public:
    virtual void Map(const MapInput &input) {
        const string &text = input.value();
        const int n = text.size();
        for (int i = 0; i < n; ) {
            // Skip past leading whitespace
            while ((i < n) && isspace(text[i]))
                ++i;

            // Find word end
            int start = i;
            while ((i < n) && !isspace(text[i]))
                ++i;

            if (start < i)
                Emit(text.substr(start, i-start), "1");
        }
    }
};
REGISTER_MAPPER(WordCounter);

// User's reduce function
class Adder: public Reducer {
    virtual void Reduce(ReduceInput *input) {
        // Iterate over all entries with the
        // same key and add the values
        int64_t value = 0;
        while (!input->done()) {
            value += StringToInt(input->value());
            input->NextValue();
        }

        // Emit sum for input->key()
        Emit(IntToString(value));
    }
};
REGISTER_REDUCER(Adder);

int main(int argc, char **argv) {
    ParseCommandLineFlags(argc, argv);

    MapReduceSpecification spec;

    // Store list of input files into "spec"
    for (int i = 1; i < argc; ++i) {
        MapReduceInput *input = spec.add_input();
        input->set_format("text");
        input->set_filepattern(argv[i]);
        input->set_mapper_class("WordCounter");
    }

    // Specify the output files:
    //    /gfs/test/freq-00000-of-00100
    //    /gfs/test/freq-00001-of-00100
    //    ...
    MapReduceOutput *out = spec.output();
    out->set_filebase("/gfs/test/freq");
    out->set_num_tasks(100);
    out->set_format("text");
    out->set_reducer_class("Adder");

    // Optional: do partial sums within map
    // tasks to save network bandwidth
    out->set_combine_class("Adder");

    // Tuning parameters: use at most 2000
    // machines and 100MB of memory per task
    spec.set_machines(2000);
    spec.set_map_megabytes(100);
    spec.set_reduce_megabytes(100);

    // Now run it
    MapReduceResult result;
    if (!MapReduce(spec, &result))
        abort();

    // Done: 'result' structure contains info
    // about counters, time taken, number of
    // machines used, etc.

    return 0;
}
```
