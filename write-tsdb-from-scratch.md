# Writing a Time Series Database from Scratch

> 译注：原文地址：https://fabxc.org/tsdb/，由 Prometheus 核心开发者 Fabian Reinartz 写作于2017-04-20。
> 由于本文篇幅较长，就不做逐字逐句地翻译了，挑选其中重要精华部分翻译。

Prometheus 的存储层在过去已经展现出了出色的性能，单机可以摄入高达每秒一百万的样本和数百万的时序，而只需占用较小的磁盘空间。尽管目前的存储层已经工作得很好了，我还是提出了一个新设计的存储子系统来弥补现有方案的缺陷以及应对更高数量级的数据规模。

## 问题，问题，问题空间

首先，快速概述下我们要实现的东西及其引发的关键问题。我们来看下 Prometheus 目前的方案，它在哪些地方做得好，还有哪些问题是要用新的设计来解决的。

### 时序数据

我们有一个随着时间推移收集数据的系统。

```
identifier -> (t0, v0), (t1, v1), (t2, v2), (t3, v3), ....
```

每个**数据点**是**时间戳**和**值**的构成的元组。为了监控的目的，时间戳是一个整型，而值可以是任意数字。64位的浮点数可以很好地表示counter和gauge的值，所以我们将其作为值的数据类型。一系列时间戳**严格单调递增**的数据点是一个**序列**，这可以用一个**标识符（identifier）**来表示。我们用的标识符是指标名称和*标签维度*的字典。标签维度将单个指标的度量空间分割开。每个指标名加上唯一的标签集就是它自己的*时间序列*，并且有一个数值流和时序联系在一起。

这是一个典型的序列标识符集合，它是用于请求计数的指标的一部分：

```
requests_total{path="/status", method="GET", instance=”10.0.0.1:80”}
requests_total{path="/status", method="POST", instance=”10.0.0.3:80”}
requests_total{path="/", method="GET", instance=”10.0.0.2:80”}
```

我们来简化这个表示形式：指标名称也可以看作是另一个指标维度 —— `__name__`。在查询时，指标名可能会被特殊处理，但这和存储方式无关。

```
{__name__="requests_total", path="/status", method="GET", instance=”10.0.0.1:80”}
{__name__="requests_total", path="/status", method="POST", instance=”10.0.0.3:80”}
{__name__="requests_total", path="/", method="GET", instance=”10.0.0.2:80”}
```

当查询时序数据时，我们希望通过标签来选择序列。在最简单的情况中，`{__name__="requests_total"}` 选择所有属于`requests_total`指标的序列。对所有选定的序列，我们获取在给定的时间窗口的数据点。

在更复杂的查询中，我们可能想迅速选取出满足若干个**指标选择器（label selectors）**的序列，并且能够表示比相等关系更加复杂的条件。比如，不等（`method!="GET"`）或者正则表达式（`method=~"PUT|POST"`）。

这就定义了存储的数据和它是如何被召回的。

### 垂直和水平

在简化的视角中，所有的数据点可以置于一个二维的平面上。水平维度表示时间，序列标识符在垂直维度上扩展。

```
series
  ^   
  │   . . . . . . . . . . . . . . . . .   . . . . .   {__name__="request_total", method="GET"}
  │     . . . . . . . . . . . . . . . . . . . . . .   {__name__="request_total", method="POST"}
  │         . . . . . . .
  │       . . .     . . . . . . . . . . . . . . . .                  ... 
  │     . . . . . . . . . . . . . . . . .   . . . .   
  │     . . . . . . . . . .   . . . . . . . . . . .   {__name__="errors_total", method="POST"}
  │           . . .   . . . . . . . . .   . . . . .   {__name__="errors_total", method="GET"}
  │         . . . . . . . . .       . . . . .
  │       . . .     . . . . . . . . . . . . . . . .                  ... 
  │     . . . . . . . . . . . . . . . .   . . . . 
  v
    <-------------------- time --------------------->
```

Prometheus 通过周期性地抓取时间序列的当前值来获取数据点。被获取数据的实体称为**对象（target）**。因此，写模式是完全**垂直并且高并发**的，因为各个对象的样本是被独立地摄取（ingest）的。

