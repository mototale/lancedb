# LanceDB 架构概览

本文档基于当前仓库的实现，对 LanceDB 的整体架构做一份面向贡献者的中文梳理。

这不是一份正式的兼容性承诺文档，而是用来帮助理解代码组织方式、核心抽象，以及本地与远端两条执行路径的工作方式。尤其是 namespace 相关能力和部分 server-side pushdown 能力，目前仍在持续演进中。

## 1 整体概览

从仓库实现来看，LanceDB 的核心思路并不是提供一个单一的数据库内核，而是：

- 以 Rust core 作为主要实现中心
- 以统一的 `Connection -> Database -> Table` 抽象对外提供接口
- 同时支持本地嵌入式使用和远端 Cloud / Enterprise 使用
- 底层数据存储建立在 Lance 数据集之上
- 对外提供 Python、TypeScript / Node.js、Rust，以及面向远端的 Java 访问方式

可以先看下面这张简化结构图：

```text
Python / Node.js / Rust SDK      Java REST Client
              |                         |
              v                         v
     Rust 核心实现 (`rust/lancedb`)    远端 API
            |
            +-- Connection
            |     |
            |     +-- ListingDatabase         本地目录 / 对象存储
            |     +-- LanceNamespaceDatabase  基于 namespace 的元数据层
            |     +-- RemoteDatabase          Cloud / Enterprise 远端客户端
            |
            +-- Table
                  |
                  +-- NativeTable  直接操作 Lance dataset
                  +-- RemoteTable  通过 REST 调远端服务
```

如果用一句话概括：

可以把 LanceDB 理解为一层构建在 Lance 之上的数据库抽象：Rust core 统一封装本地表和远端表两类后端；Python 与 Node.js / TypeScript 主要通过语言绑定复用这套能力，Rust 直接使用 core API，而 Java 当前更偏向面向 Enterprise / REST 的远端客户端。

这里需要特别区分“OSS 代码仓库”和“Enterprise 运行形态”。当前仓库既包含本地嵌入式执行路径，也包含连接 Cloud / Enterprise 的远端客户端路径。连接 Enterprise 时，Python / Node.js / Rust SDK 仍然复用这个仓库里的 `Connection`、`RemoteDatabase`、`RemoteTable`、Arrow 编解码和请求封装能力，但真正的查询执行、索引构建、compaction、缓存和对象存储读写由 Enterprise 服务端集群完成。也就是说，在 Enterprise 形态下，OSS repo 中被使用的是客户端库 / 协议适配层，而不是本地 embedded 引擎那一半能力。

### 1.1 运行时架构图

如果从“运行时有哪些组件参与请求处理”的角度来看，可以用下面这张图理解 LanceDB：

```text
+--------------------------- 应用与 SDK 层 ---------------------------+
| Python / Node.js / Rust 应用                                        |
|                                                                     |
|  Python SDK        Node.js SDK         Rust API        Java Client   |
|      |                 |                  |                |         |
|      +-------- 绑定 / SDK 封装 ----------+                |         |
+---------------------------|---------------------------------+---------+
                            |
                            v
+----------------------- LanceDB 统一入口层 ----------------------------+
| Connection                                                          |
|   - 根据 URI、配置和 feature 选择后端                                 |
|   - 暴露 create/open/list/clone 等数据库级入口                        |
+---------------------------|------------------------------------------+
                            |
                            v
+----------------------- 数据库抽象层 ---------------------------------+
| Database                                                            |
|   +-- ListingDatabase         本地目录 / 对象存储                    |
|   +-- LanceNamespaceDatabase  namespace 元数据与可下推能力           |
|   +-- RemoteDatabase          Cloud / Enterprise 远端访问            |
+---------------------------|------------------------------------------+
                            |
                            v
+------------------------- 表抽象层 -----------------------------------+
| Table -> BaseTable                                                  |
|   +-- NativeTable   直接操作 Lance dataset                           |
|   +-- RemoteTable   代表远端表资源                                   |
+----------------------+-------------------------------+---------------+
                       |                               |
                       | 本地 / OSS 路径                | 远端路径
                       v                               v
+----------------------+------------------+   +--------+----------------+
| Native 查询与存储执行层                  |   | Remote 执行层           |
|  - Query Planner                         |   |  - REST client          |
|  - Lance scanner                         |   |  - HTTP / Arrow / JSON  |
|  - DataFusion ExecutionPlan              |   |  - Cloud API            |
|  - Local index / version / manifest      |   +--------+----------------+
+----------------------+------------------+            |
                       |                               v
                       v                    +----------+-----------------+
+----------------------+------------------+ | LanceDB Cloud / Enterprise |
| Lance dataset / Object Store / FS       | +----------------------------+
|  - 本地目录                              |
|  - S3 / GCS / OSS 等对象存储             |
+-----------------------------------------+
```

