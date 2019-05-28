# Elasticsearch监控指标
## 一、监控Elasticsearch集群的重要性
Elasticsearch具有通用性，可扩展性和实用性的特点，集群的基础架构必须满足如上特性。合理的集群架构能支撑其数据存储及并发响应需求。相反，不合理的集群基础架构和错误配置可能导致集群性能下降、集群无法响应甚至集群崩溃。
适当地监视群集可以帮助您实时监控集群规模，并且可以有效地处理所有数据请求。
一般从五个不同的维度来看待集群，并从这些维度中提炼出监控的关键指标，并探讨通过观察这些指标可以避免哪些潜在问题。
* 集群健康和节点可用性
* 搜索性能
* 索引性能
* 主机级系统和网络指标
* 内存和垃圾收集

![](https://i.imgur.com/VpE0Su8.png)



## 二、集群健康和节点可用性
### 集群健康
集群、索引、分片、副本的定义不再赘述。分片数的多少对集群性能的影响至关重要。分片数量设置过多或过低都会引发一些问题。

分片数量过多，则批量写入/查询请求被分割为过多的子写入/查询，导致该索引的写入、查询拒绝率上升；
对于数据量较大的索引，当分片数量过小时，无法充分利用节点资源，造成机器资源利用率不高或不均衡，影响写入/查询的效率。

通过GET _cluster/health监视群集时，可以查询集群的状态、节点数和活动分片计数的信息。还可以查看重新定位分片，初始化分片和未分配分片的计数。
```
GET _cluster/health
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 127,
  "active_shards" : 127,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 120,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 51.417004048582996
}
```

### 集群运行的重要指标：
* Status：状态群集的状态。红色：部分主分片未分配。黄色：部分副本分片未分配。绿色：所有分片分配ok。
* Nodes：节点。包括群集中的节点总数，并包括成功和失败节点的计数。 
* Count of Active Shards：活动分片计数。集群中活动分片的数量。 
* Relocating Shards：重定位分片。由于节点丢失而移动的分片计数。
* Initializing Shards：初始化分片。由于添加索引而初始化的分片计数。
* Unassigned Shards。未分配的分片。尚未创建或分配副本的分片计数。

### 节点运行状况维度
每个节点都运行物理硬件上，需要访问系统内存，磁盘存储和CPU周期，以便管理其控制下的数据并响应对集群的请求。

Elasticsearch是一个严重依赖内存 以实现性能的系统，因此密切关注内存使用情况与每个节点的运行状况和性能相关。改进指标的相关配置更改也可能会对内存分配和使用产生负面影响，因此记住从整体上查看系统运行状况非常重要。

监视节点的CPU使用情况并查找峰值有助于识别节点中的低效进程或潜在问题。CPU性能与Java虚拟机（JVM）的垃圾收集过程密切相关。

磁盘高读写可能导致系统性能问题。由于访问磁盘在时间上是一个“昂贵”的过程，因此应尽可能减少磁盘I/O。

通过如下命令行可以实现节点级别度量指标，并反映运行它的实例或计算机的性能。

```
GET /_cat/nodes?v&h=id,disk.total,disk.used,disk.avail,disk.used_percent,ram.current,ram.percent,ram.max,cpu
id   disk.total disk.used disk.avail disk.used_percent ram.current ram.percent ram.max cpu
Hk9w    931.3gb   472.5gb    458.8gb             50.73       6.1gb          78   7.8gb  14
```
节点运行的重要指标：
* disk.total ：总磁盘容量。节点主机上的总磁盘容量。
* disk.used：总磁盘使用量。节点主机上的磁盘使用总量。
* avail disk：可用磁盘空间总量。
* disk.avail disk.used_percent：使用的磁盘百分比。已使用的磁盘百分比。
* ram：当前的RAM使用情况。当前内存使用量（测量单位）。
* percent ram：RAM百分比。正在使用的内存百分比。
* max : 最大RAM。 节点主机上的内存总量
* cpu：中央处理器。正在使用的CPU百分比。

实际业务场景中推荐使用：Elastic-HQ， cerebro监控。

## 三、搜索性能维度
我们可以通过测量系统处理请求的速率和每个请求的使用时间来衡量集群的有效性。

当集群收到请求时，可能需要跨多个节点访问多个分片中的数据。系统处理和返回请求的速率、当前正在进行的请求数以及请求的持续时间等核心指标是衡量集群健康重要因素。

搜索请求是Elasticsearch中的两个主要请求类型之一。（另一个是索引请求）。这些请求有时类似于传统数据库系统中的读写请求。Elasticsearch提供了与搜索过程的两个主要阶段（查询和获取）相对应的度量：
* 第一是查询阶段（query phase），集群将请求分发到索引中的每个分片（主分片或副本分片）。
* 第二个是获取阶段（fetch phrase），查询结果被收集，处理并返回给用户。

从开始到结束的搜索请求的路径
* （1）客户端向节点2发送搜索请求。
![](https://i.imgur.com/s60StsL.png)

* （2）2节点2（协调节点）将查询发送到索引中每个分片（主分片或副本）。
![](https://i.imgur.com/vgAgJYo.png)

* （3）每个接收到请求的分片本地执行查询（每个分片都是一个lucene实例）并将结果传递给节点2.节点2将其排序并编译成全局优先级队列。
![](https://i.imgur.com/hLNbmhn.png)

* （4）节点2发现需要获取哪些文档，并向相关的分片发送多个GET请求。
![](https://i.imgur.com/PiBHzcN.png)

* （5）每个分片加载文档并将其返回到节点2。
![](https://i.imgur.com/TVT71as.png)

* （6）节点2将搜索结果传递给客户端。
![](https://i.imgur.com/2UuwNbJ.png)

通过GET index_a/_stats查看对应目标索引状态。限于篇幅原因，返回没有给全。具体的自己实践一把吧。

```
  "search" : {
    "open_contexts" : 0,
    "query_total" : 10,
    "query_time_in_millis" : 0,
    "query_current" : 0,
    "fetch_total" : 1,
    "fetch_time_in_millis" : 0,
    "fetch_current" : 0,
    "scroll_total" : 5,
    "scroll_time_in_millis" : 15850,
    "scroll_current" : 0,
    "suggest_total" : 0,
    "suggest_time_in_millis" : 0,
    "suggest_current" : 0
  },
```

请求检索性能相关的重要指标如下：
* query_current：当前正在进行的查询数。集群当前正在处理的查询计数。
* fetch_current：当前正在进行的fetch次数。集群中正在进行的fetch计数。
* query_total：查询总数。集群处理的所有查询的聚合数。
* query_time_in_millis：查询总耗时。所有查询消耗的总时间（以毫秒为单位）。
* fetch_total：提取总数。集群处理的所有fetch的聚合数。
* fetch_time_in_millis：fetch所花费的总时间。所有fetch消耗的总时间（以毫秒为单位）。


| 度量描述 | 名称 | 公制型 |
| -------- | -------- | -------- |	
|查询总数|indices.search.query_total|吞吐量|
|查询总时间|indices.search.query_time_in_millis|性能|
|当前正在进行的查询数量|indices.search.query_current|吞吐量|
|提取总数|indices.search.fetch_total|吞吐量|
|花费在提取上的总时间|indices.search.fetch_time_in_millis|性能|
|当前正在进行的提取数|indices.search.fetch_current|吞吐量|

### 搜索性能指标的要点：
* Query load：监视当前正在进行的查询数量可以让您了解群集在任何特定时刻处理的请求数量，任何异常的尖峰或陡峭都可能指出潜在的问题。另外，可能还想监视搜索线程池队列的大小。
* Query latency：虽然Elasticsearch没有明确提供此度量标准，但是监视工具可以帮助您使用可用的度量来计算平均查询延迟，方法是以定期的时间间隔对总查询次数和总经过时间进行抽样。如果延迟超过阈值，请设置警报，如果触发，请查找潜在的资源瓶颈，或调查是否需要优化查询。
* Fetch latency：搜索过程的第二部分，即提取阶段通常比查询阶段花费的时间少得多。如果您注意到这一指标不断增加，这可能是因为缓慢的磁盘，文档的额外加工（比如，高亮显示搜索结果中的相关文本等）或请求太多结果。


## 四、索引性能维度
索引请求类似于传统数据库系统中的写入请求。如果您的Elasticsearch工作量很重，那么监控和分析elasticsearch更新索引的效率是非常重要的。在了解指标之前，让我们来探索Elasticsearch更新索引的过程。当新信息添加到索引中或现有信息被更新或删除时，索引中的每个分片将通过两个进程进行更新：refresh(更新到内存中)和flush（更新到硬盘上）。

文档的增、删、改操作，集群需要不断更新其索引，然后在所有节点上刷新它们。所有这些都由集群负责，作为用户，除了配置refresh interval之外，我们对此过程的控制有限。

增、删、改批处理操作，会形成新段（segment）并刷新到磁盘，并且由于每个段消耗资源，因此将较小的段合并为更大的段对于性能非常重要。同上类似，这由集群本身管理。

监视文档的索引速率（indexing rate）和合并时间（merge time）有助于在开始影响集群性能之前提前识别异常和相关问题。将这些指标与每个节点的运行状况并行考虑，这些指标为系统内的潜问题提供重要线索，为性能优化提供重要参考。

可以通过GET /_nodes/stats 获取索引性能指标，并可以在节点，索引或分片级别进行汇总。

```
  "merges" : {
          "current" : 0,
          "current_docs" : 0,
          "current_size_in_bytes" : 0,
          "total" : 245,
          "total_time_in_millis" : 58332,
          "total_docs" : 1351279,
          "total_size_in_bytes" : 640703378,
          "total_stopped_time_in_millis" : 0,
          "total_throttled_time_in_millis" : 0,
          "total_auto_throttle_in_bytes" : 2663383040
        },
        "refresh" : {
          "total" : 2955,
          "total_time_in_millis" : 244217,
          "listeners" : 0
        },
        "flush" : {
          "total" : 127,
          "periodic" : 0,
          "total_time_in_millis" : 13137
        },
```

索引性能维度相关重要指标：
* refresh.total：总刷新计数。刷新总数的计数。
* refresh.total_time_in_millis：刷新总时间。汇总所有花在刷新的时间（以毫秒为单位进行测量）。
* merges.current_docs：目前的合并。合并目前正在处理中。
* merges.total_docs：合并总数。合并总数的计数。
* merges.total_stopped_time_in_millis。合并花费的总时间。合并段的所有时间的聚合。

### 索引refresh
新索引的文档不能立即被搜索到。首先，它们被写入一个内存中的缓冲区，它们等待下一次索引刷新，默认情况下每秒刷新一次。刷新过程（使新索引的文档可搜索）：从内存缓冲区（in-memory buffer）中创建新的内存段（segment），然后清空缓冲区，如下所示。
![](https://i.imgur.com/vhE9jWV.png)

### 段（segment）的特殊细分
索引的分片（shard）由多个片段(segment)组成。segment是Lucene的核心数据结构，其本质上是一个用于存储索引（index）增量的集合。这些段是在每次刷新时创建的，随后随后在后台合并，以确保资源的有效使用（每个段实际上是以文件的形式存储在磁盘上，使用文件句柄，内存和CPU）。

分段可以看作是将词（terms）映射到包含这些术语的文档的小型倒排索引。每次搜索索引时，必须搜索每个分片的primary或replica版本，依次搜索该分片中的每个片段(segment)。

分段是不可变的，因此更新文档意味着：
* 在刷新过程中将信息写入新的段
* 将旧信息标记为已删除
* 当过时的段与其他段合并时，旧信息最终被删除。

### 索引flush
在将新建索引的文档添加到内存缓冲区的同时，它们也会被写入到分片的translog：一个持久化的，顺序写的，只能追加的事务日志。每隔30分钟，或者每当translog达到最大大小（默认为512MB）时，将触发flush。在flush期间，刷新内存缓冲区中的所有文档（存储在新的段中），然后将所有内存中的段都提交到磁盘，并且清除translog。

translog有助于防止节点发生故障时的数据丢失。它旨在帮助分片恢复在flush间隔之间可能已经丢失的数据。日志每5秒提交一次磁盘或每次成功的索引，删除，更新或批量请求（以先到者为准）也会触发提交。

Flush过程如下图所示：
![](https://i.imgur.com/51oRKNb.png)

Elasticsearch提供了许多指标，可用于评估索引性能并优化更新索引的方式。

|度量描述|名称|公制型|
| -------- | -------- | -------- |
|索引的文件总数|indices.indexing.index_total|吞吐量|
|索引文档总时间|indices.indexing.index_time_in_millis|性能|
|目前索引的文件数量|indices.indexing.index_current|吞吐量|
|索引刷新总数|indices.refresh.total|吞吐量|
|刷新指数的总时间|indices.refresh.total_time_in_millis|性能|
|索引刷新总数到磁盘|indices.flush.total|吞吐量|
|将索引刷新到磁盘上的总时间|	indices.flush.total_time_in_millis|性能|

### 索引要观看的性能指标
#### 索引延迟
索引延迟： Elasticsearch不会直接暴露此特定指标，但监控工具可以帮助您从可用index_total和index_time_in_millis指标计算平均索引延迟。如果您注意到延迟增加，您可能是一次尝试索引太多的文档了（Elasticsearch的官方文档建议从5到15兆字节的批量索引大小开始，并从那里缓慢增加）。

如果您计划索引大量文档，并且不需要立即可用于搜索的新信息，则可以通过减少刷新频率来优化搜索性能的索引性能，直到完成索引。该指数设置API，您可以暂时禁用的刷新间隔：
```
curl -XPUT <nameofhost>:9200/<name_of_index>/_settings -d '{
     "index" : {
     "refresh_interval" : "-1"
     } 
}'
```
  
完成索引后，您可以恢复为默认值“1s”。

#### Flush延迟
Flush延迟：由于在刷新成功完成之前，数据不会立即持久化，因此如果性能开始下降，则可能会跟踪flush延迟并采取措施。如果您看到该指标稳步增加，则意味着是磁盘较慢的问题; 此问题可能升级，最终导致您无法向索引添加新信息。您可以尝试在flush的settings中降低index.translog.flush_threshold_size。此设置确定translog到达多大时会触发flush。但是，如果你的Elasticsearch的写操作很频繁，你应该使用类似iostat或Datadog代理的工具，以保持紧密的对磁盘IO的指标的关注，必要时，考虑升级您的磁盘。

####  段（segment）合并
段合并相关的信息会告诉你目前在运行几个合并，合并涉及的文档数量，正在合并的段的总大小，以及在合并操作上消耗的总时间。在你的集群写入压力很大时，合并统计值非常重要。合并要消耗大量的磁盘I/O和CPU资源。有时候合并会拖累写入速率，如果拖累写入速率，Elasticsearch会自动限制索引请求到单个线程里。这个可以防止出现段爆炸问题，即数以百计的段在被合并之前就生成出来。
> 注意：文档更新和删除也会导致大量的合并数，因为它们会产生最终需要被合并的段碎片。

## 五、JVM健康维度
作为基于Java的应用程序，Elasticsearch在Java虚拟机（JVM）中运行。JVM在其“堆”分配中管理其内存，并通过garbage collection进行垃圾回收处理。

如果应用程序的需求超过堆的容量，则应用程序开始强制使用连接的存储介质上的交换空间。虽然这可以防止系统崩溃，但它可能会对集群的性能造成严重破坏。监视可用堆空间以确保系统具有足够的容量对于集群的健康至关重要。

JVM内存分配给不同的内存池。您需要密切注意这些池中的每个池，以确保它们得到充分利用并且没有被超限利用的风险。

垃圾收集器（GC）很像物理垃圾收集服务。我们希望让它定期运行，并确保系统不会让它过载。理想情况下，GC性能视图应类似均衡波浪线大小的常规执行。尖峰和异常可以成为更深层次问题的指标。

可以通过GET /_nodes/stats 命令检索JVM度量标准。

```
  "jvm" : {
        "timestamp" : 1557588707194,
        "uptime_in_millis" : 22970151,
        "mem" : {
          "heap_used_in_bytes" : 843509048,
          "heap_used_percent" : 40,
          "heap_committed_in_bytes" : 2077753344,
          "heap_max_in_bytes" : 2077753344,
          "non_heap_used_in_bytes" : 156752056,
          "non_heap_committed_in_bytes" : 167890944,
          "pools" : {
            "young" : {
              "used_in_bytes" : 415298464,
              "max_in_bytes" : 558432256,
              "peak_used_in_bytes" : 558432256,
              "peak_max_in_bytes" : 558432256
            },
            "survivor" : {
              "used_in_bytes" : 12178632,
              "max_in_bytes" : 69730304,
              "peak_used_in_bytes" : 69730304,
              "peak_max_in_bytes" : 69730304
            },
            "old" : {
              "used_in_bytes" : 416031952,
              "max_in_bytes" : 1449590784,
              "peak_used_in_bytes" : 416031952,
              "peak_max_in_bytes" : 1449590784
            }
          }
        },
        "threads" : {
          "count" : 116,
          "peak_count" : 119
        },
        "gc" : {
          "collectors" : {
            "young" : {
              "collection_count" : 260,
              "collection_time_in_millis" : 3463
            },
            "old" : {
              "collection_count" : 2,
              "collection_time_in_millis" : 125
            }
          }
        },
```

JVM运行的重要指标如下：
* mem：内存使用情况。堆和非堆进程和池的使用情况统计信息。
* threads：当前使用的线程和最大数量。
* gc：垃圾收集。算和垃圾收集所花费的总时间。

### 垃圾收集
Elasticsearch依靠垃圾收集过程来释放堆内存。因为垃圾收集也会消耗系统资源（为了释放资源！），您应该注意其频率和持续时间，以查看是否需要调整堆大小。将堆设置得太大可能导致垃圾收集时间长; 这些过度的停顿是危险的，因为它们可能导致您的群集错误地将节点注册为已经掉线状态。

|度量描述|名称|公制型|
| ----- | --- | --- |
|年轻代垃圾收集总数|jvm.gc.collectors.young.collection_count（jvm.gc.collectors.ParNew.collection_count在0.90.10之前）|其他|
|花费在年轻代垃圾收集上的总时间|jvm.gc.collectors.young.collection_time_in_millis（jvm.gc.collectors.ParNew.collection_time_in_millis在0.90.10之前）|其他|
|年老代垃圾收集总数|jvm.gc.collectors.old.collection_count（jvm.gc.collectors.ConcurrentMarkSweep.collection_count在0.90.10之前）|其他|
|花在年老代垃圾收集上的总时间|	jvm.gc.collectors.old.collection_time_in_millis（jvm.gc.collectors.ConcurrentMarkSweep.collection_time_in_millis在0.90.10之前）|其他|
|当前正在使用的JVM堆的百分比|jvm.mem.heap_used_percent|资源：利用|
|配置的JVM堆的数量|jvm.mem.heap_committed_in_bytes|资源：利用|

### 需要关注的JVM指标
![](https://i.imgur.com/Thy3BBz.png)

**正在使用的JVM堆**：Elasticsearch被设置为每当JVM堆使用率达到75％时，启动垃圾收集。如上所示，它被用于监视哪些节点有高堆使用量，并设置一个警报，以确定是否有任何节点始终使用超过85％的堆内存; 这表明垃圾收集率跟不上垃圾的生产率。要解决这个问题，您可以增加堆大小（只要它低于上述建议的准则），或者通过添加更多节点来扩展集群。（如果超过75%的使用率才做垃圾回收，在过大的堆内存时，每次垃圾回收的时间会很长；而过小的堆内存，则可能会造成频繁的垃圾回收，并且回收速度赶不上生产速度，因此得在堆内存的大小上作一个权衡）

**JVM堆使用与JVM堆大小的设置**：与已设置的内存（保证可用的数量）相比，了解当前使用的JVM堆的大小是有帮助的。正在使用的堆内存量通常会垃圾累积时上升和在垃圾收集时下降。如果模式随着时间的推移开始向上偏移，这意味着垃圾收集速度跟不上创建对象的速度，这可能导致垃圾收集时间慢，最终导致OutOfMemoryErrors。

#### 垃圾收集时间和频率
垃圾收集时间和频率：年轻代和年老代垃圾收集器都会经历“stop the world”的阶段，因为此时JVM会停止执行程序以收集无用的对象。在此期间，节点无法完成任何任务。由于主节点每30秒检查一次其他节点的状态，如果任何节点的垃圾收集时间超过30秒，则会导致主节点相信节点发生故障。

### 内存使用情况
如上所述，Elasticsearch非常会利用任何尚未分配给JVM堆的RAM。像Kafka一样，Elasticsearch被设计为依靠操作系统的文件系统缓存来快速可靠地提供请求。

许多变量决定了Elasticsearch是否能成功读取文件系统缓存。如果段文件最近被Elasticsearch写入磁盘，那么它已经在缓存中。但是，如果节点已被关闭并重新启动，则首次查询某个段时，该信息很可能必须从磁盘读取。这就是是为什么您需要确保群集保持稳定并且节点不会崩溃的重要原因之一。

一般来说，监视节点上的内存使用情况非常重要，同时给Elasticsearch尽可能多的RAM，这样就可以利用文件系统缓存的速度，而不会耗尽空间。

## 六、主机级网络和系统指标

|名称	|公制型|
| ---- |----|
|可用磁盘空间|资源：利用|
|I/O利用率|资源：利用|
|CPU使用率|资源：利用|
|网络字节发送/接收|资源：利用|
|打开文件描述符|资源：利用|

虽然Elasticsearch通过API提供许多特定于应用程序的指标，但您也应该从每个节点收集和监控几个主机级别的度量。

### 需要报警的系统指标
磁盘空间：如果您的Elasticsearch集群是重写入的，此度量特别重要。您不想耗尽磁盘空间，因为这样您将无法插入或更新任何内容，并且节点将失败。如果节点上不到20％可用，则可能需要使用“ curator”等工具来删除该节点上驻留的占用太多有价值磁盘空间的某些索引。

如果删除索引不是一个选项，另一个选择是添加更多节点，并让主节点自动重新分配新节点上的分片（尽管您应该注意到，这为繁忙的主节点创建了额外的工作）。另外，请记住，具有分析字段的文档（需要文本分析的字段，会执行标记化，分词，删除标点符号等操作）比具有未分析字段（精确值）的文档占用更多的磁盘空间。

### 需要监控的系统指标
**I/O利用率**：由于段的创建，查询和合并，Elasticsearch对磁盘进行了大量写入和读取。对于具有不断遇到重度I / O活动的节点的写入繁重的群集，Elasticsearch建议使用SSD来提升性能。

![](https://i.imgur.com/7XsuAqm.png)
**节点的CPU利用率**：可以在每个节点类型的热图（如上所示）中可视化CPU使用情况。例如，您可以创建三个不同的图表来表示集群中的每组节点（例如，数据节点，主节点，客户端节点），以查看是否有一种类型的节点与其他类型的节点相比较活动超载。如果看到CPU使用率增加，这通常是由于繁重的搜索或索引工作负载引起的。设置通知以确定节点的CPU使用率是否持续增加，如果需要，可以添加更多节点来重新分配负载。

**发送/接收的网络字节**：节点之间的通信是平衡集群的关键组件。您将需要监控网络，以确保其健康，并满足您对集群的需求（例如，分段在节点之间复制或重新平衡）。Elasticsearch提供有关集群通信的传输指标，但您也可以查看发送和接收的字节数，以查看网络接收的流量。

**打开文件描述符**：文件描述符用于节点到节点的通信，客户端连接和文件操作。如果此号码达到您系统的最大容量，那么新的连接和文件操作将不可能，直到旧的关闭。如果超过80％的可用文件描述符被使用，您可能需要增加系统的最大文件描述符数量。大多数Linux系统每个进程只允许1024个文件描述符。在生产中使用Elasticsearch时，您应该将操作系统文件描述符的数量重新设置得更大，如64,000。

### HTTP连接
|度量描述|名称|公制型|
| -----|---- | --- |
|当前打开的HTTP连接数|http.current_open|资源：利用|
|一段时间内打开的HTTP连接总数|http.total_opened|资源：利用|

任何语言都可以给ES发送请求，但Java将使用RESTful API通过HTTP与Elasticsearch进行通信。如果打开的HTTP连接总数不断增加，可能表示您的HTTP客户端没有正确建立持久连接。重新建立连接会在您的请求响应时间内增加额外的毫秒甚至几秒钟。确保您的客户端配置正确，以避免对性能造成负面影响，或使用已正确配置HTTP连接的官方Elasticsearch客户端。

## 七、资源饱和度和错误
Elasticsearch节点使用线程池来管理线程如何消耗内存和CPU。由于线程池设置是根据处理器数量自动配置的，所以调整它们通常没有意义。但是，最好关注队列的添加和拒绝，以了解您的节点是否无法跟上; 如果是这样，您可能需要添加更多节点来处理所有并发请求。Fielddata和过滤器高速缓存的使用是另一个要监控的领域，因为缓存的eviction(从缓存中移除，比如根据RLU)可能指向低效的查询或内存压力的迹象。

### 线程池入队和拒绝
每个节点维护许多类型的线程池; 您要监视的确切位置将取决于您对Elasticsearch的具体用途。一般来说，监控最重要的是搜索，索引，合并和bulk，它们与请求类型（搜索，索引，合并和批量操作）相对应。

每个线程池的队列的大小表示节点当前处于可用等待服务的请求数。队列允许节点跟踪并最终服务这些请求，而不是丢弃它们。一旦线程池中的任务达到最大队列大小，线程池将拒绝新的任务（根据线程池的类型而异）。

|度量描述|名称|公制型|
|-----|----|----|
|线程池中的排队线程数|thread_pool.bulk.queue、thread_pool.index.queue、thread_pool.search.queue、thread_pool.merge.queue|资源：饱和度|
|线程池的被拒绝线程数|thread_pool.bulk.rejected、thread_pool.index.rejected、thread_pool.search.rejected、thread_pool.merge.rejected|资源：错误|

### 资源指标
**线程池队列**：线程池不应设置过大，因为它们占用资源，并且如果节点关闭，还会增加丢失请求的风险。如果您看到排队和拒绝的线程数量稳步增加，您可能希望尝试减慢请求速率（如果可能），增加节点上的处理器数量或增加群集中的节点数量。如下面的截图所示，查询负载峰值与搜索线程池队列大小的峰值相关，因为节点尝试跟上查询请求的速率。
![](https://i.imgur.com/6RVZr9p.png)

**批量拒绝和批量入队**：批量操作是一次发送许多请求的更有效的方式。通常，如果要执行许多操作（创建索引或添加，更新或删除文档），则应尝试发送bulk请求，而不是许多单独的请求。

批量拒绝（bulk rejection）通常与在一个bulk请求中尝试索引太多文档有关。根据Elasticsearch的文档，批量拒绝不一定要担心。但是，您应该尝试实施线性或指数退避策略，以有效地处理批量拒绝。

### 缓存使用率指标
每个查询请求都会被发送到索引中的每个分片，然后再尝试去命中分片上的段。Elasticsearch以每个段为基础来缓存查询，以加快响应时间。另一方面，如果您的缓存过多地堆积在堆上，那么它们可能会减慢速度，而不是加快速度！

在Elasticsearch中，文档中的每个字段可以以两种形式存储：作为精确值（keyword）或全文(text)。对于keyword，如时间戳或年份，会按照它的值原原本本的存储。如果一个字段存储为全文(text)，这意味着它被分词 - 基本上它被分解成令牌，并且根据分析器的类型，可以删除标题符和停止词如“是”或“该”。分析器将该字段转换为归一化格式，使其能够匹配更广泛的查询。

例如，假设你有一个索引包含一个类型location; 该类型的每个文档都包含一个字段city，它被存储为一个分析的字符串。你索引两个文件：一个在city中存储“圣 路易斯“，另一个为”圣 保罗”。每个字符串将被转化为小写版本并转换成标记，而不用标点符号。这些术语存储在反向索引中，看起来像这样：

|术语	|文档1|	文档2|
|---|---|---|
|ST|X|X|
|路易斯|X| |
|保罗	| |X|
分析的好处是您可以搜索“st”，结果将显示两个文档都包含该术语。如果您将该city字段存储为一个keyword，那么您将不得不搜索确切的术语“圣 路易斯“或”圣 保罗“，以便看到结果文件。

Elasticsearch使用两种主要类型的缓存来更快地提供搜索请求：fielddata和filter。

#### Fielddata缓存
在field上排序或聚合时使用fielddata缓存，这个过程基本上必须把倒排索引再倒置过来，以文档顺序为每个field创建每个字段值的数组。例如，如果我们想在上述示例中找到任意包含词（term）“st”的文档中的唯一术语列表，我们将：
* 扫描倒排索引以查看哪些文档包含该术语（在本例中为Doc1和Doc2）
* 对于在步骤1中找到的每个文档，通过索引中的每个术语从该文档中收集令牌，创建如下所示的结构：


|文件	|Field（city）|
|---|---|
|文档1|圣, 路易斯|
|文档2|圣, 保罗|

现在，倒排索引已经被“反向”，从每个文档（st，路易斯和保罗）中编译出独特的令牌。编译这样的fielddata可能会消耗大量堆内存，尤其是大量的文档和术语。所有字段值都将加载到内存中。

对于1.3之前的版本，fielddata缓存大小是无限制的。从1.3版开始，Elasticsearch添加了一个fielddata断路器，如果查询尝试加载将需要超过60％的堆的fielddata，则会触发。

#### filter cache
filter cache也使用JVM堆。在2.0之前的版本中，Elasticsearch自动缓存过滤的查询，最大值为堆的10％，并且将最近最少使用的数据逐出。从版本2.0开始，Elasticsearch会根据频率和段大小自动开始优化其过滤器缓存（缓存仅发生在索引中少于10,000个文档或小于总文档3％的段）。因此，过滤器缓存指标仅适用于使用2.0之前版本的Elasticsearch用户。

例如，过滤器查询可以仅返回year字段中的值在2000-2005范围内的文档。在首次执行过滤器查询过程中，Elasticsearch将创建一个文档与过滤器匹配的位组（如果文档匹配则为1，否则为0）。使用相同过滤器后续执行查询将重用此信息。无论何时添加或更新新文档，也会更新位组。如果您在2.0之前使用Elasticsearch版本，那么您应该关注过滤器缓存以及驱逐指标（更多关于以下内容）。

|度量描述|名称|公制型|
|---|---|---|
|fielddata缓存的大小（字节）|indices.fielddata.memory_size_in_bytes|	资源：利用|
|来自fielddata缓存的驱逐次数|indices.fielddata.evictions|资源：饱和度|
|过滤器高速缓存的大小（字节）（仅版本2.x）|indices.filter_cache.memory_size_in_bytes|资源：利用|
|来自过滤器缓存的驱逐次数（仅版本2.x）|indices.filter_cache.evictions|资源：饱和度|

#### 要监视的缓存指标
![](https://i.imgur.com/WYKkFSG.png)

**Fielddata缓存驱逐**：理想情况下，您希望限制fielddate cache的驱逐（eviction）的数量，因为它们是I / O密集型的。如果您看到很多驱逐，而且目前无法增加内存，Elasticsearch建议临时fielddate cache到堆的20%; 你可以在你的config/elasticsearch.yml文件中这样做。当fielddata达到堆的20％时，它将驱逐最近最少使用的fielddata，然后允许您将新的fielddata加载到缓存中。

Elasticsearch还建议尽可能使用doc value，因为它们与fielddata的用途相同。但是，由于它们存储在磁盘上，它们不依赖于JVM堆。虽然doc值不能用于analyzed string fields，但是当在其他类型的字段上进行聚合或排序时，它们会保存fielddata的用法。在版本2.0和更高版本中，doc values在文档索引期间自动构建，这减少了许多对fielddata/堆的使用。但是，如果您使用1.0和2.0之间的版本，还可以从此功能中受益 - 只需记住在索引中创建新字段时启用它们。

**Filter cache evictions**：如前所述，filter cache驱逐指标仅在2.0版之前使用Elasticsearch版本时可用。每个段维护自己的单独过滤器高速缓存。由于驱逐是在大的segment比在小segment上成本更高的操作，因此没有明确的方法来评估每次驱逐的严重程度。但是，如果您看到越来越频繁的eviction，这可能表明您没有使用过滤器来获得最大的利益 - 您可能正在不停的创建新的过滤器，并频繁地排除旧的过滤器，从而打破了使用缓存的目的。您可能需要考虑调整您的查询（例如，使用bool查询而不是和/或/不过滤器）。

|度量描述|	名称|	公制型|
|----| ----|----|
|待处理任务数|pending_task_total|资源：饱和度|
|挂起的紧急任务数量|pending_tasks_priority_urgent|资源：饱和度|
|待处理高优先级任务数|pending_tasks_priority_high|资源：饱和度|

待处理的任务只能由主节点处理。这些任务包括创建索引并将分片分配给节点。待处理的任务按优先顺序处理 - urgent先处理，然后是high。当操作的次数发生得比主节点处理更快时，它们开始累积。如果不断增加，您需要关注这一指标。待处理任务的数量是您的群集运行平稳的良好指示。如果您的主节点很忙，并且未完成的任务数量不会减少，则可能会导致不稳定的集群。

### 不成功的GET请求
|度量描述|名称|公制型|
|------|----|-----|
|丢失的文件的GET请求总数|indices.get.missing_total|工作：错误|
|花费在文档丢失的GET请求上的总时间|indices.get.missing_time_in_millis|工作：错误|

GET请求比正常的搜索请求更简单 - 它根据其ID来检索文档。get-by-ID请求不成功意味着找不到文档ID。你通常不应该对这种类型的请求有任何担心，但是当发生GET请求失败时，还是请注意一下。