举个例子，单个 Prometheus 实例从以万计的对象采集数据点，而每个对象暴露了成百上千的不同的时序。

在每秒采集数百万数据点的规模下，**批量写**毋庸置疑是达到性能要求所必须的。将零散的数据点写入到磁盘会非常慢。因此我们想**按顺序**写入更大的块（chunk）数据。

对于机械旋转式的磁盘这是意料之中的事实，因为它们的磁头必须一直移动到不同的扇区上。尽管SSD是以快速的随机写而著称的，实际上它们并不能修改单个的字节，而只能以4KiB或者更大的页来写入。这就意味着写入16字节的样本和写入完整的4KiB的页是一样的。这种行为就是所谓的[**写放大**](https://en.wikipedia.org/wiki/Write_amplification)的一部分，这会导致SSD的磨损 —— 所以这不仅仅是慢的问题，还会损伤硬件。可以阅读["Coding for SSDs"](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)系列博客来获取更多信息。我们只需要记住：不论是旋转式磁盘还是SSD，**顺序和批量写入**都是理想的写模式。这是一个要坚持的简单原则。

查询模式和写模式区别很大。我们可以查询单个时序的单个数据点，10000个时序的单个数据点，单个时序几个星期的数据点，10000个时序几个星期的数据点，等等。在上述的二维平面上，**查询不是完全水平或者垂直的，而是两者结合的矩形形式**。

对于已知的查询，[聚合规则（Recording rules）](https://prometheus.io/docs/practices/rules/)可以缓解此问题，但这对于临时的查询并不是一个通用的解决方案。临时的查询也需要好的性能。

我们想要批量地写，但是我们得到的唯一的批次是跨越时序的数据点的垂直集合。当查询一个时序在某个时间窗口上的数据点时，不仅找到各个数据点在哪是很难的，我们还要读取磁盘上很多随机位置。单次查询可能会涉及到数百万样本，即使在最快的SSD上也会很慢。查询还会从磁盘上获取比16字节样本更多的数据。SSD会加载整页，HDD至少会读取整个扇区。不管是哪个，我们都会浪费宝贵的读吞吐量。

所以理想情况中，**相同时序的样本按顺序存储**，这样我们就可以用尽可能少的读取次数来扫描数据。我们只需要知道这个序列是从哪开始的就可以读取数据点。

在将采集数据写入到磁盘的理想模式和为了高效查询的数据布局之间，显然有着巨大的矛盾。这就是我们的TSDB要解决的**基本问题**。

#### 目前的解决方案

来看一下 Prometheus 目前的存储（我们称之为 V2）是如何解决这个问题的。

对于每个时序我们创建一个文件，文件中按顺序包含了时序样本。因为每几秒钟将单个数据点追加到所有这些文件上代价很高，我们对于每个序列把样本在内存中“攒成”1KiB大小的块（chunks），然后在块满的时候追加写到文件中。这个方法解决了大部分问题。写是批量的了，样本也按顺序存储。这还启用了非常高效的压缩格式，这种压缩格式基于样本和它前一个样本相比改变很小的性质。[Facebook Gorilla TSDB 论文](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)中描述了一个相似的基于chunk的方法，并且介绍了一种压缩格式可以把16字节的样本压缩到平均1.37字节大小。V2存储使用了多种压缩格式，包括了Gorilla的一种变式。

```
   ┌──────────┬─────────┬─────────┬─────────┬─────────┐           series A
   └──────────┴─────────┴─────────┴─────────┴─────────┘
          ┌──────────┬─────────┬─────────┬─────────┬─────────┐    series B
          └──────────┴─────────┴─────────┴─────────┴─────────┘ 
                              . . .
 ┌──────────┬─────────┬─────────┬─────────┬─────────┬─────────┐   series XYZ
 └──────────┴─────────┴─────────┴─────────┴─────────┴─────────┘ 
   chunk 1    chunk 2   chunk 3     ...
```

尽管这种基于chunk的方法很好，对于每个时序都有一个文件也带了一些问题：

* 实际上我们需要比正在采集的时间序列的数量更多的文件。在“序列搅动”部分有更多关于此的论述。几百万文件迟早会把文件系统的 [inodes](https://en.wikipedia.org/wiki/Inode) 消耗殆尽。这种情况下，我们只能通过重新格式化磁盘来恢复，而这种做法的侵入性和破坏性都过大。我们不想为了适配单个应用而格式化磁盘。

* 每秒有几千个chunk达到完整状态要被持久化，这就需要每秒数千次写磁盘。即使这可以通过将一个时序的几个完整的chunk一批写入来缓解，这反过来又增加了待持久化数据总体的内存占用。

* 将所有文件打开来读写是不可行的。因为大约99%的数据在24小时后不会被查询了。如果还会被查询，我们需要打开几千个文件，找到并将相关的数据点读入内存，然后关闭文件。由于这会导致高的查询延迟，数据块被大幅度缓存了，而这又引发了后续在“资源消耗”部分论述的问题。

* 最终，旧的数据要被删除，数据要从数百万的文件中移除掉。这意味着删除实际上是写密集型操作。而且，遍历数百万文件并分析它们的过程需要几个小时。当这个过程完成时，可能又要重新开始了。删除旧文件还会进一步导致SSD的写放大。

* 目前积累的 chunks 只存放在内存中。如果应用崩溃了，数据会丢失。为避免这一点，内存状态会周期性地检出到磁盘，这可能需要比我们能接受的数据丢失的时间窗口更长的耗时。恢复检查点可能需要几分钟，导致很长的重启时间。


现有设计中的关键点是 chunk 的概念，这一点我们肯定要保持下去。最近的 chunks 总是在内存中，这点也不错。毕竟，最近的数据是被查询最多的。

每个时序有一个文件，这一点我们需要找到替代的办法。

### 序列搅动 Series Churn

在 Prometheus 的上下文中，我们使用 *series churn* 这个术语来描述：一些时序不再活跃，即没有新的数据点，取而代之的是另一些新的活跃的时序。

举个例子，一个给定的微服务实例暴露的所有时序都有对应的“instance”标签，来标示它的来源。如果我们进行一次微服务的滚动更新，所有的实例都升到新版本，那就会发生序列搅动。在更加动态的环境中，这类事件可能每小时都在发生。集群编排系统，比如K8S，允许应用持续的自动扩缩容和频繁的滚动更新，每天可能创建成千上万的新应用实例，伴随着的是全新的时序。

```
series
  ^
  │   . . . . . .
  │   . . . . . .
  │   . . . . . .
  │               . . . . . . .
  │               . . . . . . .
  │               . . . . . . .
  │                             . . . . . .
  │                             . . . . . .
  │                                         . . . . .
  │                                         . . . . .
  │                                         . . . . .
  v
    <-------------------- time --------------------->
```

所以即使整个基础设施大致上保持规模不变，随着时间推移在数据库中的时序数量也会呈线性增长。一个 Prometheus 服务能够采集1000万时序的数据，但是如果数据需要从十亿时序中寻找的话，查询性能会大打折扣。

#### 目前的解决方案

现在的Prometheus V2存储有一个基于 **LevelDB** 的对所有存储的序列的索引。它能够查询包含一个给定的标签对的序列，但是缺少一个可扩展的方法来对不同标签选择的结果做结合。

例如，选择所有带有标签 `__name__="requests_total"` 的序列可以高效做到，但是选择所有带有标签 `instance="A" AND __name__="requests_total"` 的序列就会有扩展性问题。我们后面会说到是什么导致了这个问题，必须用什么方法来提高查询效率。

实际上，这个问题是寻找更好的存储系统的动机。Prometheus 需要一个改良的索引方法来快速搜索大量的时序。

### 资源消耗

资源消耗是尝试扩展 Prometheus 的一贯主题之一。但实际上困扰着用的户并不是绝对的资源消耗量。事实上，考虑到 Prometheus 的配置要求，它是在管理着难以置信的吞吐量。这个问题更多的是它在面对变化时的不可预测性和不稳定性。V2存储中会慢慢累积样本数据的chunks，导致内存消耗量随着时间爬升。当chunks完整时，它们会写入到磁盘并从内存中驱逐掉。最终，Prometheus的内存占用量会达到一个稳定状态。这个状态会持续到被监控环境的变化 —— 每次扩展应用或者做滚动更新时，*series churn* 会增加Prometheus的内存、CPU的占用和磁盘IO。

如果这种变化是不断发生的，那么它（译注：Prometheus的资源消耗）最终又会达到一个稳定状态，但是会比一个更加静态的环境要高很多。转换周期经常长达数小时，很难判断最大的资源使用量是多少。


每个时序有一个文件的做法也使得单次查询很容易导致 Prometheus 进程崩溃。当查询的数据没有缓存在内存中时，要打开被查询的序列的文件，包含了相关数据点的chunks被读到内存。如果数据量超过了可用内存，Prometheus 会 OOM 而突然退出。

查询完成后，加载的数据可以被释放掉，但是通常它会被缓存更长时间来更快地响应对相同数据的后续查询。后者显然是一个好的做法。


最后，我们看到了在SSD上下文中的写放大问题以及Prometheus是如何通过批量写来缓解这一问题的。尽管如此，在几个地方它仍然会导致写放大，因为太小的批次和数据没有和页边界准确对齐。实际上对于更大的 Prometheus 服务，有观察到硬件寿命的减少。对于具有高写吞吐量的数据库应用程序来说，这是相当正常的情况，但我们应该关注是否能够减轻这种情况。

## 重新开始

到目前为止，我们已经很好地了解了问题域、V2存储是如何解决它的，以及它的设计存在哪些问题。我们也看到了一些很好的概念，对此我们想要无缝适应。V2存储的相当一部分问题可以通过改进和部分重新设计来解决，但为了好玩（当然，在仔细评估了我的想法之后），我决定尝试编写整个时间序列数据库 —— 从零开始，即向文件系统写入字节。

重点关注的性能和资源占用是所选存储格式的直接结果。我们必须为数据找到正确的算法和磁盘布局，以实现性能良好的存储层。

下面我会抄捷径，直接到解决方案 —— 跳过头痛、失败的想法、无尽的草图、眼泪和沮丧。

### V3 —— 宏观设计

存储的宏观布局是什么？简单地说，在数据目录中执行 `tree` 命令展示的就是布局了。

```shell
$ tree ./data
./data
├── b-000001
│   ├── chunks
│   │   ├── 000001
│   │   ├── 000002
│   │   └── 000003
│   ├── index
│   └── meta.json
├── b-000004
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
├── b-000005
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
└── b-000006
    ├── meta.json
    └── wal
        ├── 000001
        ├── 000002
        └── 000003
```

在最顶层，我们有一系列编号的 blocks，用`b-`为前缀。每个 block 中有一个包含索引的文件和一个装有更多编号文件的“chunks”目录。“chunks”目录中就是许多序列数据点的原始chunks。和V2类似，这样做使得读取一个时间窗口内的时序数据非常简单，并且可以应用同样的高效压缩算法。这个概念已经被证明了可以很好地工作，我们无需改变它。显然不会再有每个序列一个文件了，取而代之的是为数不多的包含了chunks的文件。

“index”文件的存在应该是意料之内的。我们先假设它包含了一些黑科技，让我们可以寻找标签，标签对应的可能的值，整个时序和容纳它们数据点的chunks。


但是为什么会有好几个包含了索引和chunk文件的目录呢？为什么最后一个目录中有“wal”目录？理解了这两个问题就可以解决我们90%的问题了。

> 译注：现在（2021-09）的 Prometheus TSDB 数据目录和本文所述有些差别：（1）每个block目录中还有tombstones文件；（2）`wal`目录不在最新一个block目录中，而是一个和block平级的目录；（3）还有 `chunks_head` 目录。


#### 许多小型数据库

我们将*水平*维度，也就是时间维度，分割成不相互重叠的 blocks。每个block就像是一个完整而独立的数据库，包含了它的时间窗口内的所有时序数据。因此，它有自己的索引和chunk文件。

```

t0            t1             t2             t3             now
 ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │           │  │           │  │           │  │           │                 ┌────────────┐
 │           │  │           │  │           │  │  mutable  │ <─── write ──── ┤ Prometheus │
 │           │  │           │  │           │  │           │                 └────────────┘
 └───────────┘  └───────────┘  └───────────┘  └───────────┘                        ^
       └──────────────┴───────┬──────┴──────────────┘                              │
                              │                                                  query
                              │                                                    │
                            merge ─────────────────────────────────────────────────┘
```

每个block的数据是不可变的。当然，我们必须要能够在采集新数据时将新的序列和样本增加到最近的block中去。对于这个block，所有新数据被写入到一个内存中的数据库，它提供了和持久化blocks一样的查询性质。这个内存中的数据结构可以被高效地更新。为了防止数据丢失，所有到来的数据会被写入到临时的*预写式日志*中，也就是“wal”目录中的文件，我们可以在重启时从中恢复内存中的数据库。

所有这些文件都有它们自己的序列化格式，其中有许多我们可以预料到的东西：很多flags，offsets，varints，和CRC32校验和。这些东西想起来很好玩，读起来却很无趣。


这个布局使得我们可以把查询扇出（fan out）到所有和被查询的时间范围相关的blocks。每个block查出的部分结果被合并成完整的结果。


这个水平分区带来了一些很棒的能力：

* 当查询一个时间范围时，我们可以很简单地忽略那些不在该范围内的数据blocks。通过减少要检查的数据，它也解决了 *series churn* 的问题。

* 当完成一个block时，我们通过顺序写一些更大的文件，将数据从内存中数据库持久化。我们也避免了写放大，不管是SSD还是HDD。

* 我们保持了V2的一个良好特性，即最近的chunks，也就是被查询次数最多的，总是在内存中。

* 我们不再把chunk大小限制在固定的1KiB，这样可以更好地和磁盘上数据对齐。我们可以选择任何对单个数据点最合适的大小并选取压缩格式。

* 删除旧的数据变得非常简单迅速。我们只用删除目录就行。要知道在旧的存储中，我们必须分析并重写百万计的文件，这可能需要花费数小时。


每个block目录还有一个 `meta.json` 文件。它记录了关于该block的人类可读的信息，可以帮助理解存储和内部数据的状态。

##### mmap

从数百万个小文件到为数不多的更大的文件使得我们可以将所有的文件打开而不需要太大开销。这就解锁了 [`mmap(2)`](https://en.wikipedia.org/wiki/Mmap) 的使用，这个系统调用让我们可以

> todo

#### 压缩

这个存储要周期性地“切下”一个新的block，并将前一个已经完整的block写到磁盘。仅当这个block成功持久化时，用来恢复内存中blocks的预写式日志文件才会被删除。

我们希望将每个block保持合理的长度（典型部署情况下是两小时）来避免内存中积累太多的数据。当查询多个blocks时，我们需要将它们的结果合并成一个总体结果。这个合并的过程显然是有代价的，一个一周长的查询不应该去合并80+的部分结果。


为了达到上述两点（译注：block不能太长和不要合并太多的结果），我们引入*压缩（compaction）*。压缩描述了将一个或者多个blocks的数据写入到一个可能会更大的block的过程。这个过程可能会修改存在的数据，例如去除已被删除的数据，或者修改样本chunks的结构以提高查询性能。

```

t0             t1            t2             t3             t4             now
 ┌────────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │ 1          │  │ 2        │  │ 3         │  │ 4         │  │ 5 mutable │    before
 └────────────┘  └──────────┘  └───────────┘  └───────────┘  └───────────┘
 ┌─────────────────────────────────────────┐  ┌───────────┐  ┌───────────┐
 │ 1              compacted                │  │ 4         │  │ 5 mutable │    after (option A)
 └─────────────────────────────────────────┘  └───────────┘  └───────────┘
 ┌──────────────────────────┐  ┌──────────────────────────┐  ┌───────────┐
 │ 1       compacted        │  │ 3      compacted         │  │ 5 mutable │    after (option B)
 └──────────────────────────┘  └──────────────────────────┘  └───────────┘
```

这个例子中，我们有连续的blocks `[1, 2, 3, 4]`。1，2和3可以被压缩到一起，新的布局就是 `[1, 4]`。或者一对对地压缩，变成 `[1, 3]`。所有的时序数据仍然存在，但是包含在更少的blocks中了。这大大降低了查询时合并的代价，因为只需要合并更少的部分查询结果。

#### 保留

在V2存储中删除旧的数据是一个很慢的过程，也会给CPU，内存和磁盘带来压力。在基于block的设计中我们该怎么删除旧数据呢？非常简单，只要删除没有数据在配置的保留窗口里的block目录即可。在下面的例子中，block 1可以被安全地删除，而2要保留下来直到它完全落在边界后面。

```
                      |
 ┌────────────┐  ┌────┼─────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │ 1          │  │ 2  |     │  │ 3         │  │ 4         │  │ 5         │   . . .
 └────────────┘  └────┼─────┘  └───────────┘  └───────────┘  └───────────┘
                      |
                      |
             retention boundary
```

数据越旧，blocks会变得越大，因为我们一直在压缩blocks。需要设置一个上限，那样block就不会增大到跨度整个数据库而减少这个设计带来的原有的好处。

方便地是，这也限制了部分在保留窗口内，部分在窗口外的blocks带来的磁盘开销，例如上面例子中的block 2。当设置block的最大大小为整个保留窗口的10%时，将block 2保留下来的整个开销也被限制在10%以内。


总结一下，保留和删除过程从非常昂贵变得几乎免费了。

> 如果你已经看到这里，并且有一些数据库方面的背景，你也许会问：这里面有什么新东西吗？ —— 不见得；也许可以做得更好。
>
> 将数据在内存中批量化，在预写日志中跟踪数据，周期性地将数据刷到磁盘，这些模式现在非常普遍了。
>
> 我们看到的这些好处几乎普遍适用，不管数据的领域是什么。采用这种方法的知名开源的例子就有 LevelDB，Cassandra，InfluxDB和HBase。关键是要避免重新发明一个劣质的轮子，研究已经被证明的方法并以正确的方式应用它们。
>
> 没有地方可以添加你自己的魔法尘是不可能的情况。

### 索引

调研如何改良存储的最初动机是为解决 *series churn* 带来的问题。基于block的布局减少了应付一次查询需要考虑的序列的总数。所以假设索引查询的复杂度是*O(n^2)*，我们大幅度减少了*n*的大小，现在有了更好的*O(n^2)*复杂度 —— 额，等等，靠。

回忆下“算法101”，理论上这根本没啥用。如果过去事情很糟糕，现在还是一样糟糕。理论可以让人沮丧。


实际上，大多数查询已经能被更快地响应了。然而，那些跨越整个时间范围的查询还是很慢，即使只需要寻找几个序列。还在这些工作启动之前，我的初始想法就是对这个问题的解决方案：我们需要一个更好的[反向索引(inverted index)](https://en.wikipedia.org/wiki/Inverted_index)。

反向索引提供了一个基于数据项的内容子集来查询这些数据的方法。简单地说，可以查到所有带有标签 `app="nginx"` 的序列，而不用遍历查询所有序列并检查其是否包含这个标签。


每个序列被分配了一个唯一的ID，通过这个ID可以在常数时间内获取到索引，也就是O(1)的时间复杂度。在这种情况下，ID就是*正向索引(forward index)*。

> 例子：假设ID为10，29，和9的序列含有标签 `app="nginx"`，那么标签“nginx”的反向索引就是个简单的列表 `[10, 29, 9]`。这个列表可以用来快速检索到所有包含该标签的序列。即使还有200亿个序列，也不会影响检索的速度。

简而言之，假如*n*是序列的总数，*m*是对于给定查询的结果大小，那么使用索引来查询的复杂度是*O(m)*。 查询和要检索到的数据量（m），而不是和被搜索的数据（n）成比例，这是一个很好的性质，因为m一般要小得多。

为简洁起见，假设我们可以在常数时间内检索得到反向索引本身。

实际上，这几乎正是V2的反向索引所具有的类型，也是提供对数百万个序列进行高效查询的最低要求。敏锐的观察者会注意到，在最坏的情况下，一个标签存在于所有序列中，*m*同样是*O(n)*。这是预料之中的，而且完全没问题。如果查询所有数据，自然会花费更长的时间。然而一旦涉及到更复杂的查询，事情就会变得有问题。

#### 组合标签

和数百万个序列相关联的标签是很常见的。假定一个水平扩展的“foo”微服务，有着数百个实例，每个实例又有几千个序列。每个序列都有标签 `app="foo"`。当然，一般不会查询所有序列，而是用更多的标签来限制查询量，例如我想知道服务的实例接受了多少个请求，就会查询 `__name__="requests_total" AND app="foo"`。


为找到所有满足这两个标签选择器的序列，我们得到两者的反向索引列表，然后取交集。结果集一般会比每个输入的列表小几个数量级。由于每个输入的列表最坏情况下大小是O(n)，对两个列表做嵌套循环的暴力解法的运行时复杂度是O(n^2)。其他的集合操作也有着同样的代价，比如取并集(`app="foo" OR app="bar"`)。当把更多的标签选择器加入到查询中时，复杂度呈指数增长 —— O(n^3), O(n^4), O(n^5) ... O(n^k)。通过改变执行顺序，可以使用许多技巧来最小化有效的运行时。越复杂，就越需要了解数据的形状和标签之间的关系。这引入了很多复杂性，但并没有减少算法的最坏情况运行时。

这实质上就是V2存储中的方法，幸运的是，一个看似轻微的修改就足以获得显著的改进。如果我们假设反向索引中的ID是有序的，会发生什么呢?

假设我们初始查询的列表示例如下：

```
__name__="requests_total"   ->   [ 9999, 1000, 1001, 2000000, 2000001, 2000002, 2000003 ]
     app="foo"              ->   [ 1, 3, 10, 11, 12, 100, 311, 320, 1000, 1001, 10002 ]

             intersection   =>   [ 1000, 1001 ]
```

这个交集相当小。可以通过在每个列表的开头设置一个游标，并始终在较小的数字处前进来找到这个交集。当两个数字相等时，我们将数字加到结果中，并向前移动两个游标。总的来说，我们以zig-zag模式扫描这两个列表，由于总是在其中一个列表上移动，总成本就是O(2n) = O(n)。

对于两个以上列表的不同集合操作的过程是相似的。因此，*k* 个集合的操作，在最坏情况的查找运行时，也仅仅是修改了乘法因子(O(k*n))，而不是指数(O(n^k))。这是一个显著的进步。

我在这里描述的是规范搜索索引的简化版本，几乎所有[全文搜索引擎](https://en.wikipedia.org/wiki/Search_engine_indexing#Inverted_indices)都使用它。每个序列的描述符都被视为一个简短的“文档”，其中的每个标签（名称+固定值）都被视为其中一个“单词”。我们可以忽略搜索引擎索引中通常会遇到的许多额外数据，如单词位置和频率数据。

似乎在提高实际运行时的方法上存在着无尽的研究，这经常对输入数据做一些假设。意料之中的是，还有很多方法可以压缩反向索引，这些方法各有利弊。因为我们的“文档”很小，而“单词”在所有序列中很大程度上是重复的，所以压缩变得几乎无关紧要。例如，一个有大约440万个序列的真实数据集，其中大约有12个标签，每个标签只有不到5,000个唯一值。对于我们的初始存储版本，我们使用不带压缩的基本方法，只是添加了一些简单的调整，以跳过大范围的没有相交的ID列表。


虽然保持ID有序听起来很简单，但它并不总是一个需要保持的简单不变量。例如，V2存储将哈希值作为ID分配给新序列，因此我们无法高效地建立有序的反向索引。

另一项艰巨的任务是在数据被删除或更新时修改磁盘上的索引。通常，最简单的方法是重新计算和重写它们，但同时要保持数据库的可查询性和一致性（比较难）。V3存储是这样做的，每个block都有一个单独的不可变的索引，只能通过压缩时的重写进行修改。只有完全保存在内存中的可变block的索引需要更新。

## 基准测试

## 结论