上图主要描述 Python、Node.js 和 Rust 这几条主要运行时路径。Java 在当前仓库里更接近远端客户端，主要面向 Cloud / Enterprise，并不完全复用同一套本地嵌入式 Rust 路径。

### 1.2 运行时组件说明

从运行时视角看，LanceDB 可以拆成以下几类组件：

- 应用与 SDK 层
  这是用户直接调用的入口，包括 Python、Node.js / TypeScript、Rust，以及当前主要用于远端访问的 Java client。前面三者是当前官方文档中列出的主要 SDK 形态，Java 则更偏向 Enterprise REST API SDK。它们共同负责把用户的建表、写入、检索和过滤等操作组织为各语言的调用方式。

- 绑定与封装层
  Python 通过 PyO3，Node.js 通过 napi-rs，把 Rust core 的对象暴露给上层语言。这一层主要负责语言边界上的类型转换、异步封装和 Arrow 数据编解码，而不是重复实现数据库能力。

- `Connection`
  `Connection` 是 LanceDB 的统一入口。它会根据 URI 和配置判断当前请求应该走本地路径、namespace 路径还是远端路径，并把数据库级操作转发给具体的 `Database` 实现。

- `Database`
  `Database` 负责管理表及其元数据，而不是直接执行具体查询。它处理 namespace、表名列举、建表、开表、删表、重命名、克隆等生命周期问题。

- `ListingDatabase`
  这是本地和对象存储场景下的主要数据库实现。它负责连接目录或 object store，管理 storage options、session 和缓存，并把表打开为 `NativeTable`。

- `LanceNamespaceDatabase`
  这是 namespace 驱动的数据库实现。它把很多元数据管理工作委托给 `LanceNamespace`，同时也是 server-side pushdown 和托管版本控制等能力的重要边界。

- `RemoteDatabase`
  这是 `db://...` 场景下的数据库实现。本质上它是远端数据库访问网关，负责构造 REST 客户端、注入认证与 region 配置，并缓存已打开的远端表对象。

- `Table` / `BaseTable`
  这是 LanceDB 的统一表抽象。对上层 SDK 来说，表 API 尽量保持一致；对内部实现来说，真正执行能力由 `NativeTable` 或 `RemoteTable` 提供。

- `NativeTable`
  这是本地表实现。它直接持有 Lance dataset 相关状态，并负责本地查询、写入、索引、版本管理、schema 演进等能力。

- `RemoteTable`
  这是远端表实现。它不直接访问底层 dataset，而是把表级操作翻译成对 Cloud / Enterprise 服务的远端请求。

- 查询执行组件
  在 native 路径下，查询会经过 query builder、query planner、Lance scanner 和 DataFusion `ExecutionPlan`。这一层负责向量检索、全文检索、过滤、投影、分页以及部分混合检索逻辑。

- 存储与数据组件
  Lance dataset 是 LanceDB 在本地路径下的核心数据承载体。更底层可以落在本地文件系统或对象存储之上。跨语言边界以及不少内部边界上的数据交换，则主要依赖 Arrow 和 Arrow IPC。

