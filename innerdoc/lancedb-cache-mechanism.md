# LanceDB缓存机制

## 1 先看全貌：LanceDB到底缓存了什么

如果只看用户接口，很容易把 LanceDB 的缓存理解成一个 `index_cache_size` 参数。但从当前仓库代码来看，LanceDB 实际上缓存了多种不同对象，而且这些缓存分布在不同路径上。

先不看代码实现，先看“缓存对象”和“要解决的问题”。

| 缓存对象 | 主要出现在哪条路径 | 解决的核心问题 | 典型代表 |
| --- | --- | --- | --- |
| 索引数据、元数据、对象存储相关共享状态 | 本地 OSS / namespace | 避免重复加载索引、重复解析元数据、重复建立对象存储访问上下文 | `lance::session::Session` |
| 当前表对应的 `Dataset` 句柄 | 本地 OSS / namespace | 在“读性能”和“外部变更可见性”之间做平衡 | `DatasetConsistencyWrapper` |
| 远端表对象本身 | Remote / Cloud | 避免重复 `open/describe` 同一张表并重复构造 `RemoteTable` | `RemoteDatabase.table_cache` |
| 远端表的 schema | Remote / Cloud | 避免频繁向服务端拉取 schema | `RemoteTable.schema_cache` |
| 远端表的 location / URI | Remote / Cloud | 避免每次取 URI 都重新请求 `describe` | `RemoteTable.location` |
| 热点 table data | Enterprise 服务端 | 避免 Plan Executor 每次都回源 S3 / GCS / Azure Blob 读取热数据 | 分布式 NVMe SSD cache |

从这个角度看，LanceDB 的缓存机制不是“一个统一缓存池”，而是一套分层缓存体系：

1. 本地路径缓存的是底层资源和表状态。
2. Remote 客户端路径缓存的是远端表对象和元信息。
3. Enterprise 服务端路径缓存的是靠近执行节点的热点表数据。
4. 语言绑定层主要负责把这些能力暴露给用户，而不是自己实现缓存算法。

## 2 这些缓存分别在解决什么问题

### 2.1 本地路径最关心的问题

本地 OSS / namespace 路径下，缓存主要在解决三类问题：

1. 不要每次访问表都重新构建底层对象存储访问上下文。
2. 不要每次读都重新加载索引和元数据。
3. 不要为了保证一致性而把每次读都变成“强制刷新最新版本”。

因此，本地路径用了两层机制：

1. `Session` 负责底层资源复用。
2. `DatasetConsistencyWrapper` 负责“当前表视图”如何刷新。

### 2.2 Remote 客户端路径最关心的问题

Remote / Cloud 路径面对的是另一组问题：

1. 同一张远端表反复 `open_table()` 时，没必要每次都重新构造表对象。
2. schema 这类元信息不应该每次查询前都重新拉取。
3. 表 location 这类稳定信息也没必要反复请求服务端。

因此，Remote 客户端路径更偏向“对象缓存”和“元信息缓存”，而不是本地数据块缓存。

这里说的“Remote 客户端路径”指当前仓库里的 SDK / Rust client 侧 `RemoteDatabase`、`RemoteTable` 实现。它不是 LanceDB Enterprise 服务端内部的 Plan Executor 数据缓存。

### 2.3 Enterprise 服务端最关心的问题

LanceDB Enterprise 面对的问题又不一样：它要把对象存储作为 durable source of truth，同时让高并发查询不要反复为同一批热点数据支付对象存储访问延迟。

根据官方 Enterprise 文档，Enterprise 的缓存重点是：

1. 查询侧由 Query Node 和 Plan Executor 执行。
2. Plan Executor 使用本地 NVMe SSD 做 cache-backed reads。
3. 缓存的是 table data，而不是 table indices。
4. 缓存 miss 时回源对象存储，随后填充本地缓存。
5. 热点数据会通过分布式缓存留在靠近执行节点的位置，从而降低重复 S3 / GCS / Blob 读取和 egress 成本。

这层能力不在当前 OSS 仓库中实现，而是 Enterprise 服务端集群能力。

### 2.4 一个很重要的边界

