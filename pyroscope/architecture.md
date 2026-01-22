# Pyroscope架构

## 关于架构

* Pyroscope有一个基于微服务的架构
  * 系统有多个水平可伸缩的微服务,能独立或者并行运行都可以
  * 这些微服务都称为组件(component)
* Pyroscope设计上将所有组件的代码编译成同一个二进制文件
  * `-target` 参数控制这个单一二进制文件作为什么组件运行
  * 对于那些希望以简单方式入门的用户，Pyroscope 也支持单体模式，所有组件在同一个进程中同时运行。

### Pyroscope组件

* 大部分组件都是无状态的, 在进程重启之间不需要任何持久化数据.
* 部分组件是有状态的,依赖非易失性存储在进程重启之间防止数据丢失
* 有关每个组件的详细信息，请参阅“组件”中的相应页面。

#### 写路径

<p align="center">
  <a href="./images/write-path.svg">
    <img src="./images/write-path.svg" width="105" alt="写入路径示意图" />
  </a>
</p>
<p align="center"><sub>写入路径示意图</sub></p>
* ingesters从distributor (distributors)接收传入的profiles。
  * 每个推送请求都属于一个租户
  * ingester会将接收到的profiles追加到存储在本地磁盘上的每租户的 Pyroscope 数据库中。
* 每个ingester会在收到该租户的第一个profiles后立即创建该租户的 Pyroscope 数据库。
* 在内存中的profiles会定期刷新到磁盘并创建新的block。

#### 系列分片和复制

* 默认情况下，每个profile系列的数据会被复制到三个ingesters，每个 ingesters 会将自己的block写入长期存储。
* compactor 会将来自多个ingesters的block合并成一个block，并删除重复的sample。
* blockcompaction可以显著降低存储占用率。

#### 读路径

<p align="center">
  <a href="./images/read-path.svg">
    <img src="./images/read-path.svg" width="420" alt="读路径示意图" />
  </a>
</p>
<p align="center"><sub>读路径示意图</sub></p>

* 进入 Pyroscope 的查询会到达query-frontend组件(query-frontend)，该组件负责加速查询并将其分发给查询调度器(query-scheduler)。
* 查询调度器(query-scheduler)维护一个查询队列，并确保每个租户的查询都能得到公平执行。
* 查询器(queriers)充当工作进程，从查询调度器的队列中提取查询。
  * 查询器(queriers)连接到ingesters，以获取执行查询所需的所有数据。
* 根据所选的时间窗口，querier会调用ingester来获取近期数据，并调用存储网关(store-gateways)来获取长期存储中的数据。

#### 长期存储

* Pyroscope 存储格式的详细描述请参见块格式页面。
* Pyroscope 存储格式将每个租户的profiles存储到其自身的磁盘块中。
  * 每个磁盘块目录包含一个索引文件、一个元数据文件以及 Parquet 表。
* Pyroscope 需要以下任一对象存储来存放块文件：
  * Amazon S3
  * Google Cloud Storage
  * Microsoft Azure Storage
  * OpenStack Swift
  * 本地文件系统（仅限单节点）

## 部署模式

* Pyroscope 官方支持两种部署模式:
  * 单体模式
    * 在此模式下，所有组件都在单个进程中运行
    * 适用于只需要一个pyroscope实例的情况，因为多个实例之间不会共享信息。
  * 微服务模式
    * 在这种模式下，随着实例数量的增加，它们将共享同一个后端进行存储和查询。
* 部署模式由 -target 参数决定，可以通过 CLI 标志或 YAML 配置设置该参数。

### 单体模式

![单体模式](./images/monolithic-mode.svg)

* 单体模式会在单个进程中运行所有必需组件，这是默认的运行模式
  * 可以通过指定 `-target=all` 来设置
* 单体模式是部署 Pyroscope 最简单的方法
  * 如果您想要快速入门或在开发环境中使用 Pyroscope，则非常有用。
* 要查看当 `-target` 设置为 `all` 时运行的组件列表，请使用 `-modules` 标志运行 Pyroscope：
```bash
./pyroscope -modules
```