- 远端服务组件
  在 Cloud / Enterprise 路径下，真正的数据管理和查询执行由服务端完成。客户端侧主要负责协议封装、鉴权、请求发送和结果解析。
  对 Python / Node.js / Rust 来说，这个客户端侧仍然复用当前 OSS 仓库中的 Rust core 远端路径；但它不直接打开 Lance dataset，也不在应用进程内执行查询计划。

### 1.3 组件之间的关系

从依赖关系上看，可以把这些组件进一步归纳为三层：

- 接入层
  包括 Python / Node.js / Rust SDK，以及 Java client。前者主要复用 Rust core 暴露的能力，Java 当前则更多承担远端访问入口。

- 核心控制层
  包括 `Connection`、`Database`、`Table`、`BaseTable`。这一层决定请求被路由到哪里，以及表能力如何被统一抽象出来。

- 执行与存储层
  对 native 路径来说，这一层是 Lance dataset、Lance scanner、DataFusion、object store 和本地索引；对 remote 路径来说，这一层则是 REST client 对接的 LanceDB Cloud / Enterprise 服务。

### 1.4 两种典型运行时形态

在运行时，最常见的是下面两种形态：

- 嵌入式 / 本地形态
  应用进程内直接装载 LanceDB Rust core，通过 `ListingDatabase -> NativeTable` 访问 Lance dataset。这种模式更接近 SQLite，一般没有独立的数据库服务进程。

- 远端服务形态
  应用侧通过 `RemoteDatabase -> RemoteTable` 把请求发给 LanceDB Cloud / Enterprise。此时客户端保留统一 API，但真实执行发生在服务端。

## 2 仓库分层

仓库中和架构最相关的目录主要有：

- `rust/lancedb`：Rust 核心实现
- `python`：Python 绑定与高层 Python API
- `nodejs`：Node.js / TypeScript 绑定与高层 TS API
- `java`：Java 远端客户端
- `docs`：文档与 SDK 参考站配置

其中，Rust core 是最重要的实现中心。几个关键入口文件如下：

- `rust/lancedb/src/connection.rs`：连接建立、后端分发
- `rust/lancedb/src/database.rs`：`Database` trait 与数据库级请求定义
- `rust/lancedb/src/database/listing.rs`：本地 / 对象存储数据库实现
- `rust/lancedb/src/database/namespace.rs`：namespace-backed 数据库实现
- `rust/lancedb/src/table.rs`：公开 `Table`、`BaseTable`、`NativeTable`
- `rust/lancedb/src/remote/db.rs`：远端数据库客户端
- `rust/lancedb/src/remote/table.rs`：远端表客户端
- `rust/lancedb/src/query.rs`：查询构造器与查询请求类型
- `rust/lancedb/src/table/query.rs`：native 查询规划与执行
- `rust/lancedb/src/table/datafusion.rs`：DataFusion 集成层

## 3 核心抽象

### 3.1 Connection

`Connection` 是用户访问 LanceDB 的统一入口。

它内部主要包含两类状态：

- 一个 `Arc<dyn Database>`，表示当前连接所使用的底层数据库实现
- 一个 embedding registry，用于嵌入模型和相关能力

从职责上看，`Connection` 负责：

- 列出表
- 创建表
- 打开表
- 克隆表
- 发起部分数据库级和 namespace 级操作

也就是说，`Connection` 并不直接关心表的底层实现细节。它更像一个统一入口，负责把请求转发给具体的 `Database` 实现。

### 3.2 Database

`Database` 是 LanceDB 中最核心的抽象之一。

它不直接负责具体查询的执行，而是负责表及其元数据的管理，例如：

- namespace 的创建、查询、删除
- 表名列举与分页
- 创建表、打开表、删除表、重命名表
- 克隆表
- 返回等价的 namespace client

因此，可以把 `Database` 理解为：

“某一种后端环境下，如何管理一组表及其元数据”

