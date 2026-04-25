# Native 查询与存储执行层详解

本文档聚焦 LanceDB 在 native 路径下的查询与存储执行层，也就是本地目录、对象存储，以及部分 namespace-backed 但仍由本进程直接执行的路径。

官方在线文档主要覆盖这一路径的对外语义，例如嵌入式运行模式、对象存储接入、一致性、版本与时间旅行等；本文进一步展开到 `DatasetConsistencyWrapper`、`InsertExec`、DataFusion 适配等内部执行节点，这部分内容主要基于当前仓库代码整理，不构成公开 API 或兼容性承诺。

这里的“native”主要指：

- 表对象是 `NativeTable`
- 数据直接由当前进程访问 Lance dataset
- 查询通常在本地进程内完成规划与执行
- 写入最终直接落到 Lance dataset，而不是转成远端 REST 调用

与之相对的，是 `RemoteTable` 所代表的远端执行路径。本文不展开远端实现细节，只在必要时做对比。

## 1 范围与定位

在 LanceDB 的整体架构中，native 查询与存储执行层位于：

- 上层 SDK / `Connection` / `Database` 之下
- Lance dataset / object store 之上

它负责把用户的“读写表”请求，真正转换成：

- 针对 Lance dataset 的本地查询计划
- 针对本地或对象存储的写入与提交
- 针对版本、一致性、时间旅行和索引状态的维护

从代码结构上看，这一层的核心实现分布在以下文件中：

- `rust/lancedb/src/table.rs`
- `rust/lancedb/src/table/query.rs`
- `rust/lancedb/src/table/dataset.rs`
- `rust/lancedb/src/table/add_data.rs`
- `rust/lancedb/src/table/datafusion.rs`
- `rust/lancedb/src/table/datafusion/insert.rs`
- `rust/lancedb/src/database/listing.rs`

## 2 运行时视图

从运行时角度看，native 查询与存储执行层大致可以表示为：

```text
应用 / SDK
   |
   v
Connection
   |
   v
ListingDatabase / LanceNamespaceDatabase
   |
   v
NativeTable
   |
   +-- DatasetConsistencyWrapper
   |      |
   |      +-- 当前可读 dataset 指针
   |      +-- 一致性刷新
   |      +-- time travel / checkout_latest
   |
   +-- 查询路径
   |      |
   |      +-- query builder / AnyQuery
   |      +-- query planner (`table/query.rs`)
   |      +-- Lance scanner
   |      +-- DataFusion ExecutionPlan
   |      +-- RecordBatch stream
   |
   +-- 写入路径
   |      |
   |      +-- AddDataBuilder 预处理
   |      +-- ScannableExec / cast / reject_nan
   |      +-- RepartitionExec
   |      +-- InsertExec
   |      +-- CommitBuilder 提交
   |      +-- DatasetConsistencyWrapper.update(...)
   |
   +-- 可选 namespace 能力
          |
          +-- managed versioning
          +-- QueryTable pushdown
          +-- storage options provider

底层依赖：
   |
   +-- Lance dataset
   +-- 本地文件系统 / S3 / GCS / OSS 等对象存储
   +-- session / object store registry / storage options
```

理解这层的关键在于：

- `NativeTable` 是对外的本地表对象
- `DatasetConsistencyWrapper` 负责 dataset 指针、一致性和 time travel
- `table/query.rs` 负责把查询请求翻译成可执行计划
- `table/add_data.rs` 与 `InsertExec` 负责把写入请求翻译成可提交事务
- Lance dataset 是最终的存储与扫描基础设施

## 3 核心组件

### 3.1 ListingDatabase

`ListingDatabase` 是 native 路径最常见的数据库实现。它负责：

- 解析本地目录或对象存储 URI
- 初始化 object store、base path 和 session
- 继承和注入 storage options
- 为建表和开表准备 `WriteParams` / `ReadParams`
- 把表打开为 `NativeTable`

它本身不负责执行具体查询，但决定了 native 表是如何接入存储层的。

从实现上看，`ListingDatabase` 内部持有：

- `object_store`
- `base_path`
- `storage_options`
- `storage_options_provider`
- `session`
- `namespace_database`

其中 `session` 很关键，因为它把 object store registry、缓存和连接复用能力统一了起来。

### 3.2 NativeTable

`NativeTable` 是 native 路径中的核心表实现。它内部包含：

- 表名、namespace、id、uri
- `DatasetConsistencyWrapper`
- 读一致性配置
- 可选 `namespace_client`
- 可选的 pushdown 操作集合

可以把它理解为：