### 微服务模式

![微服务模式](./images/microservices-mode.svg)

* 在微服务模式下，组件部署在独立的进程中。
* 扩展是按组件(per component)进行的
  * 这使得扩容更加灵活
  * 故障域控制也更加细粒度
* 微服务模式是生产环境部署的首选方法，但也最为复杂。
* 在微服务模式下，每个 Pyroscope 进程都是通过 `-target` 参数设置为特定的 Pyroscope 组件（例如，`-target=ingester` 或 `-target=distributor`） 来调用的
* 要获得一个可用的 Pyroscope 实例，您必须部署所有必需的组件。


## 组件

Pyroscope 包含一系列相互协作以形成集群的组件。

* Distributor
* Ingester
* Store-gateway
* Compactor
* Querier
* Query-frontend
* Query-scheduler

### distributor

* distributor 是一个无状态组件，它从agent接收profile。
* distributor 将数据分成多个批次，并行发送到多个ingester，并将数据series分片(shards)到各个 ingester ，并按照配置的复制因子复制每个序列。
  * 默认情况下，配置的复制因子为 3。

#### 校验

* distributor 在将数据写入 ingester 之前​​，会对接收到的数据进行清理和验证。
  * 由于单个请求可能包含有效和无效的 profiles、samples、metadata和exemplars，distributor 仅将有效数据传递给数据ingester。
  * distributor 不会在其发送给数据ingester的请求中包含无效数据。
  * 如果请求包含无效数据，distributor 将返回 400 HTTP 状态码，并在响应正文中显示详细信息。
  * agent通常会记录第一个无效数据的详细信息。
* distributor 的数据清理包括以下转换
  * 确保 profile 已设置时间戳，否则将默认为 distributor 接收到profile的时间。
  * distributor 将删除值为 0 的 samples ，并将具有相同堆栈跟踪的samples相加。

#### 复制(Replication)

* distributor 将传入的序列(series)分片(shards)并复制(replicates)到各个ingester
  * 您可以通过 `-distributor.replication-factor` 标志配置每个序列写入的ingester 副本数量，默认值为 1。
  * distributor 使用一致性哈希算法，并结合可配置的复制因子，来确定哪些ingester 接收给定的序列。
* Sharding和 replication 使用ingester 的哈希环
  * 对于每个传入的序列，distributor 使用 profile名称、标签和TenantID 计算哈希值。
  * 计算出的哈希值称为令牌。
  * distributor 在哈希环中查找令牌，以确定将序列写入哪些ingester。

##### 仲裁一致性

* 由于distributor 共享对同一哈希环的访问权限，因此写入请求可以发送到任何distributor
  * 您还可以在其前面设置无状态负载均衡器。
* 为了确保查询结果的一致性，Pyroscope 在读取和写入操作中使用 Dynamo 风格的仲裁一致性(quorum consistency)。
  * distributor 等待 (n/2 + 1) 个ingester 成功响应, 其中 n 是配置的复制因子, 然后才会向agent推送请求发送成功响应。

#### 在多个 distributors 之间的负载均衡

* 我们建议在distributor实例之间随机分配写入请求的负载
  * 如果您在 Kubernetes 集群中运行 Pyroscope，则可以将 Kubernetes Service 定义为distributor的入口。
* 注意： 存在HTTP级别的请求分布不均的可能性
  * Kubernetes Service 负责在 Kubernetes 端点之间平衡 TCP 连接，但不会平衡单个 TCP 连接内的 HTTP 请求。
  * 如果您启用了 HTTP 持久连接（HTTP keep-alive），由于 Agent 使用了 HTTP keep-alive，它会为每个推送 HTTP 请求重用同一个 TCP 连接。
  * 这可能会导致distributor接收到的推送 HTTP 请求分布不均。

### ingester

* ingester 是一个有状态组件
  * 在写入路径上, 它首先将传入的profile写到磁盘存储
  * 在读取路径上, 返回series samples以供查询使用