当前代码中最重要的三个 `Database` 实现是：

- `ListingDatabase`
- `LanceNamespaceDatabase`
- `RemoteDatabase`

### 3.3 Table 与 BaseTable

对用户可见的 `Table` 是统一表对象，但它内部实际上包着 `Arc<dyn BaseTable>`。

这意味着：

- 上层 SDK 尽量只暴露一套表 API
- 底层可以是 native table，也可以是 remote table
- 本地和远端在很多能力上共享同一套使用模型

`BaseTable` 定义了所有“表状对象”应具备的共同行为，包括：

- schema 获取
- 行数统计
- 查询规划与查询执行
- 写入、删除、更新、merge
- 索引创建与管理
- schema 演进
- optimize
- version / checkout / tags

这层抽象非常关键，因为它是 LanceDB 统一 API 的基础。

## 4 连接分发与后端选择

LanceDB 会根据连接 URI 决定采用哪条后端路径。

### 4.1 本地与 OSS 路径

当 URI 不是 `db://...` 时，连接通常走 `ListingDatabase`。

这类 URI 主要包括：

- 本地目录
- `s3://...`、`gs://...` 一类对象存储位置
- 一些本地开发或内存场景

在这条路径下，LanceDB 的行为更接近嵌入式数据库：

- 当前进程直接打开并管理 Lance dataset
- 数据读写通过本地存储或对象存储完成
- 查询通常在本地进程内执行

### 4.2 远端路径

当 URI 是 `db://...` 时，连接会走 `RemoteDatabase`。

在这条路径下：

- Rust core 不再直接操作本地 dataset
- 它会构造 REST 客户端
- API key、region、host override、TLS 等配置在连接时注入
- 表与查询操作会被转换成对 LanceDB Cloud / Enterprise 的远端调用

因此，对用户来说 API 看起来依然一致，但背后的执行模型已经切换成了客户端 / 服务端模式。

这也是理解 Enterprise 形态时最重要的边界：LanceDB OSS 的 SDK / Rust core 在这里承担客户端作用。它负责把统一 API 映射成远端请求，维护远端表对象、schema / location 等客户端侧状态，并解析服务端返回的 Arrow / JSON 结果。它不负责在本地执行向量检索、读取对象存储中的 table data、维护 Enterprise 的 NVMe cache，或者调度服务端后台任务。

从调用路径上看，可以粗略理解为：

```text
应用代码
  -> Python / Node.js / Rust SDK
  -> OSS repo 中的 Connection / RemoteDatabase / RemoteTable
  -> REST / Arrow / JSON 请求
  -> LanceDB Enterprise 服务端
  -> Query Node / Plan Executor / Background workers
  -> 对象存储与服务端缓存
```

## 5 本地架构：ListingDatabase 与 NativeTable

### 5.1 ListingDatabase 的职责

`ListingDatabase` 是本地和对象存储场景下的主要数据库实现。

它负责：

- 解析 URI 与 object store
- 管理 storage options
- 管理 session 和缓存
- 打开和创建本地表
- 把目录型布局与 namespace 能力衔接起来

从概念上说，`ListingDatabase` 主要解决的问题是：

“给定一个目录或对象存储根路径，如何在这里发现、创建和打开 LanceDB 表”

### 5.2 NativeTable

`NativeTable` 是本地表的实际实现。

它内部维护的关键信息包括：

- 表名和 namespace
- 表 URI
- Lance dataset 的一致性包装
- 读一致性配置
- 可选的 namespace client
- 可选的 pushdown 操作集合

这一层最能体现 LanceDB 的“嵌入式”特征：

- 表数据是当前进程直接访问的
- 索引是本地 dataset 的索引
- 查询、写入、schema 变更等能力主要在本进程完成

### 5.3 读一致性模型

本地路径下，读一致性是可配置的。

当前代码里的模型包括：

- `Manual`：手动刷新
- `Eventual(Duration)`：在给定时间间隔后刷新
- `Strong`：每次读取都检查最新状态

