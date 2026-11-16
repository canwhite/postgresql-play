# 05 · 相关工具：什么时候该换别的家伙事

**学习目标**

读完之后，你应当能够：

1. 用**一个维度**（而不是"功能多少"）解释 PostgreSQL 与 MySQL、MongoDB 的根本差异。
2. 对每个外部组件说出：**它解决什么问题、和 PostgreSQL 重叠在哪、什么时候该引入它**。
3. 拿到一个真实需求，能用 5.8 的选择规则给出结论，并说清楚理由落在哪个证据上。
4. 分清哪些比较是**有据可查的**，哪些是源文档的**主张但未经外部核实**。

前置：已读 [03 · 核心能力](03-core-functionality.md)。

---

## 5.1 选型的三个维度

源文档给出了本章的立论方式：

> 这三者的差异不在"功能多少"，而在**它们各自把什么放在第一位**。
> — [Source-derived · S-16](07-verification-record.md#s-16)

这句话把选型从"功能清单打勾"换成了"优先级对齐"。本课程把它拆成三个可操作的维度：

| 维度 | 提问方式 | 后果 |
| :--- | :--- | :--- |
| **数据模型优先级** | 你的数据天生是"文档"还是"关系"？ | 决定要不要 MongoDB |
| **写入形态优先级** | 这张表是"改一次"还是"改一万次"？ | 决定能不能用一张表当队列 |
| **检索深度优先级** | 需要的是"能查到"还是"搜得准、排得对"？ | 决定要不要 Elasticsearch |

这三问分别对应源文档的三条依据：写入形态 → [S-12](07-verification-record.md#s-12)（不要把它当消息队列）、检索深度 → [S-15](07-verification-record.md#s-15)（全文检索有能力上限）、数据模型 → [S-20](07-verification-record.md#s-20)（MongoDB 的真正优势收缩到数据模型天然文档化的场景）。OLAP 边界 [S-13](07-verification-record.md#s-13) 属于分析型负载，不在本节这三问之内。

---

## 5.2 PostgreSQL vs. MySQL

源文档的核心判断：

> 一句话概括：**MySQL 是一辆让你快速上路的小轿车；PostgreSQL 是一辆能处理复杂地形的越野车，但你需要会开、会保养**。
> — [Source-derived · S-18](07-verification-record.md#s-18)

逐行拆开源文档的对比表（[S-17](07-verification-record.md#s-17)），并为每条标注证据强度：

| 维度 | 源文档的 PostgreSQL 侧 | 源文档的 MySQL 侧 | 证据强度 |
| :--- | :--- | :--- | :--- |
| 设计哲学 | 功能完整、标准严格、可扩展 | 简单、快速、生态成熟 | 源文档主张（主观概括） |
| 复杂查询 | 窗口函数、递归 CTE、丰富类型，表达力强 | "8.0 后大幅补课，但深度仍有差距" | ✅ **已核实**——MySQL 8.0 的确新增了窗口函数与 CTE（[R-21](07-verification-record.md#r-21)、[R-23](07-verification-record.md#r-23)、[R-24](07-verification-record.md#r-24)） |
| 半结构化数据 | JSONB + GIN 索引，可索引查询内部键值 | JSON 支持较轻量，索引能力有限 | ✅ 部分核实——PostgreSQL 侧有官方依据（[R-04](07-verification-record.md#r-04)、[R-05](07-verification-record.md#r-05)）；MySQL 侧"较轻量"是概括性判断 |
| MVCC 实现 | 行级多版本 + VACUUM，读不阻塞写 | Undo Log 回滚段，长事务下 undo 膨胀是主要风险 | ✅ **已核实**——MySQL 官方文档确认 InnoDB 把旧版本存在回滚段，且"回滚段可能增长过大"（[R-22](07-verification-record.md#r-22)） |
| 典型负载 | 复杂分析、GIS、企业级系统、强一致性、扩展需求 | 高并发简单 OLTP、电商、社交、常规 CRUD | 源文档主张（经验判断） |
| 读写混合性能 | "2026 年基准中，读写混合 TPS 显著领先" | "纯读点查场景仍有优势" | ⚠️ **未核实**——见 [5.7 节](#57-三条没有证据支撑的对比) |
| 运维负担 | 更重：VACUUM、膨胀、索引策略需要主动管理 | 相对更轻，生态工具成熟 | ⚠️ 部分核实——PostgreSQL 侧有官方依据（autovacuum 需按阈值触发，[R-03](07-verification-record.md#r-03)）；MySQL 侧"更轻"是相对判断 |

### 5.2.1 MySQL 8.0 到底补了什么

"MySQL 8.0 后大幅补课"这句在官方文档里有确切出处：

> **Window functions.** MySQL now supports window functions that, for each row from a query, perform a calculation using rows related to this row.
>
> **Common table expressions.** MySQL now supports common table expressions, both nonrecursive and recursive.
> — [Research-derived · R-21](07-verification-record.md#r-21)（MySQL 8.0 Reference Manual, "1.3 What Is New in MySQL 8.0"）

同一份文档还列出了 8.0 的 JSON 增强：`->>` 操作符、`JSON_ARRAYAGG()` / `JSON_OBJECTAGG()` 聚合、`JSON_TABLE()`、部分原地更新等（[R-21](07-verification-record.md#r-21)）。

**所以"8.0 后大幅补课"这个判断是站得住的**——窗口函数和 CTE 这两块曾经确实是 MySQL 的硬伤。

⚠️ 但要注意"补课 ≠ 抹平差距"。MySQL 文档对递归 CTE 还规定了一个默认深度上限：

> By default, `cte_max_recursion_depth` has a value of 1000, causing the CTE to terminate when it recurses past 1000 levels.
> — [Research-derived · R-24](07-verification-record.md#r-24)

PostgreSQL 侧是否同样存在默认递归深度上限，本课程**尚未查证**（[R-10](07-verification-record.md#r-10) 只说明 `RECURSIVE` 的语义，未涉及深度限制），因此此处不下结论。

### 5.2.2 两套 MVCC 的失效模式不同

> **示例标签**：Research-derived（两侧机制依据 R-01/R-02 与 R-22；推论为 Illustrative）
>
> | | PostgreSQL | MySQL / InnoDB |
> | :--- | :--- | :--- |
> | 旧版本存哪 | 表本身（堆页里的死元组） | 回滚段（undo tablespace） |
> | 何时回收 | autovacuum 按阈值触发 | 长事务持有快照时无法丢弃 |
> | 典型失效 | 高频 `UPDATE` → 死元组堆积 → 表膨胀 | 长事务 → 回滚段增长 → 撑满 undo 表空间 |
>
> PostgreSQL 侧官方原文：
>
> > The space it occupies must then be reclaimed for reuse by new rows, to avoid unbounded growth of disk space requirements. This is done by running `VACUUM`.
> > — [R-02](07-verification-record.md#r-02)
>
> MySQL 侧官方原文：
>
> > It is recommend that you commit transactions regularly, including transactions that issue only consistent reads. Otherwise, InnoDB cannot discard data from the update undo logs, and the rollback segment may grow too big, filling up the undo tablespace in which it resides.
> > — [R-22](07-verification-record.md#r-22)
>
> **运维含义**：PostgreSQL 要盯着 autovacuum 是否跟得上；MySQL 要盯着有没有长事务。**同一个"MVCC 读不阻塞写"的能力，两边付账的方式不同。**

---

## 5.3 PostgreSQL vs. MongoDB

源文档的关键判断：

> 关键变化在于：**PostgreSQL 的 JSONB 已经覆盖了相当一部分过去只有 MongoDB 才能优雅处理的场景**，同时保留了关系完整性、Join 和真正的 ACID 事务。MongoDB 的真正优势收缩到了**数据模型天然文档化、且水平扩展需求从第一天就存在**的场景。
> — [Source-derived · S-20](07-verification-record.md#s-20)

### 5.3.1 MongoDB 官方怎么说"事务"

> In MongoDB, an operation on a single document is atomic. The MongoDB document model supports embedding related data within a single document, which reduces the need for multi-document transactions. When your application requires atomicity across multiple documents, use transactions.
>
> In most cases, a distributed transaction incurs a greater performance cost over single document writes, and the availability of distributed transactions should not be a replacement for effective schema design. For many scenarios, the denormalized data model (embedded documents and arrays) will continue to be optimal for your data and use cases.
> — [Research-derived · R-25](07-verification-record.md#r-25)

这段官方文档正好印证了源文档说的"事务是'后补的能力'"——MongoDB 官方的建议顺序是**先设计好文档模型，实在需要跨文档原子性才上事务**。

还有一条硬约束：

> Transactions are supported on replica sets and sharded clusters.
>
> To use transactions, the `featureCompatibilityVersion` for all members of the deployment must be at least: Replica Set → 4.0; Sharded Cluster → 4.2.
> — [Research-derived · R-25](07-verification-record.md#r-25)

**即：多文档事务要求副本集或分片集群**（对应源文档说的 "4.0+"，且副本集需 FCV ≥ 4.0、分片集群需 ≥ 4.2）。

### 5.3.2 分片：MongoDB 是内建的，PostgreSQL 要靠扩展

> Sharding is a method for distributing data across multiple machines. MongoDB uses sharding to support deployments with very large data sets and high throughput operations.
>
> A MongoDB sharded cluster consists of the following components: shard … Routing with mongos … config servers …
> — [Research-derived · R-26](07-verification-record.md#r-26)

PostgreSQL 侧：

> Citus gives you the Postgres you love plus horizontal scale-out with schema-based and row-based sharding. […] You can start with a single Citus node, then add nodes & rebalance shards when you need to grow.
> — [Research-derived · R-34](07-verification-record.md#r-34)

**这是两者在"水平扩展"上的真实差别**：MongoDB 的分片是一等公民、集群形态原生；PostgreSQL 要额外引入 Citus 这类扩展，运维面随之扩大。源文档说的"分片依赖 Citus 等扩展"是准确的（[S-19](07-verification-record.md#s-19)）。

### 5.3.3 对照表（含证据强度）

| 维度 | 源文档结论 | 证据强度 |
| :--- | :--- | :--- |
| 数据模型 | 表 + 预定义 Schema + SQL；JSONB 容纳半结构化 | ✅ 已核实（[R-04](07-verification-record.md#r-04)） |
| Schema 灵活性 | PostgreSQL 严格列 + JSONB 可控灵活；MongoDB 真灵活，代价是脏数据靠应用层兜底 | ⚠️ 概括性判断，双方均无逐字出处 |
| Join | PostgreSQL 多表 Join 是看家本领；MongoDB `$lookup` 昂贵笨拙，官方范式是反范式化冗余 | ⚠️ MongoDB 侧有间接支持——官方文档反复强调"嵌入文档可减少对多文档事务的需求"（[R-25](07-verification-record.md#r-25)） |
| 事务 | PostgreSQL 多行 ACID 是基石；MongoDB 4.0+ 支持多文档事务但官方明示有性能代价 | ⚠️ MongoDB 侧 ✅ 已核实（[R-25](07-verification-record.md#r-25)）；PostgreSQL 侧为源文档主张（[S-19](07-verification-record.md#s-19)） |
| 水平扩展 | PostgreSQL 流复制成熟、分片依赖 Citus；MongoDB 内置分片和副本集 | ⚠️ 部分核实——分片部分 ✅ 已核实（[R-26](07-verification-record.md#r-26)、[R-34](07-verification-record.md#r-34)）；"跟踪主库的备库 / 热备只读查询" ✅ 已核实（[R-39](07-verification-record.md#r-39)）；"PostgreSQL 流复制成熟"是源文档主张（[S-19](07-verification-record.md#s-19)），本课程未查证 |
| 典型数据 | 实体关系丰富、需要强一致性和复杂查询 vs 数据天然"自包含文档" | 源文档主张（经验判断） |

---

## 5.4 什么时候该引入 Redis 或消息中间件

源文档的判断：

> **❌ 不要把它当消息队列**
>
> 高频轮询、认领、更新任务状态，会产生海量死元组，让 autovacuum 彻底跟不上。
> — [Source-derived · S-12](07-verification-record.md#s-12)

**Redis 是什么**：

> Redis can be used as a database, cache, streaming engine, message broker, and more.
> — [Research-derived · R-28](07-verification-record.md#r-28)

**GitLab 的实际做法**：

> We use Sidekiq as our background job processor.
>
> GitLab stores Sidekiq jobs and their arguments in Redis.
> — [Research-derived · R-27](07-verification-record.md#r-27)

### 什么时候该引入

| 信号 | 说明 | 对应证据 |
| :--- | :--- | :--- |
| 一张表被高频 `UPDATE` 状态字段 | 每次 `UPDATE` 都留下一个死元组，频率越高死元组累积越快（具体量级需按自己的 SLO 评估） | [S-03](07-verification-record.md#s-03)、[R-02](07-verification-record.md#r-02) |
| `n_dead_tup` 持续增长且 autovacuum 追不上 | 观察证据 | [R-16](07-verification-record.md#r-16) |
| 需要"取出即消费"而不是"取出即查询" | 语义不同：队列是**投递**，表是**存储** | [R-28](07-verification-record.md#r-28) |
| 需要跨进程/跨机的可靠投递 | 消息中间件的职责 | [S-12](07-verification-record.md#s-12) |

> **示例标签**：Illustrative（课程建议；机制依据 [S-03](07-verification-record.md#s-03)、[R-02](07-verification-record.md#r-02)）

⚠️ **引入前先确认一件事**：如果你的"队列"其实**不需要立即消费**（比如可以每 5 分钟跑一次批处理），那么一张普通表 + 一个定时任务就够了，不必引入 Redis。队列中间件解决的是**投递语义**，不是"表不够快"。

### 什么时候**不**该引入

源文档同时提醒：

> **❌ 不要把它当作唯一的"万能锤"**
>
> 当系统从几千用户增长到百万级时，把缓存、队列、分析、向量搜索全部塞进一个 PostgreSQL 实例，会从务实的选择变成架构隐患。
> — [Source-derived · S-14](07-verification-record.md#s-14)

⚠️ 这里的"几千 → 百万级"是源文档作者的经验阈值，**没有可引用的一手来源**（见 [07 章 U-03](07-verification-record.md#u-03)）。请把它当作方向性提醒，而不是精确的容量水位线。

这条边界的另一面是：**不要为了"少一个组件"而把不该在数据库里的东西硬塞进去。** 队列、缓存、全文检索各有更合适的工具（[S-14](07-verification-record.md#s-14)、[S-15](07-verification-record.md#s-15)）。

---

## 5.5 什么时候该引入 Elasticsearch / OpenSearch

源文档的判断：

> **⚠️ 全文检索有能力上限**
>
> 内置的 `tsvector` 全文检索在中小规模够用，但面对大规模、高并发的搜索需求，专用搜索引擎（如 Elasticsearch/OpenSearch）在功能深度和性能上仍然更合适。
> — [Source-derived · S-15](07-verification-record.md#s-15)

两者的官方定位：

> Elasticsearch is an open source, distributed search and analytics engine built for speed, scale, and AI applications.
> — [Research-derived · R-29](07-verification-record.md#r-29)

> OpenSearch is open source that stays open: vendor-neutral, fully Apache 2.0 licensed from ingest to dashboard
>
> Index once, then search, retrieve, and analyze across everything you build and run.
> — [Research-derived · R-30](07-verification-record.md#r-30)

**内置全文检索的机制性局限**（来自 PostgreSQL 官方文档）：匹配完全发生在 `tsvector` 上，GIN 索引为每个词元存一条索引项（[R-07](07-verification-record.md#r-07)、[R-08](07-verification-record.md#r-08)），而 GIN 的写入成本本身也不低（[R-05](07-verification-record.md#r-05)）。文档量级上去后，索引体积与写入成本都会上升，"分词/同义词/拼写纠错/相关度调优"这类高级能力，也会推动你走向专用引擎。

| 你的搜索需求 | 建议 | 证据 |
| :--- | :--- | :--- |
| 站内搜索、**中小规模**（源文档口径，未给出具体文档数）、够用即可 | PostgreSQL `tsvector` + GIN | [S-15](07-verification-record.md#s-15)、[R-07](07-verification-record.md#r-07) |
| 大规模、高并发、需要同义词/纠错/复杂相关度 | Elasticsearch / OpenSearch | [S-15](07-verification-record.md#s-15)、[R-29](07-verification-record.md#r-29)、[R-30](07-verification-record.md#r-30) |
| 已经在用 PostgreSQL 之外的分析栈 | 统一到已有栈通常更省事 | 课程判断 |

---

## 5.6 扩展生态：留在一个数据库里，但不是白拿的

源文档的说法：

> 此外，数组、范围类型、自定义类型，以及通过扩展获得的 PostGIS（地理）、pgvector（向量）、TimescaleDB（时序）等能力，让它成为一个"可组装的数据库平台"。
> — [Source-derived · S-04](07-verification-record.md#s-04)

每个扩展的官方定位：

| 扩展 | 官方定位 | 证据 |
| :--- | :--- | :--- |
| PostGIS | "extends the capabilities of the PostgreSQL relational database by adding support for storing, indexing, and querying geospatial data" | [R-31](07-verification-record.md#r-31) |
| pgvector | "Open-source vector similarity search for Postgres"；支持 HNSW 与 IVFFlat 索引；默认精确最近邻，加索引后可切换为近似搜索 | [R-32](07-verification-record.md#r-32) |
| TimescaleDB（现属 Tiger Data） | 基于 Postgres 的时序平台：快速摄取、廉价存储多年数据、实时与历史数据一起查询；基于时间与键的分区；行存写入 + 列存分析 + 压缩；200+ 时间分析 SQL 函数 | [R-33](07-verification-record.md#r-33) |
| Citus | "the Postgres you love plus horizontal scale-out with schema-based and row-based sharding" | [R-34](07-verification-record.md#r-34) |

> **示例标签**：Illustrative（课程判断；各扩展的官方定位见上表）

**这个生态是 PostgreSQL "一库多用"底座最实的部分**，但每装一个扩展就多一条升级与运维路径。判断标准应该是：**这个能力是不是核心业务的一部分、且它与关系数据的关联足够紧？**

- 地理：位置和实体数据强关联 → PostGIS 很合适。
- 向量：向量与业务实体的关联往往也很紧（商品、文档）→ pgvector 合适。
- 时序：如果数据量极大且查询模式固定，专门的时序库可能有更优的压缩比与保留策略（TimescaleDB 自己也强调"行存 + 列存 + 压缩"）。

---

## 5.7 三条没有证据支撑的对比

本课程在查证过程中发现源文档表格里有两行**没有找到可核实的一手来源**，它们被保留为"源文档主张"，而不是被悄悄删掉或当作事实使用。

### ⚠️ U-01：读写混合 TPS 的基准对比

源文档原文：

> | **读写混合性能** | 2026 年基准中，读写混合 TPS 显著领先 | 纯读点查场景仍有优势 |
> — [Source-derived · S-17](07-verification-record.md#s-17)

**问题**：该行没有点名任何基准的名称、版本、方法或数据来源，因此**无法验证**。

**本课程的检索尝试与结果**（详见 [07 章 U-01](07-verification-record.md#u-01)）：尝试访问 `benchbase.org` 两次均连接被重置；`cybertec-postgresql.com/benchmarks/` 返回 403；DuckDuckGo 的 html 与 lite 端点均返回人机验证挑战；Bing 返回的"2026 基准"结果全部是内容农场类页面（无方法论、无原始数据），**不符合"官方文档、标准、原始维护者或原始发布"的可信证据标准**，因此一律不予采信。

**处理方式**：本课程**不使用**这一行作为任何论据。第 5.2 节表格中它被标为"未核实"。

### ⚠️ U-02：Stripe 的队列迁移案例

源文档原文：

> Stripe 和 GitLab 都公开复盘过这类问题，最终将队列负载迁移到 Redis 或专业消息中间件。
> — [Source-derived · S-12](07-verification-record.md#s-12)

**GitLab 部分 ✅ 已核实**：GitLab 官方开发文档确认其后台作业由 Sidekiq 处理、作业与参数存于 Redis（[R-27](07-verification-record.md#r-27)）。

**Stripe 部分 ⚠️ 未能核实**：检索尝试记录见 [07 章 U-02](07-verification-record.md#u-02)。结论是**没有找到**一篇 Stripe 的公开文章描述"把队列负载从 Postgres 迁走"这件事。

**处理方式**：本课程只引用已核实的 GitLab 事实，Stripe 部分作为源文档主张保留并明确标注，不作为论据使用。

### ⚠️ U-03：规模阈值（几千用户 → 百万级）

源文档说"当系统从几千用户增长到百万级时"（[S-14](07-verification-record.md#s-14)）。**这个具体数字没有找到任何可引用的一手来源**（详见 [07 章 U-03](07-verification-record.md#u-03)）。本课程把它当作方向性提醒（单一实例的职责会随规模增长而失控），而不是精确的水位线。

---

## 5.8 选择规则

把前面所有内容压缩成一张可执行的决策表。

| 你遇到的情况 | 该做什么 | 理由落在哪 |
| :--- | :--- | :--- |
| 数据有明确的实体关系、需要多表 Join 和强一致事务 | ✅ **PostgreSQL** | [S-05](07-verification-record.md#s-05)、[R-13](07-verification-record.md#r-13) |
| 简单的表结构、常规 CRUD、团队不想管运维细节 | ✅ **MySQL** 仍然合理 | [S-17](07-verification-record.md#s-17)、[S-22](07-verification-record.md#s-22)（"8.0 后大幅补课"已核实：[R-21](07-verification-record.md#r-21)）；⚠️ "高并发点查"出自未核实的"读写混合性能"行（[U-01](07-verification-record.md#u-01)），不作为论据 |
| 数据天然是自包含文档，且**从第一天起**就有水平扩展需求 | ✅ **MongoDB**（分片是内建能力） | [S-20](07-verification-record.md#s-20)、[R-26](07-verification-record.md#r-26) |
| 文档型数据，但需要关系完整性 + 复杂 Join + ACID | ✅ **PostgreSQL + JSONB** | [S-20](07-verification-record.md#s-20)、[R-04](07-verification-record.md#r-04)、[R-05](07-verification-record.md#r-05) |
| 高频更新的任务队列 / 计数器 | ✅ **Redis 或专用消息中间件** | [S-12](07-verification-record.md#s-12)、[R-28](07-verification-record.md#r-28) |
| PB 级列式分析、大规模并行 | ✅ **数据仓库 / 列式存储** | [S-13](07-verification-record.md#s-13)、[S-22](07-verification-record.md#s-22) |
| 大规模高并发全文检索、同义词、相关度调优 | ✅ **Elasticsearch / OpenSearch** | [S-15](07-verification-record.md#s-15)、[R-29](07-verification-record.md#r-29) |
| 地理 / 向量 / 时序，且与关系数据强关联 | ✅ **PostgreSQL + 对应扩展** | [S-04](07-verification-record.md#s-04)、[R-31](07-verification-record.md#r-31)、[R-32](07-verification-record.md#r-32)、[R-33](07-verification-record.md#r-33) |
| 单实例已经同时扛着缓存+队列+分析+向量 | ⚠️ **随规模增长会成为隐患**，再按上表逐项归位 | [S-14](07-verification-record.md#s-14)、[U-03](07-verification-record.md#u-03) |

**一条元规则**：5.8 表中指向 PostgreSQL 能力边界的各行，都可以用 [03 章 3.6 的能力判定表](03-core-functionality.md#36-能力判定表把它用在这一行的哪一侧)反查（MySQL / MongoDB 两行的依据见本章 5.2、5.3 与 [03 章 3.5](03-core-functionality.md#35-明确的边界四条不要) 的边界讨论）。选型的本质是**把你的需求定位到能力表或边界表的哪一行**，然后接受那一行的代价。

---

## 5.9 本节完成检查

- [ ] 我能不用"功能多少"而用"各自把什么放在第一位"来解释 PG 与 MySQL/MongoDB 的差异。
- [ ] 我能说出 MySQL 8.0 补上了哪两项能力（窗口函数、CTE），以及递归 CTE 剩下的一个具体限制（默认深度 1000）。
- [ ] 我能解释 PostgreSQL 与 MySQL 两种 MVCC **付账方式**的差别。
- [ ] 我能说出 MongoDB 多文档事务的两个前提条件（副本集/分片集群 + FCV 版本要求）。
- [ ] 对 Redis / Elasticsearch / PostGIS / pgvector / TimescaleDB / Citus，每个我都能说出它解决什么问题、什么时候引入。
- [ ] 我知道源文档里哪几处断言**没有**得到外部证据支持（基准对比、Stripe 案例、规模阈值，见 [07 章未解决问题](07-verification-record.md#未解决问题)），并且不会在转述时把它们当事实。