* 来自distributor 的传入profile 不会立即写入长期存储，而是保存在ingester的内存中或卸载(offloaded)到ingester的磁盘上。
  * 最终，所有profile 都会写入磁盘并定期上传到长期存储。
  * 因此，queriers在读取路径上执行查询时，可能需要同时从ingester和长期存储中获取samples。
* 任何调用ingesters的 Pyroscope 组件首先会查找哈希环(hash ring)中注册的ingester，以确定哪些ingester可用。
* 每个 ingester 可能处于以下状态之一：
  * PENDING
    * ingester刚刚启动。
    * 在此状态下，ingester不接收写入或读取请求。
  * JOINING
    * ingester启动并加入环。
    * 在此状态下，ingester不接收写入或读取请求。
    * ingester从磁盘加载令牌（如果配置了 `-ingester.ring.tokens-file-path`），或者生成一组新的随机令牌。
    * 最后，ingester（可选）会观察环中是否存在令牌冲突，冲突解决后，将进入ACTIVE状态。
  * ACTIVE
    * ingester正在运行。
    * 在此状态下，ingester可以接收写入和读取请求。
  * LEAVING
    * ingester正在关闭并离开环。
    * 在此状态下，ingester不接收写入请求，但仍可以接收读取请求。
  * UNHEALTHY
    * ingester未能向哈希环发送心跳信号。
    * 在此状态下，distributor 会绕过ingester，这意味着ingester不会接收写入或读取请求。

#### ingester写入去放大

* ingester会将最近接收到的sample存储在内存中，以便对write做去放大(de-amplification)。
  * 如果ingester立即将接收到的sample写入长期存储，由于长期存储压力过大，系统将难以扩容。
  * 因此，ingester会对内存中的samples进行batch和compaction，并定期将其上传到长期存储。
* 写入去放大是 Pyroscope 总体拥有成本 (TCO: total cost of ownership) 低的主要原因。

#### ingester故障和数据丢失

* 如果ingester进程崩溃或突然退出，所有尚未上传到长期存储的内存中的profile都可能丢失。

* 以下方法可以缓解这种故障模式：Replication

##### Replication

* 默认情况下，每个profile series 都会复制到三个ingester
  * 如果至少有两个ingester接收到数据（复制因子为 3），则写入 Pyroscope 集群的操作才算成功。
  * 如果 Pyroscope 集群丢失一个ingester，则丢失ingester的头块中保存的内存profile至少可以在另一个ingester中找到。
* 如果只有一个ingester发生故障，则不会丢失任何profile。
* 如果多个ingester发生故障，并且故障影响到所有保存特定profile series副本的ingester，则profile可能会丢失。

### Store-gateway

* Pyroscope 中的存储网关负责在长期存储桶中查找profiling data
  * 单1个存储网关负责长期存储中的一部分block(子集)，并由querier调用。

### compactor

* compactor通过合并block来提高查询性能并减少长期存储占用。
* compactor负责以下功能：
  * 将给定租户的多个blockcompaction成一个经过优化的更大block。
    * 这可以去除重复数据并减小索引大小，从而降低存储成本。
    * 查询更少的block速度更快，因此也能提高查询速度。
  * 保持每个租户的bucket index更新
    * 查询器(queriers)和存储网关(store-gateways)使用bucket index(bucket index)来发现存储中的新block和已删除block。
* compactor是无状态的。

#### compaction是怎么工作的

* Compaction以租户为单位进行。
* compactor以可配置的固定时间间隔运行。

* 垂直compaction
  * 垂直compaction会将同一租户在同一时间段内（默认为 1 小时）由ingester上传的所有block合并成一个单独的block
  * 它还会对由于数据复制而最初写入 N 个block的样本进行去重
  * 垂直compaction会将单个时间段内的block个数从等于ingester数量减少到每个租户一个block
* 水平compaction
  * 水平compaction会在垂直compaction之后触发
  * 它会将多个具有相邻时间段的block compaction成一个更大的block
  * 水平compaction后，相关block的总大小不会改变
  * 水平compaction可能会显著减小存储网关(store-gateways)在内存中保存的索引(index)和索引头(index-header)的大小。