这套配置主要针对 OSS / embedded 场景。Cloud 路径下的一致性行为不是通过同一个开关暴露出来的。

## 6 namespace 层：统一元数据与可下推能力

从当前实现看，namespace 不是一个边缘性的附属能力，而是 LanceDB 架构里越来越重要的一层。

### 6.1 LanceNamespaceDatabase

`LanceNamespaceDatabase` 是一个基于 `LanceNamespace` 的 `Database` 实现。

它负责把数据库级元数据管理委托给 namespace client，包括：

- namespace 的列举、创建、删除、描述
- 表的列举、声明、打开
- 托管版本控制
- 某些操作的服务端下推

### 6.2 为什么这层重要

namespace 层的重要性在于它正在缩小本地与远端两种部署模式的差距。

它提供了一个更统一的元数据边界，使得：

- 本地目录型存储也可以通过 namespace 风格接口暴露出来
- 远端服务也能复用同一套元数据模型
- 某些原本只能本地执行的操作，未来可以切换为 server-side pushdown

这也是为什么 `ListingDatabase` 自己内部也会连接一个 namespace-backed 数据库，而不是把 namespace 完全视为远端独有能力。

## 7 远端架构：RemoteDatabase 与 RemoteTable

### 7.1 RemoteDatabase

`RemoteDatabase` 是 `db://...` 场景下的数据库实现。

它主要负责：

- 构造 REST 客户端
- 注入认证头和 region 信息
- 注入 TLS 和其他客户端配置
- 缓存已打开的 remote table
- 把数据库级操作翻译成远端 API 请求

这一层本质上就是远端数据库访问网关。

### 7.2 RemoteTable

`RemoteTable` 是远端表的实现。

它内部主要包含：

- REST client
- 表名和 namespace 标识
- 服务端版本信息
- version / location 缓存
- schema 缓存

与 `NativeTable` 不同，`RemoteTable` 不直接操作 Lance dataset，而是代表一个远端资源，通过 HTTP 与 LanceDB Cloud / Enterprise 通信。

这意味着同一套表 API，背后可能映射到两种完全不同的执行模式：

- native：直接本地执行
- remote：远端调用，由服务端执行

### 7.3 OSS 在 Enterprise 形态中的角色

在 Enterprise 形态中，不能把 LanceDB OSS 理解成“服务端数据库内核”。更准确的说法是：

- OSS repo 提供客户端 SDK、远端连接实现和统一 API 抽象。
- `RemoteDatabase` 是远端数据库访问网关。
- `RemoteTable` 是远端表资源代理。
- 表操作、查询、索引等能力会被翻译成 HTTP API 请求。
- Arrow / JSON 结果会在客户端侧解码并转换成各语言 SDK 的返回值。

它不承担这些服务端职责：

- 在应用进程内执行 Enterprise 查询计划。
- 直接扫描 Enterprise 对象存储中的 table data。
- 在应用进程内构建或维护 Enterprise 索引。
- 管理 Enterprise 的 Plan Executor、NVMe 数据缓存或后台 compaction / indexing worker。

这也是为什么同一个 LanceDB SDK 既可以作为 embedded OSS 数据库使用，也可以作为 Enterprise 客户端使用。两种模式共享上层 API 和部分客户端实现，但执行与存储职责完全不同。

## 8 查询架构

LanceDB 的查询能力包括：

- 向量检索
- 全文检索
- 过滤
- 投影
- SQL 风格表达式
- 混合检索与 rerank

在架构上，可以把它拆成两层：

- 面向用户的查询构造层
- 面向后端的执行层

### 8.1 查询构造层

查询构造层主要在 `rust/lancedb/src/query.rs` 中。

这层会把用户操作组织成一组查询请求对象，例如：

- `QueryRequest`
- `VectorQueryRequest`
- `AnyQuery`

这些结构表达的是用户想做什么，而不是后端具体如何执行。