“面向用户的本地表对象 + 指向当前 Lance dataset 的可变句柄 + 若干执行策略配置”

它既负责：

- 查询入口
- 写入入口
- schema / version / tag 等元数据操作

也负责把这些操作分发到更底层的 query、add、update、merge、optimize 等模块。

### 3.3 DatasetConsistencyWrapper

`DatasetConsistencyWrapper` 是 native 路径里非常重要但容易被忽略的一层。

它包住的是一个当前可用的 `Dataset` 指针，并在此基础上提供：

- Lazy / Strong / Eventual 三种读一致性模式
- time travel 版本固定
- checkout latest
- reload
- 写入完成后的 dataset 更新

这一层存在的原因是：

- 读请求需要一个“当前可读”的 dataset 引用
- 写入后 dataset 版本会变化，表对象需要感知
- 表对象可能处于最新版本模式，也可能处于 time travel 模式

换句话说，native 路径不是每次都重新打开一个 dataset，而是通过 wrapper 来维护当前 dataset 的视图。

### 3.4 Query Planner

`rust/lancedb/src/table/query.rs` 是 native 查询规划的核心实现。

它负责：

- 根据 `AnyQuery` 归一化查询请求
- 选择本地执行还是 namespace server pushdown
- 从当前 dataset 构造 `Scanner`
- 配置向量检索、全文检索、过滤、投影、limit、offset 等条件
- 生成最终的 `ExecutionPlan`

这层可以看作 native 查询路径中的“编排器”。

### 3.5 DataFusion 集成层

`rust/lancedb/src/table/datafusion.rs` 负责把 LanceDB 表适配成 DataFusion 的 `TableProvider`。

这一层的意义不只是“支持 SQL”，更重要的是：

- 让 LanceDB 可以接入 DataFusion 的计划与表达式系统
- 让表扫描和插入操作暴露为 DataFusion 可识别的执行节点
- 在查询与写入上复用 DataFusion 生态里的执行基础设施

其中几个重要角色包括：

- `BaseTableAdapter`
- `MetadataEraserExec`
- `InsertExec`
- `ScannableExec`

### 3.6 AddDataBuilder 与 InsertExec

native 写入并不是简单地把输入 batch 直接塞给 dataset。

它大致分成两步：

1. `AddDataBuilder` 先把用户输入预处理成一个 DataFusion `ExecutionPlan`
2. `InsertExec` 再消费这个计划，把结果写入 Lance dataset 并提交事务

这样做的好处是：

- 写入前可以做 schema 校验和 cast
- 可以在写入前插入 embedding 生成逻辑
- 可以插入 NaN 向量校验
- 可以用 DataFusion 的重分区能力提高并行写入效率

## 4 查询执行链路

### 4.1 从 Table 到 NativeTable

用户通过 SDK 调用 `table.query()`、`vector_search()` 或更高层的查询接口时，最终会落到 `NativeTable`。

在 `BaseTable` 层，native 查询主要暴露三类接口：

- `create_plan(...)`
- `query(...)`
- `analyze_plan(...)`

在 `NativeTable` 的实现里，这三者分别委托给：

- `query::create_plan(...)`
- `query::execute_query(...)`
- `query::analyze_query_plan(...)`

因此，`NativeTable` 本身更像一个路由层，真正的查询规划逻辑在 `table/query.rs`。

### 4.2 本地执行还是 namespace pushdown

native 查询并不总是意味着“只能本地执行”。

`execute_query(...)` 的第一步会先判断：

- 是否启用了 `QueryTable` pushdown
- 是否存在可用的 `namespace_client`

如果这两个条件都满足，查询会走 `execute_namespace_query(...)`，把请求转换成 namespace `query_table` API。

否则，才进入真正的本地执行路径 `execute_generic_query(...)`。

也就是说，native table 是“本地表实现”，但并不排斥在某些条件下把查询下推给 server。

### 4.3 查询请求归一化

`create_plan(...)` 会先把查询统一整理成 `VectorQueryRequest`。

这一步有两个作用：

- 普通查询和向量查询可以走统一的底层构建逻辑
- 后续 planner 只需要围绕一个主查询结构工作

在这一步之后，planner 会拿到：

- 当前 dataset
- 当前 schema
- 查询向量
- 可选的目标列
- limit / offset / filter / projection / FTS / distance 等参数

### 4.4 获取当前 dataset 视图

planner 在开始构造计划前，会先调用：

```text
table.dataset.get().await
```

这一步拿到的不是裸 `Dataset`，而是经过 `DatasetConsistencyWrapper` 管理后的当前 dataset 视图。