![compaction](./images/compactor-horizontal-and-vertical-compaction.png)

#### 扩容

* 对于拥有大量租户的集群，可以对compaction进行优化。
* 配置可指定compactor在按租户进行compaction时的垂直和水平扩展方式。

##### 垂直扩容

* `-compactor.compaction-concurrency` 用于配置单个compactor实例中同时运行的最大compaction任务数
* 每个compaction任务占用一个 CPU 核心。

##### 水平扩容

* 默认情况下，租户blocks可以由任何 Grafana Pyroscope compactor进行compaction
  * 当您通过将 `-compactor.compactor-tenant-shard-size`（或其相应的 YAML 配置选项）设置为大于 0 且小于可用compactor数量的值来启用compactor混洗分片(shuffle sharding)时，只有指定数量的compactor才能对给定租户的blocks进行compaction。

#### compaction算法

* Pyroscope 使用了一种名为“拆分合并(split-and-merge)”的复杂compaction算法。
* 拆分合并(split-and-merge)算法的设计旨在克服时间序列数据库 (TSDB) 索引的限制，并避免在任何compaction阶段，对于非常大的租户，compaction块无限增长的情况。
* 这种compaction策略是一个两阶段过程：拆分和合并。默认配置禁用拆分阶段。
  * 在拆分阶段，例如第一级compaction（2 小时），compactor会将所有source blocks分成 N 个组（-compactor.split-groups）
    * 对于每个组，compactor会compaction块，但不会生成单个结果块，而是输出 M 个块（-compactor.split-and-merge-shards），称为拆分块(split blocks)。
    * 每个拆分块(split block)仅包含 M 个分片(shards)中给定分片的序列的子集。
    * 在拆分阶段结束时，compactor会生成 N * M 个blocks，并在每个block的 meta.json 文件中包含指向其对应分片(shard)的引用。
  * compactor会合并每个分片(shard)的拆分数据块(split blocks)
    * 这将compaction给定分片(shard)的所有 N 个拆分数据块
    * 合并操作会将数据块(block)的数量从 N * M 减少到 M
    * 在给定的compaction时间范围内，每个分片(shard)都会有一个已compaction的block。

![compactor-split-and-merge](./images/compactor-split-and-merge.png)

合并操作随后会根据其他配置的compaction时间范围（例如 1 小时和 4 小时）运行。它会compaction属于同一分片的块。

此策略适用于拥有大量租户的集群。分片数 M 可通过 `-compactor.split-and-merge-shards` 为每个租户单独配置，并可根据每个租户的序列数进行调整。租户的序列越多，可配置的分片数就越多。这样做可以提高compaction并行化程度，并控制每个分片compaction块的大小。

拆分组数 N 也可通过 `-compactor.split-groups` 为每个租户单独调整。增加此值会在拆分阶段生成更多块数更少的compaction作业。这使得多个compactor可以同时处理这些作业，从而更快地完成拆分阶段。但是，增加这个值也会在分裂阶段产生更多的中间块，这些中间块只能在合并阶段减少。

如果在compaction过程中 `-compactor.split-and-merge-shards` 的配置发生更改，则该更改只会影响尚未拆分的数据块的compaction。已拆分的数据块在合并时将使用原始配置。原始配置存储在每个拆分数据块的 meta.json 文件中。

拆分和合并操作可以水平扩展。不冲突且不重叠的作业将并行执行。

#### compactor分片

compactor会对来自单个租户或多个租户的compaction作业进行分片。单个租户的compaction作业可以拆分并由多个compactor实例处理。

当compactor池增大或缩小时，租户和作业会自动重新分片到可用的compactor实例上，无需任何人工干预。

compactor分片使用哈希环。启动时，compactor会生成随机令牌并将其自身注册到compactor哈希环中。运行期间，它会按照 `-compactor.compaction-interval` 定义的时间间隔定期扫描存储桶，以发现存储中的租户列表，并compaction每个租户的、哈希值与分配给该实例的令牌范围相匹配的数据块。