### 8.2 Native 查询执行

对于 `NativeTable`，查询大致沿着下面这条路径执行：

```text
Query builder
  -> AnyQuery / VectorQueryRequest
  -> native query planner (`table/query.rs`)
  -> Lance scanner + DataFusion ExecutionPlan
  -> Arrow RecordBatch stream
```

这条路径里主要完成的工作包括：

- 推断或确定向量列
- 配置 nearest-neighbor 查询
- 应用过滤、limit、offset、投影
- 处理多向量查询
- 生成 DataFusion `ExecutionPlan`
- 最终执行并返回 Arrow `RecordBatch` 流

因此，native 查询并不是简单地“扫文件”，而是由 Lance scanner 和 DataFusion 执行计划协同完成。

### 8.3 namespace pushdown

对于 `NativeTable`，部分操作在相关配置开启后可以下推到 namespace server。

当前代码中显式支持的下推操作包括：

- `QueryTable`
- `CreateTable`

这意味着同样的上层 API，在某些配置下可以从本地执行切换为 server-side 执行，而不需要改动 SDK 接口形态。

### 8.4 DataFusion 集成

`table/datafusion.rs` 提供了 LanceDB 和 DataFusion 的集成适配层。

这层的作用是把 LanceDB 表适配成 DataFusion 的 `TableProvider`，从而让 LanceDB：

- 参与 SQL 规划与执行
- 复用 DataFusion 的表达式与计划机制
- 支持更复杂的 filter / projection / UDTF 逻辑

所以，LanceDB 当前的查询执行既不是“只依赖 Lance”，也不是“只依赖 DataFusion”，而是两者协同工作：

- Lance 提供底层存储、扫描和向量能力
- DataFusion 提供执行计划、表达式和 SQL 集成能力

## 9 存储与数据表示

### 9.1 物理存储层

LanceDB 的底层物理数据模型建立在 Lance 之上，而不是自研一套完全独立的存储引擎。

这一点在代码里体现得很明显：

- native 表直接打开 Lance dataset
- 向量列使用 Arrow 的定长 list 类型表示
- schema 和批数据都基于 Arrow 表达
- 对象存储能力来自 Lance 及其相关抽象

### 9.2 数据交换层

在跨语言绑定和部分模块边界上，Arrow IPC 是核心的数据交换格式。

典型场景包括：

- Node.js 通过 Arrow IPC buffer 与 Rust 交换 schema 和 batch
- Python 把 PyArrow 数据转换成 Rust 侧 Arrow reader
- 查询结果以 Arrow `RecordBatch` 流形式返回

这使得 LanceDB 在多语言之间仍然保持列式、批量的数据通路。

## 10 语言绑定层

### 10.1 Python

Python 侧大致分为两层：

- `python/src`：PyO3 扩展层，直接暴露 Rust 对象
- `python/python/lancedb`：更 Pythonic 的高层 API

Rust 扩展层会直接暴露诸如：

- `Connection`
- `Table`
- query 相关对象

高层 Python API 则进一步包装成：

- `AsyncTable`
- `LanceTable`
- `RemoteTable`

其中：

- `AsyncTable` 更接近 Rust 异步对象
- `LanceTable` 是面向同步 Python 用户的包装
- 同步 API 本质上通常通过事件循环辅助器去调用异步接口

因此，Python 侧主要承担的是语言习惯上的适配，而不是重复实现数据库逻辑。

### 10.2 Node.js / TypeScript

Node.js 侧也采用类似模式：

- `nodejs/src`：napi-rs 绑定层
- `nodejs/lancedb`：高层 TypeScript API

绑定层会暴露：

- `Connection`
- `Table`
- 各类查询和表操作

TypeScript 再组织成：

- 抽象 `Table`
- 具体包装类如 `LocalTable`
- 基于 Arrow 的 schema / data 编解码逻辑

它与 Rust core 的边界也大量使用 Arrow IPC。

### 10.3 Java