它的行为取决于一致性模式：

- Lazy：直接返回缓存 dataset
- Strong：读取前刷新到最新版本
- Eventual：优先返回缓存，并在 TTL 到期时异步或同步刷新

如果表当前处于 time travel 模式，则始终返回被固定的版本。

这意味着查询层看到的 dataset 已经带有“一致性语义”，而不是完全裸露的存储对象。

### 4.5 Scanner 构造

拿到 dataset 后，planner 会创建 Lance 的 `Scanner`：

```text
let mut scanner: Scanner = ds_ref.scan();
```

后续绝大多数查询条件，都会被映射到这个 scanner 上。

如果查询包含向量，planner 会进一步：

- 确定或推断向量列
- 判断是普通向量还是二进制向量
- 计算 `top_k`
- 调用 `scanner.nearest(...)`
- 设置 `minimum_nprobes`、`maximum_nprobes`
- 设置 `ef`、`distance_range`、`distance_metric`
- 设置 `refine_factor`

这一步是 native 向量检索真正落到 Lance 能力上的关键位置。

### 4.6 多向量查询

`table/query.rs` 对多向量查询做了特殊处理。

如果请求里带有多个 query vector，会分两种情况：

- 如果目标列本身是 list 结构，则把多个向量拼成更高阶结构
- 如果不是，则为每个向量分别构造子计划，最后用 union 方式合并

这说明 native 查询层不仅是一个“参数透传层”，还承担了多向量查询的计划拆解与重组职责。

### 4.7 过滤、投影与全文检索

在向量相关配置之外，planner 还会继续把其他查询条件映射到 scanner：

- `limit` / `offset`
- `prefilter`
- SQL filter
- Substrait filter
- DataFusion expr filter
- 列投影
- 动态投影
- 基于表达式的投影
- `with_row_id`
- `fast_search`
- `full_text_search`
- `disable_scoring_autoprojection`

这些配置最终共同组成一个 Lance scanner plan，再由：

```text
scanner.create_plan().await
```

生成真正的 `ExecutionPlan`。

### 4.8 执行计划与结果流

在本地执行路径下，`execute_generic_query(...)` 会：

1. 调用 `create_plan(...)`
2. 通过 `execute_plan(...)` 执行计划
3. 再用 `MaxBatchLengthStream` 和 `TimeoutStream` 包装输出流
4. 最终返回 `DatasetRecordBatchStream`

因此 native 查询的最终结果并不是一次性 materialize 出来的大结果集，而是 Arrow `RecordBatch` 流。

这和 LanceDB 整体的列式、流式执行模型是一致的。

### 4.9 analyze_plan

`analyze_query_plan(...)` 的逻辑很直接：

- 先构造 plan
- 再调用 `lance_analyze_plan(...)`

这使得 native 路径不仅能执行查询，也能直接暴露底层计划结构，方便调试与性能分析。

## 5 DataFusion 在 native 查询中的角色

### 5.1 BaseTableAdapter

`BaseTableAdapter` 把 `BaseTable` 适配成 DataFusion 的 `TableProvider`。

这意味着原本从 LanceDB API 发起的查询，也可以被嵌入到 DataFusion 的上下文中。

在 `scan(...)` 中，`BaseTableAdapter` 会把 DataFusion 的：

- projection
- filters
- limit

重新组装成 LanceDB 的 `QueryRequest`，然后调用：

```text
table.create_plan(&AnyQuery::Query(query), options)
```

也就是说，DataFusion 对 LanceDB 而言并不是完全独立的第二套查询系统，而是共享同一套 `BaseTable -> create_plan` 能力。

### 5.2 MetadataEraserExec

`MetadataEraserExec` 的作用是去掉 batch metadata。

这是一个很小但很实际的适配层，目的是避免 DataFusion 在传播 Arrow metadata 时引发不必要的问题。

它说明 native 执行层不仅要“能跑”，还要处理 LanceDB 与 DataFusion 在细节上的行为差异。

### 5.3 Filter Pushdown

`BaseTableAdapter` 对 DataFusion filter 的处理方式是：

- 把多个 filter expr 合并成一个 DataFusion 表达式
- 再放入 `QueryFilter::Datafusion`
- 最终让 native planner 把这个 filter 继续下沉到 scanner

从架构上看，这意味着：

- 上层 SQL / DataFusion 看到的是 DataFusion filter
- native 执行层看到的是统一后的 LanceDB filter 表达
- 真正执行时仍尽量落到 Lance scanner

## 6 写入与存储执行链路

native 路径下的写入，可以分成三种典型入口：