要配置compactor的哈希环，请参阅配置成员列表。

#### 启动时等待稳定的哈希环

集群冷启动或同时增加两个或多个compactor实例可能会导致每个新compactor实例的启动时间略有不同。然后，​​每个compactor基于不同的哈希环状态运行其首次compaction。这并非错误情况，但效率可能较低，因为多个compactor实例可能几乎同时开始compaction同一租户。

为了缓解此问题，可以将compactor配置为在启动时等待稳定的哈希环。如果在至少 `-compactor.ring.wait-stability-min-duration` 的时间内没有向哈希环添加或从中移除任何实例，则认为哈希环稳定。compactor等待的最大时间由标志 `-compactor.ring.wait-stability-max-duration`（或相应的 YAML 配置选项）控制。一旦compactor完成等待（无论是由于哈希环稳定还是由于达到最大等待时间），它将正常启动。

-compactor.ring.wait-stability-min-duration 的默认值为零时，将禁用等待环稳定性。

#### compaction job顺序

compaction任务顺序

compactor允许通过 `-compactor.compaction-jobs-order` 标志（或其对应的 YAML 配置选项）配置compaction任务顺序。配置的顺序定义了哪些compaction任务应该首先执行。支持以下 `-compactor.compaction-jobs-order` 值：

最小范围最旧块优先（默认值）

此顺序优先处理范围最小、最旧的块。

例如，如果compaction范围为 1 小时、4 小时和 8 小时，compactor将首先compaction 1 小时范围的数据块，并优先处理其中最旧的块。compaction完 1 小时范围内的所有块后，它将处理 2 小时范围的数据块，最后处理 8 小时范围的数据块。

所有拆分任务都会被移到工作队列的前端，因为在给定的时间范围内完成所有拆分任务可以解除合并任务的阻塞。

最新块优先

此排序方式优先处理最新的时间范围，无论其compaction级别如何。

例如，compaction范围为 1 小时、4 小时和 8 小时，compactor会首先compaction最新的块（直至 8 小时范围），然后再compaction更早的块。此策略优先处理最新的块，因为它们的查询频率最高。

#### 块删除

成功完成compaction后，原始数据块将从存储中删除。数据块删除并非立即执行，而是分两步进行：

首先，将原始数据块标记为待删除；这称为软删除。

当数据块被标记为待删除的时间超过可配置的 `-compactor.deletion-delay` 值后，该数据块将从存储中删除；这称为硬删除。

compactor负责标记数据块和执行硬删除操作。软删除基于存储在存储桶中数据块位置的一个名为 `delete-mark.json` 的小型文件。

软删除机制允许查询器和存储网关在原始数据块被删除之前有时间发现新的compaction数据块。如果立即硬删除这些原始数据块，则某些涉及compaction数据块的查询可能会暂时失败或返回部分结果。

#### compaction磁盘利用率

compactor需要将数据块从存储桶下载到本地磁盘，并且需要将compaction后的数据块存储到本地磁盘，然后再上传到存储桶。最大的租户可能需要大量的磁盘空间。

假设 `max_compaction_range_blocks_size` 是最大租户在最长 `-compactor.block-ranges` 周期内的总数据块大小，则估算所需最小磁盘空间的表达式为：

```text
compactor.compaction-concurrency * max_compaction_range_blocks_size * 2
```

### querier

* querier是一个无状态组件
* 它通过获取读取路径上的profiles series和labels来对查询表达式进行计算
* querier使用ingesters 收集最近写入的数据，并使用store-gateways进行长期存储。

#### 连接到 ingesters

* 您必须使用与配置ingesters相同的 `-ingester.ring.*` 标志（或其各自的 YAML 配置参数）来配置querier，以便querier可以访问ingesters哈希环并发现ingesters的地址。

### query-frontend

* query-frontend是一个无状态组件，它提供与 querier相同的 API
  * 可用于加速读取路径，并通过 query-scheduler 确保租户之间的公平调度。