当前仓库中的证据表明：

1. 本地 OSS 路径确实会消费 `Session`，把它传到 object store 和 dataset 读写链路中。
2. Cloud / Enterprise remote 客户端路径不会消费这个本地 `Session`。
3. 但 remote 客户端路径并不是没有缓存，而是换成了 `RemoteTable`、schema、location 这些更贴近控制面的缓存对象。
4. Enterprise 服务端还有另一套分布式 NVMe 数据缓存，用来减少对象存储读取；这不是 SDK 里的 `Session` 或 `RemoteTable.schema_cache`。

所以不能简单说“LanceDB 有缓存”或者“Cloud 没缓存”。更准确的说法应该是：

- 本地 OSS 路径：以资源缓存和表状态缓存为主
- Remote 客户端路径：以表对象和元信息缓存为主
- Enterprise 服务端路径：以分布式热点 table data 缓存为主

## 3 可以把 LanceDB 的缓存机制分成三条主线

### 3.1 本地 OSS / namespace 路径

这一条路径的核心缓存对象有两个：

1. `Session`
2. `DatasetConsistencyWrapper`

它们分别负责不同层面的事情：

- `Session` 解决“底层资源怎么复用”
- `DatasetConsistencyWrapper` 解决“当前表视图何时刷新”

### 3.2 Remote / Cloud 路径

这一条路径的核心缓存对象有三个：

1. `RemoteDatabase.table_cache`
2. `RemoteTable.schema_cache`
3. `RemoteTable.location`

它们分别负责：

- 避免重复构造远端表对象
- 避免重复拉 schema
- 避免重复拉 location

### 3.3 Enterprise 服务端路径

这一条路径的核心缓存对象不是当前仓库里的 Rust struct，而是 Enterprise 服务端的分布式缓存层：

1. Plan Executor 的 NVMe SSD cache
2. 服务端缓存分片和副本
3. 对象存储回源路径

它们分别负责：

- 缓存热点 table data
- 让后续查询从靠近计算的位置读取数据
- 在 cache miss 时回源对象存储并填充缓存

### 3.4 一张“本地路径 vs remote 客户端路径 vs Enterprise 服务端路径”的缓存关系图

下面这张图把“谁在缓存什么”和“哪些是公共原语、哪些是模块自己管理”放到一张图里：

```text
LanceDB cache architecture
|
|-- Local OSS / namespace path
|   |
|   |-- Shared cache context: Session  (re-export from lance)
|   |   |
|   |   |-- index cache
|   |   |-- metadata cache
|   |   `-- object store registry / shared object-store state
|   |
|   `-- Table view cache: DatasetConsistencyWrapper
|       |
|       |-- Lazy
|       |-- Strong
|       `-- Eventual
|           `-- uses BackgroundCache<Arc<Dataset>, Error>
|
|-- Remote / Cloud path
|   |
|   |-- Database-level object cache: RemoteDatabase.table_cache
|   |   `-- caches Arc<RemoteTable>
|   |
|   `-- Table-level metadata/state cache: RemoteTable
|       |
|       |-- schema_cache
|       |   `-- uses BackgroundCache<SchemaRef, Error>
|       |
|       `-- location
|           `-- caches Option<String> in RwLock
|
`-- Enterprise server-side path  (outside this OSS repo)
    |
    |-- Query Node
    |   `-- validates, plans, and routes remote table requests
    |
    |-- Plan Executor fleet
    |   |
    |   |-- distributed NVMe SSD cache
    |   |   `-- caches hot table data, not table indices
    |   |
    |   `-- cache miss
    |       `-- read from object storage and populate local cache shard
    |
    `-- Object storage
        `-- durable table data, manifests, and index artifacts

Reusable cache primitives / building blocks
|
|-- Session
|   `-- shared cache context for local path, implemented in lance
|
`-- BackgroundCache
    `-- generic TTL + refresh-window + background-refresh cache in lancedb