- 建表时的初始写入
- 已有表上的 `add(...)`
- 通过 DataFusion / SQL 触发的 `INSERT INTO`

### 6.1 建表时的初始写入

在 `ListingDatabase::create_table(...)` 中，本地建表大致经历以下步骤：

1. 确定表 URI
2. 提取和合并 storage overrides
3. 准备 `WriteParams`
4. 调用 `NativeTable::create(...)`

`prepare_write_params(...)` 会把连接级配置写入最终的 `WriteParams`，包括：

- storage options
- storage options provider
- data storage version
- manifest path 配置
- stable row ids
- overwrite 模式
- session

然后 `NativeTable::create(...)` 会进一步：

- 用 `InsertBuilder::new(uri).with_params(&params)` 创建写入器
- 调用 `execute_stream(...)` 写入初始数据
- 返回新的 dataset
- 用这个 dataset 初始化 `DatasetConsistencyWrapper`

因此，native 建表不是“先建空表再补写数据”，而是直接通过 Lance 的写入路径落成第一个 dataset 版本。

### 6.2 打开表时的存储接入

`ListingDatabase::open_table(...)` 负责把本地表重新接回 native 执行层。

这一步会准备：

- `ReadParams`
- 继承的 storage options
- 可选的 storage options provider
- session
- index cache 配置

最终通过 `NativeTable::open_with_params(...)`：

- 构造 `DatasetBuilder`
- 根据需要挂上 commit handler
- 加载 dataset
- 用 `DatasetConsistencyWrapper::new_latest(...)` 包装

也就是说，native 读路径的“存储接入点”主要在 `ListingDatabase`，而不是分散在每个查询调用里。

### 6.3 AddDataBuilder 预处理

对于已有表上的 `add(...)`，LanceDB 并不会直接把用户数据写盘，而是先做一轮预处理。

`AddDataBuilder::into_plan(...)` 主要负责：

- 检查输入 schema 与表 schema 是否兼容
- 根据表定义应用 embedding 逻辑
- 用 `ScannableExec` 把输入包装成 DataFusion plan
- 必要时 cast 到目标表 schema
- 根据配置拒绝包含 NaN 的向量

最终输出的是 `PreprocessingOutput`，其中包含：

- 预处理后的 `ExecutionPlan`
- 是否 overwrite
- 是否可重扫
- write options
- 写入模式
- 进度跟踪器

这一步非常关键，因为它把原始输入数据转换成了“可执行写入计划”，为后续并行写入和事务提交打下基础。

### 6.4 并行写入与重分区

`NativeTable::add(...)` 在预处理之后，还会决定写入并行度。

如果用户没有显式设置 `write_parallelism`，它会：

- 先 peek 第一批数据
- 根据 batch 大小、行数和 CPU 情况估算分区数

如果最终分区数大于 1，就会给预处理后的 plan 再包一层：

- `RepartitionExec`

采用 `RoundRobinBatch` 方式把输入打散到多个分区。

这说明 native 写入路径不是单线程串行地推数据，而是显式利用 DataFusion 分区模型做并行写入。

### 6.5 InsertExec 的提交流程

`InsertExec` 是 native 写入路径里最核心的执行节点之一。

它的执行逻辑是：

1. 每个分区各自执行 `execute_uncommitted_stream(...)`
2. 每个分区先产生未提交事务
3. 所有分区完成后，把事务合并
4. 由最后完成的分区统一提交
5. 提交完成后，用新 dataset 更新 `DatasetConsistencyWrapper`

也就是说，native 并行写入的模型不是“每个分区各自独立提交”，而是：

- 分区并行写数据
- 提交阶段统一合并事务并原子提交

这样做可以减少并发提交带来的协调复杂度，也能让表对象在提交完成后切换到最新 dataset 版本。

### 6.6 写入后的 dataset 更新

事务提交成功后，`InsertExec` 会调用：

```text
ds_wrapper.update(new_dataset)
```

这一步的意义非常大：

- 它让当前表对象感知到最新版本的 dataset
- 它保证后续读请求不必重新手工打开表
- 如果 wrapper 当前处于 Eventual 模式，还会触发缓存失效

也就是说，native 写入链路最终不是只把数据写到存储里，还要把“表对象当前看到的数据版本”一并推进。

### 6.7 SQL INSERT 路径

native 路径还支持通过 DataFusion / SQL 写入。

这条路径的关键在于：

- `BaseTableAdapter::insert_into(...)`
- `NativeTable::create_insert_exec(...)`

DataFusion 在遇到 `INSERT INTO` 时，会把输入 plan 交给 LanceDB 的 `create_insert_exec(...)`，后者再构造 `InsertExec`。