从仓库内容看，Java 侧当前定位更接近远端客户端，主要面向：

- LanceDB Cloud
- LanceDB Enterprise

它并不像 Python 和 Node.js 那样，提供一套完整的本地嵌入式绑定层。

## 11 典型请求路径

### 11.1 本地查询路径

```text
应用代码
  -> SDK Table / Query API
  -> Rust `Table`
  -> `NativeTable`
  -> `table/query.rs` 中的查询规划
  -> Lance scanner + DataFusion plan
  -> Arrow RecordBatch stream
  -> SDK 结果转换
```

### 11.2 远端查询路径

```text
应用代码
  -> SDK Table / Query API
  -> Rust `Table`
  -> `RemoteTable`
  -> REST client
  -> LanceDB Cloud / Enterprise
  -> HTTP response
  -> Arrow / JSON 解码
  -> SDK 结果转换
```

### 11.3 本地建表路径

```text
应用代码
  -> Connection.create_table(...)
  -> Database.create_table(...)
  -> ListingDatabase / LanceNamespaceDatabase
  -> NativeTable
  -> Lance dataset 写入路径
```

## 12 架构边界与代码职责

当前代码的职责边界整体比较清晰，大致可以这样理解：

- `Connection`：统一入口与后端分发
- `Database`：表生命周期与元数据管理
- `Table`：统一表对象
- `BaseTable`：native / remote 共同行为契约
- `NativeTable`：本地和对象存储路径下的真实表实现
- `RemoteTable`：远端服务表实现
- `query.rs`：查询请求构造
- `table/query.rs`：native 查询规划与执行
- `table/datafusion.rs`：SQL 与执行计划集成
- Python / Node.js：Rust core 语言绑定与 SDK 封装
- Java：远端 REST client 封装

这也决定了很多新功能的接入路径通常比较固定：

1. 在 Rust `Table` 与 `BaseTable` 中补齐能力
2. 在 `NativeTable` 中实现
3. 如果远端支持，在 `RemoteTable` 中实现
4. 暴露到 Python / Node.js 绑定
5. 如果远端 SDK 需要，再评估 Java / REST 侧是否补齐
6. 添加测试和文档

## 13 当前演进方向

### 13.1 namespace 正在变成更核心的抽象

从代码演进趋势看，namespace 不只是元数据附加层，而是在逐渐成为本地与远端之间更统一的协调边界，尤其体现在：

- 多 namespace 管理
- 托管版本控制
- server-side pushdown
- 本地目录布局与 namespace 模型的衔接

### 13.2 统一 API，分化执行

LanceDB 对外暴露的是统一 API，但内部执行路径并不完全一致。

这是一种明确的架构取舍：

- 优点是 SDK 体验统一
- 成本是内部需要持续维护 native / remote 两套能力对齐

因此在新增能力时，往往需要先明确一个问题：

这个功能是：

- 仅 native 支持
- 仅 remote 支持
- 还是两端都支持

### 13.3 Rust core 仍然是能力源头

从仓库结构和实现方式看，Rust core 仍然是 LanceDB 的真实能力中心。

Python 和 Node.js 主要是绑定和封装，而不是各自维护一套独立的数据库实现。

## 14 总结

理解 LanceDB 的一个比较有效的方式，是按层次去看：

- Lance 提供底层数据集、存储和扫描能力
- Rust core 提供统一的数据库、表和查询抽象
- namespace 层逐渐承担统一元数据与协调职责
- `NativeTable` 与 `RemoteTable` 分别代表本地和远端两种后端
- Python / Node.js / Rust 在上层暴露主要开发接口
- Java 当前更偏向面向 Cloud / Enterprise 的远端访问接口

因此，LanceDB 同时支持两种重要使用模式：

- 类似 SQLite 的嵌入式、本地进程内使用
- 面向 Cloud / Enterprise 的远端服务式使用

而用户在多数情况下仍然可以通过一套相对统一的表 API 来使用这些能力。