* 在这种情况下，queriers充当工作进程，从队列中拉取任务，执行任务，并将结果返回给query-frontend进行聚合。
* 出于高可用性的考虑，我们建议您至少运行两个query-frontend副本。
* 由于使用query-frontend时必须同时运行query-scheduler，因此您必须至少运行一个query-scheduler副本。
* 以下步骤描述了查询如何在query-frontend中流转。
  * query-frontend接收查询
  * query-frontend通过与query-scheduler通信，将查询放入队列中，等待querier领取。
  * querier从队列中领取查询并执行它。
  * querier将结果返回给query-frontend，然后query-frontend汇总结果并将其转发给客户端。

### query-scheduler

* 查询调度器是一个无状态组件，它维护一个待执行查询队列，并将工作负载分配给可用的查询器。
* 使用query-frontend时，查询调度器是必需组件。

![查询](./images/query-scheduler-architecture.png)

* 以下流程描述了一次查询在 Pyroscope 集群中的流转过程：
  * query-frontend(query-frontend)接收查询，然后将其拆分和分片，或从缓存中提供服务。
  * query-frontend(query-frontend)将查询放入查询调度器的队列中。
  * 查询调度器(query-scheduler)将查询存储在内存队列中，等待查询器(querier)领取。
  * 查询器(querier)领取查询并执行。
  * 查询器(querier)将结果发送回query-frontend(query-frontend)，query-frontend(query-frontend)再将结果转发给客户端。

#### 使用query-scheduler的优势

* 查询调度器支持query-frontend的扩容。

#### 配置

* 要使用查询调度器，query-frontend和查询器需要发现查询调度器实例的地址。
* 查询调度器使用基于环的服务发现机制来发布自身，该机制通过成员列表配置进行配置。

#### 运维注意事项

* 为了实现高可用性，请运行两个查询调度器副本。

## bucket index

* bucket index是一个租户级文件
  * 其中包含存储中的block(blocks)列表和block删除标记
  * bucket index(bucket index)存储在后端对象存储中，由compactor定期更新，并由存储网关(store-gateways)用于发现存储中的block。

### 优势

* 存储网关(store-gateways)必须拥有几乎最新的存储桶(storage bucket)的视图，以便在查询时找到要查找的正确block并加载block。
  * Ingesters 会定期向存储桶添加新block，同时将数据卸载到长期存储中。
  * compactor随后会compaction这些block，并将原始block标记为待删除。
  * 实际删除操作会在与参数 `-compactor.deletion-delay` 关联的延迟值之后执行。
  * 尝试获取已删除的block会导致查询失败。
  * 因此，在此上下文中，“几乎最新”的视图是指其过时时间小于 `-compactor.deletion-delay` 值的视图。
* 因此，存储网关(store-gateways)需要定期扫描存储桶，以查找由ingester或compactor上传的新block，以及由compactor删除（或标记为删除）的block。
* 启用bucket index(bucket index)后，存储网关(store-gateways)会定期查找租户级bucket index(bucket index)，而不是通过“列出对象(list objects)”操作扫描存储桶(bucket)。
* 这带来了以下优势：
  * 减少存储网关对对象存储的 API 调用次数
  * 存储网关不再执行“列出对象”存储 API 调用

### index的结构

* bucket-index.json.gz 文件包含：
  * blocks
    * 租户的完整block列表，包括标记为删除的block。
    * 部分block不包含在索引中。
  * block_deletion_marks
    * block删除标记列表
  * updated_at
    * 一个 Unix 时间戳，精确到秒，显示索引上次更新并写入存储的时间。

### bucket index是如何被更新的

* compactor会定期扫描bucket，并将更新后的bucket index上传到存储中。
  * 您可以通过 `-compactor.cleanup-interval` 配置bucket index的更新频率。
* bucket index的使用虽然是可选的, 但即使设置了 `-blocks-storage.bucket-store.bucket-index.enabled=false`，compactor仍然会构建和更新该索引。
  * 此行为确保每个租户的bucket index都存在，并且如果 Grafana Pyroscope 集群操作员在运行中的集群中启用bucket index，则可以保证查询结果的一致性。
  * 保持bucket index更新所带来的开销并不显著。