这意味着 SQL 写入与 API 写入并不是两套完全不同的底层实现，最终都会汇聚到同一个 native 插入执行节点。

## 7 一致性、版本与时间旅行

### 7.1 为什么需要 DatasetConsistencyWrapper

在 native 路径下，表对象是长生命周期对象。

如果没有 `DatasetConsistencyWrapper`，就会遇到几个问题：

- 写入后表对象不知道 dataset 已经变成新版本
- 多个表实例之间如何处理可见性
- time travel 与 latest 模式如何切换
- 何时刷新 dataset 元数据

wrapper 的作用，就是把这些“当前表看到哪个版本”的问题统一收口。

### 7.2 三种一致性模式

wrapper 支持三种一致性模式：

- Lazy
  直接返回当前缓存的 dataset，不主动刷新

- Strong
  每次读取前都刷新到最新版本

- Eventual
  以 TTL 为基础做后台刷新或到期刷新

这三种模式主要影响：

- `get()` 的行为
- 表对象多久能看到其他写入者产生的新版本

### 7.3 time travel 与 latest 模式

wrapper 还支持：

- `as_time_travel(...)`
- `as_latest(...)`
- `reload()`

当表进入 time travel 模式后：

- 当前 dataset 会固定到指定版本
- 写入会被禁止
- 读请求总是针对固定版本执行

恢复到 latest 模式后，表才会重新跟踪最新版本。

因此，native 路径下的版本语义并不是散落在各个 API 里，而是通过 wrapper 统一维护。

## 8 与 namespace 的关系

native 路径不等于“完全不依赖 namespace”。

实际上，namespace 相关能力在 native 层至少体现在三处：

- 查询下推
  当启用 `QueryTable` pushdown 且存在 `namespace_client` 时，native table 可以把查询下推到 namespace server。

- managed versioning
  `NativeTable::open_with_params(...)` 在需要时会挂载基于 namespace 的 external manifest commit handler，从而让版本提交纳入 namespace 管理。

- storage options provider
  `create_from_namespace(...)` 可以通过 namespace client 获取存储访问配置，支持凭证动态刷新等场景。

所以更准确地说，native 层是“以本地 dataset 为中心的执行层”，但它已经预留了和 namespace 协作的能力。

## 9 与 RemoteTable 的差异

从职责上看，`NativeTable` 和 `RemoteTable` 都实现了 `BaseTable`，但底层模型完全不同。

`NativeTable` 的特点是：

- 直接持有 dataset 视图
- 本地构造 scanner 和 `ExecutionPlan`
- 写入时直接提交 Lance 事务
- 本地维护一致性与 time travel 状态

`RemoteTable` 的特点是：

- 不直接持有底层 dataset
- 通过 REST 调远端服务
- 查询与写入主要在服务端完成
- 客户端侧更像协议适配层

因此，native 查询与存储执行层可以看作 LanceDB 真正“嵌入式数据库”能力最集中的部分。

## 10 调试与阅读建议

如果你想沿着代码阅读 native 查询与存储执行层，比较推荐按下面顺序看：

1. `rust/lancedb/src/table.rs`
   先理解 `NativeTable` 的字段和 `BaseTable` 实现入口。

2. `rust/lancedb/src/table/dataset.rs`
   再理解 dataset wrapper 如何维护一致性、版本和 time travel。

3. `rust/lancedb/src/table/query.rs`
   看查询请求如何被转换成 scanner 和 `ExecutionPlan`。

4. `rust/lancedb/src/table/datafusion.rs`
   看 LanceDB 如何与 DataFusion 的 `TableProvider` 模型对接。

5. `rust/lancedb/src/table/add_data.rs`
   看写入前的预处理是怎么做的。

6. `rust/lancedb/src/table/datafusion/insert.rs`
   看真正的并行写入、事务合并和提交逻辑。

7. `rust/lancedb/src/database/listing.rs`
   最后回到数据库层，看本地表是如何接入 object store、session 和 storage options 的。

## 11 总结

可以把 LanceDB 的 native 查询与存储执行层概括为三句话：

- 查询侧，它把统一的表查询请求翻译为 Lance scanner 和 DataFusion `ExecutionPlan`
- 写入侧，它把用户输入先预处理为可执行计划，再通过 `InsertExec` 并行写入并原子提交
- 状态侧，它通过 `DatasetConsistencyWrapper` 统一维护 dataset 视图、一致性、版本和 time travel

这三部分共同构成了 LanceDB 在本地 / 对象存储场景下最核心的执行基础。