```

看这张图时，有两个点最容易帮助建立正确心智模型：

1. 本地路径、remote 客户端路径和 Enterprise 服务端路径并不共享一个统一缓存池，而是不同层面的分层缓存。
2. 真正可复用的公共缓存原语主要是 `Session` 和 `BackgroundCache`，其余如 `RemoteDatabase.table_cache`、`RemoteTable.location`、`DatasetConsistencyWrapper` 更像是各模块基于这些能力搭建的具体缓存层。
3. Enterprise 的 NVMe 数据缓存是服务端集群能力，不能从当前 OSS 仓库代码直接推导实现细节，需要以 Enterprise 官方文档和部署配置为准。

## 4 本地路径：先理解 `Session`，再理解 `DatasetConsistencyWrapper`

### 4.1 `Session` 缓存的不是“某次查询结果”，而是底层资源

LanceDB 在 Rust 顶层直接 re-export 了 `lance::session::Session` 和 `ObjectStoreRegistry`：

- `rust/lancedb/src/lib.rs:266-267`

Python 和 Node 也把 `Session` 暴露成一等对象，而不是隐藏参数：

- Python：`python/src/session.rs:9-13`
- Node：`nodejs/src/session.rs:10-14`

从这些接口和注释可以直接确认，`Session` 至少承担三类职责：

1. index cache
2. metadata cache
3. object store registry

对应证据：

- Python 文档和构造参数：`python/src/session.rs:30-60`
- Node 文档和构造参数：`nodejs/src/session.rs:32-64`

也就是说，`Session` 缓存的重点不是“SQL/向量查询结果”，而是底层运行时会反复访问的资源。

### 4.2 `Session` 主要解决什么问题

从当前仓库代码，可以把它理解成在解决下面几个问题：

1. 索引数据不应每次都重新加载。
2. 元数据和 schema 信息不应每次都重新走完整读取流程。
3. object store 相关的共享状态应尽量复用，而不是为每张表重复创建。

这也是为什么 Python / Node 的 `Session` 都支持：

1. 配置 index cache 大小
2. 配置 metadata cache 大小
3. 观测 `size_bytes`
4. 观测 `approx_num_items`

对应代码：

- Python：`python/src/session.rs:43-97`
- Node：`nodejs/src/session.rs:43-87`

### 4.3 `Session` 在本地路径中的生命周期

如果用户不显式传入 session，本地 `ListingDatabase` 会自己创建默认 session：

- `rust/lancedb/src/database/listing.rs:398-401`
- `rust/lancedb/src/database/listing.rs:476-479`

随后它会立刻参与 object store 初始化：

- `rust/lancedb/src/database/listing.rs:412-416`
- `rust/lancedb/src/database/listing.rs:477-481`

这说明 `Session` 的定位不是“查询优化选项”，而是连接级上下文。

更准确地说，它的生命周期通常是：

`connect -> database -> object store -> read/write params -> dataset/table operations`

### 4.4 `Session` 如何沿调用链向下传递

本地写路径中，`ListingDatabase` 会把 session 写进 `WriteParams`：

- `rust/lancedb/src/database/listing.rs:717`

本地读路径中，`open_table()` 会把 session 注入 `ReadParams`：

- `rust/lancedb/src/database/listing.rs:1053-1061`

Namespace 路径也会继续透传：

- namespace connect 时向下传 session：`rust/lancedb/src/database/namespace.rs:82-91`
- namespace open/create table 时继续传给 `NativeTable`：
  - `rust/lancedb/src/database/namespace.rs:206-215`
  - `rust/lancedb/src/database/namespace.rs:330-340`
  - `rust/lancedb/src/database/namespace.rs:347-356`

`NativeTable` 层再把它并入底层读写参数：

- 读：`rust/lancedb/src/table.rs:1474-1480`
- 写：`rust/lancedb/src/table.rs:1701-1707`

### 4.5 当前仓库对 `Session` 能证明到什么程度

当前仓库可以直接证明：

1. `Session` 是 LanceDB 本地路径的核心缓存入口。
2. 它至少涵盖 index cache、metadata cache、object store registry。
3. 它可以跨连接复用，Python / Node 文档就是这样设计的。

但当前仓库不能直接证明：

1. 底层淘汰算法是不是 LRU。
2. `size_bytes` 的统计范围是否只包含 index 和 metadata cache。
3. object store registry 内部是否还分成更多子缓存。

原因很简单：`lance::session::Session` 的真实实现不在 `lancedb` 仓库里。

所以比较稳妥的结论是：

> LanceDB 暴露并透传了 Lance 的 session 级缓存能力，但更底层的缓存算法细节需要去 `lance` 仓库确认。

## 5 本地路径的第二层：缓存的不是资源，而是当前 `Dataset`

### 5.1 `DatasetConsistencyWrapper` 缓存的对象是什么

`DatasetConsistencyWrapper` 包装的是当前表对应的 `Arc<Dataset>`：

- `rust/lancedb/src/table/dataset.rs:13-20`

这和 `Session` 不是一个层面：

- `Session` 更像底层运行时缓存
- `DatasetConsistencyWrapper` 更像表级视图缓存

### 5.2 它主要解决什么问题

如果没有这层机制，本地表读取会面临一个两难：

1. 如果每次读都直接使用旧 dataset，性能好，但看不到其他进程的更新。
2. 如果每次读都强制刷新 latest，能看到更新，但性能差。

`DatasetConsistencyWrapper` 解决的就是这个折中问题。

### 5.3 `read_consistency_interval` 实际上也在控制缓存策略

`DatasetConsistencyWrapper` 有三种模式：

1. `Lazy`
2. `Strong`
3. `Eventual(BackgroundCache<Arc<Dataset>, Error>)`

代码位置：

- `rust/lancedb/src/table/dataset.rs:32-47`

它和 `read_consistency_interval` 的关系是：

1. `None` -> `Lazy`
2. `Some(Duration::ZERO)` -> `Strong`
3. `Some(d > 0)` -> `Eventual`

位置：

- `rust/lancedb/src/table/dataset.rs:52-63`

这意味着 `read_consistency_interval` 不只是“一致性开关”，也是 dataset 视图缓存的刷新策略开关。

### 5.4 `Eventual` 模式下，它是怎么工作的

在 eventual consistency 下：

1. TTL = `read_consistency_interval`
2. refresh window = `min(3s, TTL / 4)`

位置：

- `rust/lancedb/src/table/dataset.rs:42-47`
- `rust/lancedb/src/table/dataset.rs:57-60`

`get()` 的行为分三种：

1. 缓存仍然新鲜：直接返回当前 dataset
2. 进入 refresh window：后台刷新，但先返回旧值
3. 已经过期：同步等待刷新完成

位置：

- `rust/lancedb/src/table/dataset.rs:73-107`

所以它本质上缓存的是“当前表版本视图”，而不是某条查询结果。

### 5.5 写入之后这层缓存怎么更新

写操作完成后，`NativeTable` 会显式调用 `self.dataset.update(dataset)`：

- `rust/lancedb/src/table.rs:1769`
- `rust/lancedb/src/table.rs:2092`
- `rust/lancedb/src/table.rs:2110`
- `rust/lancedb/src/table.rs:2121`
- `rust/lancedb/src/table.rs:2135`
- `rust/lancedb/src/table.rs:2153`
- `rust/lancedb/src/table.rs:2350`
- `rust/lancedb/src/table.rs:2358`

而 `DatasetConsistencyWrapper::update()` 会：

1. 替换当前 dataset 指针
2. 如果是 eventual 模式，则让背景缓存失效并等待下次刷新

位置：

- `rust/lancedb/src/table/dataset.rs:110-132`

所以这不是一个纯被动缓存，而是读写协同缓存。

## 6 `index_cache_size` 为什么不是理解 LanceDB 缓存机制的正确入口

### 6.1 这个参数还在，但已经不再是主轴

Rust 的 `OpenTableBuilder` 仍然保留了 `index_cache_size`：

- `rust/lancedb/src/connection.rs:145-157`

Python 和 Node 也还保留这个参数：

- Python：`python/python/lancedb/db.py:415-429`
- Node：`nodejs/lancedb/connection.ts:85-98`

但这条路径已经明显退居兼容层：

- Python 会发弃用警告：`python/python/lancedb/db.py:951-959`
- Node 类型定义标记为 deprecated：`nodejs/lancedb/connection.ts:87-88`

### 6.2 为什么说它不是主入口

因为它和 `Session` 根本不是同一个抽象：

1. `index_cache_size` 是 entry 数量维度。
2. `Session` 是字节容量维度。
3. `index_cache_size` 只覆盖 index cache。
4. `Session` 同时覆盖 metadata cache 和 object store 共享上下文。

### 6.3 它在本地路径中实际上是怎样生效的

`ListingDatabase::open_table()` 中，`index_cache_size` 只会在“没有用户自定义 `ReadParams`”时写进默认 `ReadParams`：

- `rust/lancedb/src/database/listing.rs:1053-1058`

但之后无论如何仍然会再执行：

- `read_params.session(self.session.clone())`
- 位置：`rust/lancedb/src/database/listing.rs:1061`

所以它更像是：

- 一个局部兼容参数

而不是：

- LanceDB 缓存机制的核心入口

## 7 Remote 客户端路径：缓存的不是数据块，而是表对象和元信息

### 7.1 Remote / Cloud / Enterprise 客户端不会消费本地 `Session`

`ConnectBuilder::execute_remote()` 直接构造 `RemoteDatabase::try_new(...)`，并没有把 `self.request.session` 传进去：

- `rust/lancedb/src/connection.rs:846-868`

与之相对，本地路径会把 `request.session` 交给 `ListingDatabase::connect_with_options(...)`：

- `rust/lancedb/src/connection.rs:886-890`

因此可以确认：

1. `Session` 是本地 OSS / namespace 路径的缓存入口。
2. Remote 客户端路径不走这条链。

### 7.2 但 Remote 客户端路径并不是没有缓存

Remote 客户端路径至少有三类缓存对象：

1. `RemoteDatabase.table_cache`
2. `RemoteTable.schema_cache`
3. `RemoteTable.location`

它们缓存的是：

- 远端表对象
- schema
- location

而不是：

- 查询结果
- 远端数据页
- 远端索引块

如果连接的是 LanceDB Enterprise，这些客户端侧缓存仍然存在，但它们只负责减少客户端重复构造对象和重复拉元信息。Enterprise 的热点数据缓存发生在服务端 Plan Executor 层，和这里的 `RemoteDatabase.table_cache` / `RemoteTable.schema_cache` 不是同一层。

## 8 Remote 客户端路径的第一层：`RemoteDatabase.table_cache`

### 8.1 它缓存的对象是什么

`RemoteDatabase` 里有一个明确字段：

- `table_cache: Cache<String, Arc<RemoteTable<S>>>`
- 位置：`rust/lancedb/src/remote/db.rs:191-193`

这层缓存缓存的是 `RemoteTable` 对象本身。

### 8.2 它主要解决什么问题

它解决的是：

1. 同一张表重复 `open_table()` 时没必要总是重新向服务端确认并重建对象。
2. 同一个远端表对象上已经有的本地状态可以复用，例如后续的 schema cache、location cache。

### 8.3 它的容量和生命周期

初始化参数：

1. TTL = 300 秒
2. max_capacity = 10,000

位置：

- `rust/lancedb/src/remote/db.rs:238-241`

`open_table()` 行为：

1. 先按 key 查 `table_cache`
2. 命中则直接返回
3. 未命中才去 `describe`
4. 然后构造 `RemoteTable` 并写回缓存

位置：

- `rust/lancedb/src/remote/db.rs:564-598`

### 8.4 这层缓存的 key 为什么值得注意

`build_cache_key()` 使用长度前缀编码 namespace 和 table name，再 hex 编码：

- `rust/lancedb/src/remote/db.rs:328-336`

这能避免多级 namespace、分隔符冲突、表名中包含特殊字符时出现歧义。

### 8.5 失效和迁移

对于 rename / drop：

- rename 后会移除旧 key 并写入新 key：`rust/lancedb/src/remote/db.rs:611-631`
- drop 后会移除缓存：`rust/lancedb/src/remote/db.rs:638-642`

所以这不是一个“只进不出”的对象缓存。

## 9 Remote 客户端路径的第二层：schema cache 和 location cache

### 9.1 `schema_cache` 缓存的对象和目标

`RemoteTable` 中直接带有：

- `schema_cache: BackgroundCache<SchemaRef, Error>`
- 位置：`rust/lancedb/src/remote/table.rs:213-215`

初始化常量：

- `SCHEMA_CACHE_TTL = 30s`
- `SCHEMA_CACHE_REFRESH_WINDOW = 5s`
- 位置：`rust/lancedb/src/remote/table.rs:69-70`

它解决的是“schema 不应该每次都重新向服务端获取”。

### 9.2 `schema_cache` 如何工作

`schema()` 方法会先走缓存快路径：

- 如果 `try_get()` 命中，直接返回
- 否则通过 `get(...)` 拉取并写入缓存

位置：

- `rust/lancedb/src/remote/table.rs:1198-1213`

### 9.3 `schema_cache` 何时失效

代码中可以清楚看到三类失效源：

1. schema 相关错误
2. 版本切换
3. 明确的 schema 变更操作

错误触发失效的状态码只有少数几类：

- 400
- 404
- 422
- 500

位置：

- `rust/lancedb/src/remote/table.rs:383-393`
- `rust/lancedb/src/remote/table.rs:404-414`
- `rust/lancedb/src/remote/table.rs:766-776`

显式操作触发失效也有明确证据，例如：

- checkout / checkout_latest / checkout_tag：`rust/lancedb/src/remote/table.rs:1142-1153`, `1663-1665`
- overwrite / schema 变更：`rust/lancedb/src/remote/table.rs:969-970`, `1006-1007`, `1711-1713`, `1765-1767`, `1792-1794`

### 9.4 `location` 缓存更简单，但同样有价值

`RemoteTable` 还持有：

- `location: RwLock<Option<String>>`
- 位置：`rust/lancedb/src/remote/table.rs:213-214`

`uri()` 的策略非常直接：

1. 如果本地已有 location，就直接返回
2. 否则调用 `describe()`
3. 成功后写回缓存

位置：

- `rust/lancedb/src/remote/table.rs:1932-1953`

这层缓存很简单，但对频繁读取表 URI 的路径可以省掉重复请求。

## 10 `BackgroundCache` 是这套机制里的通用基础设施

`BackgroundCache` 是 LanceDB 自己实现的通用缓存原语，至少被两处关键逻辑复用：

1. `DatasetConsistencyWrapper::Eventual`
2. `RemoteTable.schema_cache`

代码位置：

- `rust/lancedb/src/utils/background_cache.rs`

它不是简单 memoize，而是显式建模了三种状态：

1. `Empty`
2. `Current`
3. `Refreshing`

位置：

- `rust/lancedb/src/utils/background_cache.rs:64-74`

它解决的两个关键问题是：

1. 多个并发请求不要重复触发同一个 fetch
2. invalidate 之后，旧的后台刷新结果不要回写脏数据

对应实现证据：

- 共享 in-flight future：`rust/lancedb/src/utils/background_cache.rs:137-165`
- generation 防止失效后旧值回写：`rust/lancedb/src/utils/background_cache.rs:176-184`

## 11 一个容易忽视但很重要的点：`storage_options=None` 和 `Some({})` 不等价

`ListingDatabase::open_table()` 里有一段非常关键的注释：

- `storage_options=None` 会使用 connection 的 session store registry
- `storage_options=Some({})` 会创建一个新的 connection
- 这个 connection 会在 Python GC table 对象时从缓存里丢掉

位置：

- `rust/lancedb/src/database/listing.rs:1016-1022`

这说明了两件事：

1. session 里的 store registry 不只是“配置对象”，它还实际影响复用。
2. “空字典”和“不传”在缓存复用语义上并不等价。

对使用者的实际建议很直接：

- 如果不需要覆盖 storage options，就不要显式传空字典。

## 12 语言绑定层暴露了哪些缓存能力

### 12.1 Python / Node

Python 和 Node 都明确暴露了：

1. `Session`
2. 自定义 cache size
3. `size_bytes`
4. `approx_num_items`
5. `connect(..., session=...)`

而且有对应 smoke test 验证 cache 在操作后增长：

- Python：`python/python/tests/test_session.py:7-38`
- Node：`nodejs/__test__/session.test.ts:14-44`

### 12.2 Java

当前仓库 `java/` 目录中，没有发现与 `Session`、`index_cache_size`、`metadata_cache` 直接对应的公开接口痕迹。

这只能说明：

- 以当前代码树为准，Java 侧没有像 Python / Node 这样显式暴露 session-level cache configuration。

## 13 LanceDB Enterprise 服务端缓存机制

这一节基于截至 2026-04-23 的 LanceDB 官方 Enterprise 文档整理。需要先明确边界：Enterprise 服务端实现不在当前 OSS 仓库中，所以这里不能像前面分析 `RemoteTable.schema_cache` 那样直接用当前代码证明具体字段和淘汰实现。这里记录的是公开文档描述出来的架构语义。

### 13.1 Enterprise 缓存解决的核心问题

LanceDB Enterprise 仍然采用计算和存储分离：

1. durable table data、manifest、index artifacts 保存在对象存储中。
2. query-serving 和 background workers 作为可替换、可横向扩展的计算层运行。
3. cache 只是加速层，不是 source of truth。

它要解决的问题是：在 S3 / GCS / Azure Blob 这类对象存储上，冷数据访问会受到网络和对象存储 round trip 影响。如果每次查询都直接回源对象存储，原始向量列、metadata 列、过滤列或 refine 阶段需要的数据都可能成为延迟瓶颈。

Enterprise 的做法是把热点 table data 缓存在靠近读执行的位置。

### 13.2 Enterprise 读路径里的缓存位置

官方架构文档把读路径拆成几层：

1. client 请求进入 Query Node。
2. Query Node 校验请求、解析表、生成执行计划。
3. 对读密集型查询，Query Node 可以把读执行交给 Plan Executor。
4. Plan Executor 从对象存储读取需要的 table data，并在有帮助时使用 cache-backed reads。
5. 结果回到 Query Node，再返回给 client。

因此，Enterprise 的数据缓存不在 SDK 进程里，也不在用户应用所在机器上，而是在服务端 Plan Executor fleet 附近。

### 13.3 Plan Executor 缓存什么，不缓存什么

Enterprise FAQ 里有一个很关键的边界：

> The plan executor caches the table data, not the table indices.

这意味着：

1. Plan Executor cache 的对象是 table data。
2. index artifacts 仍然是 Lance table 的一部分，durable 存放在对象存储中。
3. 不要把 Enterprise 的 Plan Executor cache 理解成“把索引全部常驻内存”。
4. 也不要把它和 OSS SDK 里的 `Session` index cache 混为一谈。

结合前面关于查询的讨论，可以这样理解：如果查询需要读取候选行的原始向量、返回列、过滤列或其他 table data，Enterprise 的服务端缓存可以让这些读取从本地 NVMe 命中，而不是每次都回源对象存储。

### 13.4 Enterprise 缓存的组织方式

官方对比文档描述了几个关键点：

1. Enterprise 使用 NVMe SSD 作为 hybrid cache。
2. 首次读取会填充缓存，后续读取可以从本地磁盘返回。
3. 缓存跨 fleet 分片，文档描述为使用 consistent hashing。
4. 热点向量或表数据会留在本地 NVMe，直到按 LRU 一类策略老化淘汰。
5. cache miss 会回源对象存储，并把读取到的数据填充到本地缓存 shard。
6. 缓存命中可以降低对象存储访问延迟和 egress 成本。

这个机制和 OSS 的区别不只是“缓存介质不同”，而是部署模型不同：OSS 是应用进程直接读对象存储，Enterprise 是远端服务集群在读路径中集中处理 cache-backed reads。

### 13.5 和 OSS 缓存机制的差异

可以用下面这张表建立边界：

| 维度 | LanceDB OSS / native | LanceDB Enterprise |
| --- | --- | --- |
| 运行模式 | 嵌入式，运行在应用进程内 | 远端分布式集群 |
| durable 数据位置 | 本地文件系统或对象存储 | 对象存储 |
| 数据读取方 | 应用进程内的 LanceDB / Lance 执行路径 | 服务端 Query Node / Plan Executor |
| 主要可见缓存 | `Session` index cache、metadata cache、dataset 视图缓存 | Plan Executor 的分布式 NVMe table data cache |
| 缓存共享范围 | 通常受单进程 / 单 session 限制 | 服务端 fleet 内按分片和副本复用 |
| 缓存的数据对象 | 主要是索引、元数据、对象存储上下文、dataset 句柄 | table data，不是 table indices |
| 运维模型 | 用户负责进程、资源和 `optimize()` 节奏 | 服务端负责缓存、调度、后台 indexing / compaction |

官方 Enterprise 对比文档中说 “OSS Cache: None”，这个说法的语境是服务端分布式 table data cache：OSS 没有 Enterprise 那种跨 fleet 的 NVMe 热点数据缓存。它并不否认当前 SDK / Lance 里存在 session-level index / metadata cache。

### 13.6 对向量检索的影响

Enterprise 的 table data cache 对下列场景最有价值：

1. 查询返回原始 `vector` 列或其他大列。
2. `refine_factor` 触发读取完整未压缩向量做精排。
3. metadata filtering 需要读取过滤列或候选行数据。
4. 同一批热点数据被高频重复访问。
5. 多个应用副本或用户共享同一个 Enterprise 查询集群。

如果查询只命中索引数据并且只返回很少的标量列，那么对象存储 table data 读取压力本来就较小，Enterprise 缓存的收益会相对低一些。反过来，如果查询需要频繁读取候选行的原始向量或大字段，Enterprise 的 NVMe cache 就是降低尾延迟和对象存储费用的关键。

### 13.7 官方参考

当前总结主要参考这些官方文档：

1. LanceDB Enterprise vs OSS: https://docs.lancedb.com/enterprise/features
2. Enterprise Architecture: https://docs.lancedb.com/enterprise/architecture
3. Enterprise FAQ: https://docs.lancedb.com/faq/faq-enterprise
4. Enterprise Performance Characteristics: https://docs.lancedb.com/enterprise/performance

## 14 测试覆盖能说明什么

当前仓库里，缓存机制并不是完全没有测试。

主要能看到三类：

1. `DatasetConsistencyWrapper` 的 Rust 单测
   - lazy / strong / eventual 一致性行为
   - 参考：`rust/lancedb/src/table/dataset.rs:430+`
2. `RemoteTable.schema_cache` 的 Rust 单测
   - TTL、错误失效、操作失效、checkout 失效
   - 参考：`rust/lancedb/src/remote/table.rs:4254+`
3. Python / Node 对 `Session` 的 smoke test
   - 参考：`python/python/tests/test_session.py`
   - 参考：`nodejs/__test__/session.test.ts`

相对薄弱的地方在于：

1. 没看到 Rust 侧专门验证 `Session` 配置边界的单测。
2. `lance::session::Session` 的真实内部实现不在本仓库，所以 LanceDB 这一层无法完全证明更底层的淘汰和容量控制细节。
3. Enterprise 服务端 Plan Executor / NVMe cache 不在当前 OSS 仓库中，所以本仓库测试不能覆盖这层服务端缓存机制。

## 15 最后总结：怎样用一句话理解 LanceDB 缓存机制

最适合的总结不是“LanceDB 有一个缓存层”，而是：

> LanceDB 在本地路径缓存底层资源和当前表视图，在 Remote 客户端路径缓存远端表对象与元信息，在 Enterprise 服务端路径缓存热点 table data；不同缓存对象分别服务于资源复用、一致性折中、减少远端元数据请求和降低对象存储读延迟。

如果后续还要继续深入，最值得继续看的顺序应该是：

1. `Session` 的底层实现，确认 index / metadata cache 的内部结构
2. `DatasetConsistencyWrapper` 如何与表读写交互
3. `RemoteTable.schema_cache` 的失效策略是否还需要更细粒度优化
4. Enterprise 部署侧 Plan Executor cache 的容量、分片、副本和监控指标配置