### 如何被store-gateway使用

* store-gateway在启动时以及定期给每个租户获取属于其shard的bucket index，并将其用作存储中块和删除标记的真实来源(source of truth)。
  * 这消除了定期扫描存储桶以发现属于其分片的块的需要。

## Block format

* 本文档描述了 Pyroscope 如何将数据存储在其block中。
  * 每个block(block)属于一个租户，并由唯一的 ULID 标识。
    * [ULID](https://github.com/ulid/spec)
  * block(block)内包含多个文件：
    * 元数据文件 meta.json
      * 其中包含有关block内容的信息
      * 例如profiling数据的时间范围
    * TSDB 索引 index.tsdb
      * 将外部标签映射到存储在 profiles 表中的profiles
    * profiles.parquet
      * 包含profiles的 Parquet 表
    * symbols.symdb
      * 包含存储在block的profiles的符号信息。

### 数据模型

* block内的数据模型与 Google 的 pprof wire format的 proto 定义基本一致。
* Profile series labels 包含在数据ingestion时收集的附加信息，可用于选择特定的profiles。
  * 它们类似于 Prometheus/Loki labels，典型的labels名称为namespace和 Pod，用于描述profiles来自哪个工作负载。
* 每个ingested的profile都会添加到 profile 表中的新行中。
  * 如果不同型号的表格中缺少条目，也会将其插入。

![数据模型](./images/data-model.svg)

## hash rings

* 哈希环是一种分布式一致性哈希方案，被 Pyroscope 广泛用于分片(sharding)和复制(replication)。

### Pyroscope 中哈希环的工作原理

* Pyroscope 中的哈希环用于以一致的方式在组件的多个副本之间共享工作负载，以便任何其他组件都可以决定与哪个地址通信。
  * 要共享的工作负载或数据首先会被哈希处理，哈希结果用于确定其所属的环成员。
* Pyroscope 使用 fnv32a 哈希函数
  * fnv32a函数返回 32 位无符号整数
  * 因此其值介于 0 和 (2^32)-1 之间（含 0 和 2^32）
  * 该值称为 token，用作数据的 ID
  * token 确定性地决定了数据在哈希环上的位置
  * 这使得可以独立确定哪个 Pyroscope 实例是任何特定数据的权威所有者。
* 例如: 
  * profiles会被分片到不同的ingesters中
  * 给定profiles的令牌是通过对该profiles的所有标签和租户 ID 进行哈希运算计算得出的：结果是一个位于令牌空间内的 32 位无符号整数。
  * 拥有该令牌范围的ingester实例拥有该令牌序列，包括profiles令牌。
* 为了将可能的令牌集合 (2^32) 分配到集群中可用的实例上，给定 Pyroscope 组件（例如ingester）的所有运行实例都会加入一个哈希环。
  * 哈希环是一种数据结构，它将令牌空间分割成多个范围，并将每个范围分配给一个给定的 Pyroscope 环成员。
* 启动时，实例会生成随机的令牌值，并将其注册到环中。
  * 每个实例注册的值决定了给定令牌的归属。
  * 令牌的归属对象是注册了大于待查找令牌值的最小实例（当值达到 (2^32)-1 时，该值会回绕到零）。
* 为了在多个实例之间复制数据，Pyroscope 从数据的权威所有者开始，顺时针方向遍历环，寻找副本。
  * 数据会复制到遍历环过程中找到的下一个实例。

#### 一个实例

* 为了更好地理解其工作原理，我们以四个ingester和一个介于 0 到 9 之间的令牌空间为例：
  * ingester #1 在环中注册，令牌为 2
  * ingester #2 在环中注册，令牌为 4
  * ingester #3 在环中注册，令牌为 6
  * ingester #4 在环中注册，令牌为 9
* Pyroscope 接收到一个带有标签 {__name__="process_cpu", instance="1.1.1.1"} 的profile。它对分析报告的标签进行哈希运算，哈希函数的结果即为令牌 3。
* 为了找到令牌 3 的所有者，Pyroscope 在环中查找令牌 3，并找到注册令牌大于 3 的最小ingester。
  * 注册令牌为 4 的ingester #2 是分析报告 {__name__="process_cpu", instance="1.1.1.1"} 的权威所有者。

![非复制的hash环](./images/hash-ring-without-replication.png)

* 当复制次数设置为 3 时，Pyroscope 会将每个profile复制到3个ingester。
  * 找到profile的权威所有者后，Pyroscope 会继续顺时针遍历环路，找到剩余的两个需要复制该profile的实例。
  * 在下面的示例中，profile被复制到ingester #3 和 #4 的实例。

![复制的hash环](./images/hash-ring-with-replication.png)

#### 一致性hash

* 哈希环保证了一致性哈希的特性
* 当向环中添加或移除实例时，一致性哈希会最大限度地减少从一个实例移动到另一个实例的令牌数量。
  * 平均而言，需要移动到不同实例的令牌数量仅为 n/m，其中 n 是令牌总数（32 位无符号整数），m 是环中注册的实例数量

### 使用hash环的组件

有几个 Pyroscope 组件需要哈希环。以下每个组件都会构建一个独立的哈希环：

* Ingester 对数据序列进行分片和复制。
* Distributor 强制执行速率限制。

### Pyroscope 实例之间如何共享哈希环

* 哈希环数据结构需要在 Pyroscope 实例之间共享
  * 为了将更改传播到哈希环，Pyroscope 使用键值存储(key-value store)。
  * 键值存储(key-value store)是必需的，并且可以针对不同组件的哈希环进行独立配置。

### 使用哈希环构建的功能

* Pyroscope 主要使用哈希环进行分片和复制
* 使用哈希环构建的功能包括：
  * 服务发现
    * 实例可以通过查找已注册到哈希环的实例来相互发现
  * 心跳
    * 实例会定期向哈希环发送心跳信号，以表明其正在运行。
    * 如果实例在一段时间内未收到心跳信号，则被视为不健康。

## memberlist and gossip protocol

* Memberlist 是一个 Go 语言库
  * 它使用基于 gossip 的协议来管理集群成员关系、节点故障检测和消息传递
  * Memberlist 是最终一致性模型, 并通过尝试通过多条路由与可能已失效的节点通信来部分容忍网络分区(network partitions)。
* Pyroscope 使用 memberlist 在实例之间实现哈希环数据结构。
* 每个实例都维护着哈希环的副本
  * 每个 Pyroscope 实例在本地更新哈希环,并使用 memberlist 将更改传播到其他实例
  * 本地生成的更新和从其他实例接收的更新会合并在一起，形成实例上哈希环的当前状态。

### 成员列表如何传播哈希环变更

* 使用基于memberlist的键值存储时，每个 Pyroscope 实例都会使用以下技术将哈希环数据结构传播到其他实例：
  * 仅传播最近变更引入的差异。
  * 传播完整的哈希环数据结构。
* 定时传播增量: 每隔 `-memberlist.gossip-interval` ，实例会随机选择由 `-memberlist.gossip-nodes` 配置的所有 Pyroscope 集群实例的子集，并将最新变更发送到选定的实例。
  * 此操作频繁执行，是传播变更的主要技术。
* 定时传播全量: 此外，每隔 `-memberlist.pullpush-interval`，实例会随机选择 Pyroscope 集群中的另一个实例，并将键值存储的全部内容（包括所有哈希环）传输给该实例
  * 除非 `-memberlist.pullpush-interval` 为零，这将禁用此行为
  * 此操作完成后，两个实例的键值存储内容将完全相同
  * 此操作计算成本更高，因此执行频率较低
  * 该操作确保哈希环定期协调一致，达到共同状态

## 参考文档

1. [Pyroscope architecture](https://grafana.com/docs/pyroscope/latest/reference-pyroscope-architecture/)