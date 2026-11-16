# 07 · 验证记录

本文件是本课程的可追溯性底账。它记录：源文件是什么、每一条结论来自源文档的哪一行、外部查证用了什么 URL 与什么原文、有哪些问题**没有**查到答案、审校发现了什么、以及最终如何晋级到发布目录。

---

## 源文件与校验

| 项 | 值 |
| :--- | :--- |
| 根源文件（只读） | `/Users/doing/Desktop/postgresql-play/source.md` |
| 文件类型 | 普通 Markdown 文件（UTF-8） |
| 字节数 | 8196 |
| SHA-256 | `c2fad4f2569fa6316d05fcbbe58d14585dd00750309c4455d42395854d763854` |
| 解析出的结构 | 1 个无标题导语段 + 5 个 `###` 章节 + 2 个 `####` 子章节 + 2 组对比表格（L54–L62、L68–L75） |
| 源文件是否被修改 | **否**。本课程全程只读源文件；校验和在下表记录的点位复核过 |
| 源内容性质 | 公开的技术评论文章（讨论开源软件的公开特性），不含内部主机名、凭据、客户数据或私有配置 → 归类为 **public**，外部查证无需逐条审批 |

---

## 证据标签约定

课程正文中出现的每一条事实性结论都带下列三种标签之一：

| 标签 | 含义 | 记录位置 |
| :--- | :--- | :--- |
| `Source-derived · S-xx` | 直接来自源文档的表述（可能含作者主观判断） | [源定位记录](#源定位记录) |
| `Research-derived · R-xx` | 本课程查证官方文档 / 官方站点后补充的事实 | [研究记录](#研究记录) |
| `Illustrative` | 为讲清楚而构造的例子，**不主张任何未经查证的行为** | 无 |

> **两条抄录约定**：
> 1. **引文字符归一化**：摘录为源文档原文，仅将源文的中文弯引号（“ ”）统一为直引号（" "）以与全课程排版一致，文字内容未作改动。
> 2. **省略号即省略**：摘录中出现的 `……` / `…` 表示原文此处被省略，非课程改写。

**规则**：`Source-derived` 只表示"源文档这么写"，**不表示"这是事实"**。源文档中的主观判断与经验阈值同样以 `Source-derived` 标注，并在下方"未解决问题"中说明其证据强度。

---

## 源定位记录

每条记录给出：**源文档标题** → **行号范围** → **原文摘录** → **影响的课程章节**。

### S-01
- **影响章节**：01（1.1）
- **源标题**：文件开头导语（无标题段）
- **行号**：L1
- **摘录**：「它解决的核心问题是 **"在保证数据绝对可靠的前提下，尽可能让你在一个数据库里做完所有事"**，代价则是你需要接受它更重的运维负担和对使用方式的更高要求。」

### S-02
- **影响章节**：01（1.2）、02（2.1）
- **源标题**：`### 🧬 特性：一体化内核带来的"全能"底座`
- **行号**：L3–L5
- **摘录**：「PostgreSQL 的特性不是零散的功能清单，而是围绕"**正确性**"和"**可扩展性**"构建的一个整体。」

### S-03
- **影响章节**：01（1.2）、02（2.2、2.3、2.7）、03（3.1）、05（5.4）
- **源标题**：`** 并发控制的底色：MVCC 与它的代价`
- **行号**：L7–L8
- **摘录**：「PostgreSQL 的核心是 **MVCC（多版本并发控制）**，实现了"读不阻塞写，写不阻塞读"。但它的实现方式很关键：更新一行数据时，它**不会原地覆盖，而是写入一个全新的行版本**，旧版本留在数据块中成为"死元组"，靠后台的 `VACUUM` 进程回收。……**它天生适合读多写少或读写均衡的负载，但极不适合作为高频更新的队列或任务表**，因为高频 `UPDATE` 会产生海量死元组，拖垮 autovacuum。」

### S-04
- **影响章节**：01（1.2）、02（2.1、2.4）、03（3.1、3.6）、04（4.3）、05（5.6、5.8）
- **源标题**：`** "一库多用"的底气：JSONB 与丰富类型`
- **行号**：L10–L11
- **摘录**：「PostgreSQL 最显著的差异化能力是 **JSONB**。它以二进制格式存储，支持 **GIN 索引**，可以高效查询 JSON 内部的任意键值。……此外，数组、范围类型、自定义类型，以及通过扩展获得的 PostGIS（地理）、pgvector（向量）、TimescaleDB（时序）等能力，让它成为一个"可组装的数据库平台"。」

### S-05
- **影响章节**：01（1.2）、02（2.5）、03（3.1、3.6）、05（5.8）
- **源标题**：`** SQL 表达的完整性`
- **行号**：L13–L14
- **摘录**：「**窗口函数、递归 CTE、LATERAL JOIN、FULL OUTER JOIN** 都是原生且完整的。当你的查询涉及多表聚合、层次化递归或复杂的分析逻辑时，PostgreSQL 往往能用一条相对清晰的 SQL 表达，而不是把逻辑拆散到应用层。」

### S-06
- **影响章节**：02（2.6、2.8）、03（3.1）
- **源标题**：`** 索引体系的针对性`
- **行号**：L16–L17
- **摘录**：「除了常规 B-Tree，它的 **GIN**（JSONB、数组、全文检索）、**GiST**（地理空间）、**BRIN**（超大有序表）等索引类型，让特定场景的查询可以做到"对症下药"。但索引越多，写放大的代价也越大，**这是一个需要主动管理的权衡**。」
- **审校备注**：源文档把 BRIN 描述为"超大有序表"的索引；官方原文强调的是"取值与物理顺序高度相关"，未把"表很大"列为前提。见 [审计记录 A-03](#a-03)。

### S-07
- **影响章节**：01（1.3）
- **源标题**：`### 🛠️ 使用：正确的方式比"会写 SQL"更重要`
- **行号**：L19–L21
- **摘录**：「PostgreSQL 的使用门槛不在语法，而在**建模决策和运维意识**。」

### S-08
- **影响章节**：04（4.6）
- **源标题**：`** 建模原则：固定列优先，动态字段进 JSONB`
- **行号**：L23–L24
- **摘录**：「**能作为固定字段的，就不要塞进 JSONB**。……否则你会得到一张"看起来灵活、查起来全表扫描"的表，JSONB 的索引优势也会被滥用稀释。」

### S-09
- **影响章节**：02（2.3）、04（4.5、4.7）
- **源标题**：`** 必须建立 VACUUM 意识`
- **行号**：L26–L27
- **摘录**：「因为 MVCC 的设计，**任何高频写入的表都需要关注表膨胀和 autovacuum 的状态**。……在写入负载较高的系统中，调整 autovacuum 参数是运维的一部分。」

### S-10
- **影响章节**：02（2.7）、03（3.1）、04（4.1、4.2、4.3）
- **源标题**：`** 安装与入门`
- **行号**：L29–L30
- **摘录**：「在 Linux 上通过系统包管理器即可安装，默认端口通常是 `5432`。入门路径很直接：建库、建表、写入、查询、加索引、理解事务边界。真正需要花时间的是**执行计划（`EXPLAIN ANALYZE`）和索引策略**，而不是 SQL 语法本身。」

### S-11
- **影响章节**：03（3.5）
- **源标题**：`### 🚧 边界：它明确不擅长什么`
- **行号**：L32–L34
- **摘录**：「PostgreSQL 的边界比它的能力更值得记住，因为很多问题源于**把它用在它设计上就不擅长的场景**。」

### S-12
- **影响章节**：02（2.7）、03（3.5、3.6）、05（5.1、5.4、5.7、5.8）
- **源标题**：`** ❌ 不要把它当消息队列`
- **行号**：L36–L37
- **摘录**：「高频轮询、认领、更新任务状态，会产生海量死元组，让 autovacuum 彻底跟不上。Stripe 和 GitLab 都公开复盘过这类问题，最终将队列负载迁移到 Redis 或专业消息中间件。」
- **证据强度**：机制部分 ✅ 已核实（[R-02](07-verification-record.md#r-02)、[R-03](07-verification-record.md#r-03)）；GitLab 案例 ✅ 已核实（[R-27](07-verification-record.md#r-27)）；Stripe 案例 ⚠️ 未核实，见 [U-02](#u-02)

### S-13
- **影响章节**：03（3.5、3.6）、05（5.1、5.8）
- **源标题**：`** ❌ 不要指望它替代 OLAP 数据仓库`
- **行号**：L39–L40
- **摘录**：「PostgreSQL 适合**中大规模**数据，但**不适合 PB 级、超高频、超廉价存储的大数据分析场景**。列式压缩、大规模并行处理不是它的主战场。」

### S-14
- **影响章节**：03（3.5、3.6）、05（5.4、5.7、5.8）
- **源标题**：`** ❌ 不要把它当作唯一的"万能锤"`
- **行号**：L42–L43
- **摘录**：「当系统从几千用户增长到百万级时，把缓存、队列、分析、向量搜索全部塞进一个 PostgreSQL 实例，会从务实的选择变成架构隐患。单实例的写放大、索引膨胀和 VACUUM 压力会逐渐成为瓶颈。」
- **证据强度**："几千 → 百万"这一具体阈值 ⚠️ 未找到可引用的一手来源，见 [U-03](#u-03)

### S-15
- **影响章节**：03（3.1、3.5、3.6）、05（5.1、5.4、5.5、5.8）
- **源标题**：`** ⚠️ 全文检索有能力上限`
- **行号**：L45–L46
- **摘录**：「内置的 `tsvector` 全文检索在中小规模够用，但面对大规模、高并发的搜索需求，专用搜索引擎（如 Elasticsearch/OpenSearch）在功能深度和性能上仍然更合适。」

### S-16
- **影响章节**：01（1.1）、05（5.1）
- **源标题**：`### ⚖️ 与 MySQL、MongoDB 的根本分野`
- **行号**：L48–L50
- **摘录**：「这三者的差异不在"功能多少"，而在**它们各自把什么放在第一位**。」

### S-17
- **影响章节**：05（5.2、5.7、5.8）
- **源标题**：`#### PostgreSQL vs. MySQL：完整性与表达力 vs. 简单与吞吐`（对比表格）
- **行号**：L52–L62
- **摘录**：表格 7 行——设计哲学 / 复杂查询 / 半结构化数据 / MVCC 实现 / 典型负载 / 读写混合性能 / 运维负担。其中「8.0 后大幅补课」「Undo Log 回滚段，长事务下 undo 膨胀是主要风险」「2026 年基准中，读写混合 TPS 显著领先」为关键行。
- **证据强度**：逐行标注见 [05 章 5.2](05-related-tools.md#52-postgresql-vs-mysql)；其中"读写混合性能"一行 ⚠️ 未核实，见 [U-01](#u-01)

### S-18
- **影响章节**：05（5.2）
- **源标题**：`#### PostgreSQL vs. MySQL`（一句话概括）
- **行号**：L64
- **摘录**：「**MySQL 是一辆让你快速上路的小轿车；PostgreSQL 是一辆能处理复杂地形的越野车，但你需要会开、会保养**。」
- **证据强度**：作者比喻，非事实主张。

### S-19
- **影响章节**：05（5.3）
- **源标题**：`#### PostgreSQL vs. MongoDB：关系正确性 vs. 文档灵活性`（对比表格）
- **行号**：L66–L75
- **摘录**：表格 6 行——数据模型 / Schema 灵活性 / Join / 事务 / 水平扩展 / 典型数据。
- **证据强度**：逐行标注见 [05 章 5.3.3](05-related-tools.md#533-对照表含证据强度)

### S-20
- **影响章节**：05（5.1、5.3、5.8）
- **源标题**：`#### PostgreSQL vs. MongoDB`（关键变化）
- **行号**：L77
- **摘录**：「**PostgreSQL 的 JSONB 已经覆盖了相当一部分过去只有 MongoDB 才能优雅处理的场景**，同时保留了关系完整性、Join 和真正的 ACID 事务。MongoDB 的真正优势收缩到了**数据模型天然文档化、且水平扩展需求从第一天就存在**的场景。」

### S-21
- **影响章节**：01（1.1、1.3）
- **源标题**：`### 💎 最终判断：PostgreSQL 的"性价比"公式`
- **行号**：L79–L83
- **摘录**：「它非常划算，如果你的核心需求是数据正确性、复杂查询表达力、以及"尽量少引入额外组件"的整合能力……你愿意为此承担 **VACUUM 运维、表膨胀监控和更谨慎的建模决策**。」

### S-22
- **影响章节**：05（5.8）
- **源标题**：`### 💎 最终判断`（不划算条件）
- **行号**：L85
- **摘录**：「它不划算，如果**你的负载是**高频更新的队列/任务表**、**PB 级 OLAP 分析**，或者你的团队**没有精力主动管理膨胀和执行计划**。在这些边界内，Redis、专用消息中间件、数据仓库或一个更轻量的 MySQL 可能是更务实的选择。」

---

## 研究记录

**检索日期**：全部为 2026-11-16（UTC），并逐条写入各 R-xx 记录。
**采信标准**：官方文档 / 官方手册 / 官方产品页 / 项目官方仓库 README / 厂商官方开发文档。**搜索引擎摘要一律不作为证据**；本记录中的每条"原文摘录"都来自实际打开并评估过的页面正文。
**"影响章节"的含义**：该 ID 在课程正文（01–06 章）中**实际出现**的章节，可由脚本逐条核对；正文中每一处引用都能回查到底账，每条底账也能反查回正文。

### PostgreSQL 官方文档

#### R-01
- **影响章节**：02（2.1、2.2、2.8）、03（3.1）、05（5.2）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/mvcc-intro.html （§13.1 Introduction）
- **摘录**：「The main advantage of using the MVCC model of concurrency control rather than locking is that in MVCC locks acquired for querying (reading) data do not conflict with locks acquired for writing data, and so reading never blocks writing and writing never blocks reading.」
- **置信度**：高

#### R-02
- **影响章节**：01（1.2）、02（2.3、2.8）、03（3.6）、05（5.2、5.4）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/routine-vacuuming.html （§24.1.2 Recovering Disk Space）
- **摘录**：「In PostgreSQL, an UPDATE or DELETE of a row does not immediately remove the old version of the row.」「But eventually, an outdated or deleted row version is no longer of interest to any transaction.」「The space it occupies must then be reclaimed for reuse by new rows, to avoid unbounded growth of disk space requirements.」「This is done by running VACUUM.」
- **置信度**：高

#### R-03
- **影响章节**：02（2.3、2.8）、04（4.5）、05（5.2）、06（阶段5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/routine-vacuuming.html （§24.1、§24.1.2、§24.1.6 The Autovacuum Daemon）
- **摘录**：「PostgreSQL has an optional but highly recommended feature called autovacuum, whose purpose is to automate the execution of VACUUM and ANALYZE commands.」「In the default configuration, autovacuuming is enabled and the related configuration parameters are appropriately set.」「vacuum threshold = Minimum(vacuum max threshold, vacuum base threshold + vacuum scale factor * number of tuples)」「The autovacuum daemon attempts to work this way, and in fact will never issue VACUUM FULL.」「Thus, moderately-frequent standard VACUUM runs are a better approach than infrequent VACUUM FULL runs for maintaining heavily-updated tables.」「The default thresholds and scale factors are taken from postgresql.conf, but it is possible to override them (and many other autovacuum control parameters) on a per-table basis.」
- **置信度**：高

#### R-04
- **影响章节**：02（2.1、2.4、2.8）、03（3.1、3.2）、04（4.3）、05（5.2、5.3、5.8）、06（阶段3）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/datatype-json.html （§8.14 JSON Types）
- **摘录**：「The `json` and `jsonb` data types accept almost identical sets of values as input. The major practical difference is one of efficiency. The `json` data type stores an exact copy of the input text, which processing functions must reparse on each execution; while `jsonb` data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead, but significantly faster to process, since no reparsing is needed. `jsonb` also supports indexing, which can be a significant advantage.」「Because the `json` type stores an exact copy of the input text, it will preserve semantically-insignificant white space between tokens, as well as the order of keys within JSON objects. Also, if a JSON object within the value contains the same key more than once, all the key/value pairs are kept. (The processing functions consider the last value as the operative one.) By contrast, `jsonb` does not preserve white space, does not preserve the order of object keys, and does not keep duplicate object keys. If duplicate keys are specified in the input, only the last value is kept.」「In general, most applications should prefer to store JSON data as `jsonb`, unless there are quite specialized needs, such as legacy assumptions about ordering of object keys.」
- **官方示例（可逐字对照的输出）**：
  ```sql
  SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::json;
  -- {"bar": "baz", "balance": 7.77, "active":false}

  SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::jsonb;
  -- {"bar": "baz", "active": false, "balance": 7.77}
  ```
- **置信度**：高

#### R-05
- **影响章节**：01（1.2）、02（2.1、2.4、2.6、2.8）、03（3.1、3.2、3.5、3.6）、04（4.3、4.7）、05（5.2、5.5、5.8）、06（阶段3）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/gin.html （§65.4.1–65.4.6）
- **摘录**：「GIN always stores and searches for keys, not item values per se.」「A GIN index stores a set of (key, posting list) pairs, where a posting list is a set of row IDs in which the key occurs. Each key value is stored only once, so a GIN index is very compact for cases where the same key appears many times.」「Of the two operator classes for type jsonb, `jsonb_ops` is the default.」「`jsonb_path_ops` supports fewer operators but offers better performance for those operators.」「Insertion into a GIN index can be slow due to the likelihood of many keys being inserted for each item.」「For bulk insertions into a table it is advisable to drop the GIN index and recreate it after finishing bulk insertion.」
- **算子类可索引操作符**：`jsonb_ops` → `@>`、`@?`、`@@`、`?`、`?|`、`?&`；`jsonb_path_ops` → `@>`、`@?`、`@@`
- **置信度**：高

#### R-06
- **影响章节**：02（2.6）、03（3.1）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/indexes-types.html （§11.2.1–11.2.6）
- **摘录**：「B-trees can handle equality and range queries on data that can be sorted into some ordering.」「Also, an IS NULL or IS NOT NULL condition on an index column can be used with a B-tree index.」「The optimizer can also use a B-tree index for queries involving the pattern matching operators LIKE and ~ if the pattern is a constant and is anchored to the beginning of the string — for example, col LIKE 'foo%' or col ~ '^foo', but not col LIKE '%bar'.」「Hash indexes store a 32-bit hash code … such indexes can only handle simple equality comparisons.」「GiST indexes are not a single kind of index, but rather an infrastructure within which many different indexing strategies can be implemented.」「As an example, the standard distribution of PostgreSQL includes GiST operator classes for several two-dimensional geometric data types, which support indexed queries using these operators: << &< &> >> <<| &<| |&> |>> @> <@ ~= &&」「GiST indexes are also capable of optimizing "nearest-neighbor" searches.」「SP-GiST indexes, like GiST indexes, offer an infrastructure that supports various kinds of searches. SP-GiST permits implementation of a wide range of different non-balanced disk-based data structures, such as quadtrees, k-d trees, and radix trees (tries).」「GIN indexes are "inverted indexes" which are appropriate for data values that contain multiple component values, such as arrays.」「BRIN indexes (a shorthand for Block Range INdexes) store summaries about the values stored in consecutive physical block ranges of a table. Thus, they are most effective for columns whose values are well-correlated with the physical order of the table rows.」
- **置信度**：高
- **注意**：该页**未**出现"BRIN 用于超大有序表"的表述，见 [A-03](#a-03)

#### R-07
- **影响章节**：02（2.6）、03（3.1、3.4、3.5、3.6）、05（5.5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/textsearch-indexes.html （§12.9 Preferred Index Types for Text Search）
- **摘录**：「GIN indexes are the preferred text search index type. As inverted indexes, they contain an index entry for each word (lexeme), with a compressed list of matching locations.」「A GiST index is lossy, meaning that the index might produce false matches, and it is necessary to check the actual table row to eliminate such false matches.」
- **官方建索引语法示例**：
  ```sql
  -- GIN（列必须是 tsvector 类型）
  CREATE INDEX name ON table USING GIN (column);
  -- GiST（列可以是 tsvector 或 tsquery 类型）
  CREATE INDEX name ON table USING GIST (column [ { DEFAULT | tsvector_ops } (siglen = number) ] );
  ```
- **置信度**：高

#### R-08
- **影响章节**：02（2.8）、03（3.1、3.4、3.5）、04（4.7）、05（5.5）、06（阶段3）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/textsearch-intro.html （§12.1、§12.1.2）
- **摘录**：「A data type tsvector is provided for storing preprocessed documents, along with a type tsquery for representing processed queries.」「For text search purposes, each document must be reduced to the preprocessed tsvector format. Searching and ranking are performed entirely on the tsvector representation of a document — the original text need only be retrieved when the document has been selected for display to a user.」「Full text searching in PostgreSQL is based on the match operator @@, which returns true if a tsvector (document) matches a tsquery (query).」「The elements of a tsvector are lexemes, which are assumed already normalized, so rats does not match rat.」
- **官方示例（可逐字对照的输出）**：
  ```sql
  SELECT to_tsvector('fat cats ate fat rats') @@ to_tsquery('fat & rat');
  ?column?
  ----------
  t

  SELECT 'fat cats ate fat rats'::tsvector @@ to_tsquery('fat & rat');
  ?column?
  ----------
  f
  ```
- **配置说明**：单参数形式 `to_tsvector(text)` / `to_tsquery(text)` 使用服务器的 `default_text_search_config`；上例第一行返回 `t` 依赖该配置为英文系（如 `english`，其词干器把 `rats` 归一为 `rat`）。
- **置信度**：高

#### R-09
- **影响章节**：01（1.2）、02（2.5）、03（3.1、3.3）、04（4.3）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/functions-window.html （§9.22 Window Functions）
- **摘录**：「Window functions provide the ability to perform calculations across sets of rows that are related to the current query row.」「Aggregate functions act as window functions only when an OVER clause follows the call; otherwise they act as plain aggregates and return a single row for the entire set.」
- **置信度**：高

#### R-10
- **影响章节**：01（1.2）、02（2.5）、03（3.3、3.6）、05（5.2）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/queries-with.html （§7.8、§7.8.2 Recursive Queries）
- **摘录**：「The optional RECURSIVE modifier changes WITH from a mere syntactic convenience into a feature that accomplishes things not otherwise possible in standard SQL. Using RECURSIVE, a WITH query can refer to its own output.」
- **置信度**：高

#### R-11
- **影响章节**：01（1.2）、02（2.5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/queries-table-expressions.html （§7.2.1.5 LATERAL Subqueries）
- **摘录**：「Subqueries appearing in FROM can be preceded by the key word LATERAL. This allows them to reference columns provided by preceding FROM items. (Without LATERAL, each subquery is evaluated independently and so cannot cross-reference any other FROM item.)」
- **置信度**：高

#### R-12
- **影响章节**：01（1.2）、02（2.5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/sql-select.html （FROM Clause → FULL OUTER JOIN）
- **摘录**：「FULL OUTER JOIN returns all the joined rows, plus one row for each unmatched left-hand row (extended with nulls on the right), plus one row for each unmatched right-hand row (extended with nulls on the left).」
- **置信度**：高

#### R-13
- **影响章节**：02（2.2、2.7、2.8）、03（3.6）、05（5.8）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/transaction-iso.html （§13.2 Transaction Isolation）
- **摘录**：「The SQL standard defines four levels of transaction isolation. The most strict is Serializable …」「In PostgreSQL, you can request any of the four standard transaction isolation levels, but internally only three distinct isolation levels are implemented, i.e., PostgreSQL's Read Uncommitted mode behaves like Read Committed.」「Read Committed is the default isolation level in PostgreSQL. When a transaction uses this isolation level, a SELECT query (without a FOR UPDATE/SHARE clause) sees only data committed before the query began; it never sees either uncommitted data or changes committed by concurrent transactions during the query's execution.」「In effect, a SELECT query sees a snapshot of the database as of the instant the query begins to run.」「Also note that two successive SELECT commands can see different data, even though they are within a single transaction, if other transactions commit changes after the first SELECT starts and before the second SELECT starts.」「The Repeatable Read isolation level only sees data committed before the transaction began; it never sees either uncommitted data or changes committed by concurrent transactions during the transaction's execution.」「The Repeatable Read isolation level is implemented using a technique known in academic database literature and in some other database products as Snapshot Isolation.」「The Serializable isolation level provides the strictest transaction isolation.」「The Serializable isolation level is implemented using a technique known in academic database literature as Serializable Snapshot Isolation, which builds on Snapshot Isolation by adding checks for serialization anomalies.」「To set the transaction isolation level of a transaction, use the command SET TRANSACTION.」
- **置信度**：高

#### R-14
- **影响章节**：04（4.4、4.7）、06（阶段4）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/sql-explain.html
- **摘录**：「The ANALYZE option causes the statement to be actually executed, not only planned. Then actual run time statistics are added to the display, including the total elapsed time expended within each plan node (in milliseconds) and the total number of rows it actually returned. This is useful for seeing whether the planner's estimates are close to reality.」「Keep in mind that the statement is actually executed when the ANALYZE option is used. Although EXPLAIN will discard any output that a SELECT would return, other side effects of the statement will happen as usual. If you wish to use EXPLAIN ANALYZE on an INSERT, UPDATE, DELETE, MERGE, CREATE TABLE AS, or EXECUTE statement without letting the command affect your data, use this approach: BEGIN; EXPLAIN ANALYZE …; ROLLBACK.」「The overhead of repeatedly reading the system clock can slow down the query significantly on some systems, so it may be useful to set this parameter to FALSE when only actual row counts, and not exact times, are needed. Run time of the entire statement is always measured, even when node-level timing is turned off with this option.」
- **置信度**：高

#### R-15
- **影响章节**：04（4.2、4.7）、06（阶段2）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/runtime-config-connection.html
- **摘录**：`port`：「The TCP port the server listens on; 5432 by default. Note that the same port number is used for all IP addresses the server listens on. This parameter can only be set at server start.」`max_connections`：「The default is typically 100 connections, but might be less if your kernel settings will not support it (as determined during initdb).」
- **置信度**：高

#### R-16
- **影响章节**：02（2.3）、04（4.5、4.7）、05（5.4）、06（阶段5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/monitoring-stats.html （`pg_stat_all_tables` / `pg_stat_user_tables`）
- **摘录**：`n_live_tup` —「Estimated number of live rows」；`n_dead_tup` —「Estimated number of dead rows」；`last_vacuum` —「Last time at which this table was manually vacuumed (not counting VACUUM FULL)」；`last_autovacuum` —「Last time at which this table was vacuumed by the autovacuum daemon」；`last_analyze` —「Last time at which this table was manually analyzed」；`last_autoanalyze` —「Last time at which this table was analyzed by the autovacuum daemon」
- **置信度**：高

#### R-17
- **影响章节**：02（2.3）、04（4.5）、06（阶段5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/pgstattuple.html
- **摘录**：「The pgstattuple module provides various functions to obtain tuple-level statistics.」「pgstattuple returns a relation's physical length, percentage of 'dead' tuples, and other info.」「pgstattuple_approx is a faster alternative to pgstattuple that returns approximate results.」「pgstattuple acquires only a read lock on the relation. So the results do not reflect an instantaneous snapshot; concurrent updates will affect them.」
- **`pgstattuple()` 返回列**：`table_len`（物理关系长度）、`tuple_len`（存活元组总长度）、`tuple_percent`（存活元组占比）、`dead_tuple_len`（死元组总长度）、`dead_tuple_percent`（死元组占比）、`free_space`、`free_percent`、`usable_space`、`usable_percent`。
- **`pgstattuple_approx()` 返回列**：把存活元组一列改名为 `approx_tuple_count` / `approx_tuple_len` / `approx_tuple_percent`，空闲空间一列改名为 `approx_free_space` / `approx_free_percent`，并多出 `scanned_percent`（扫描比例）；死元组一列仍为 `dead_tuple_count` / `dead_tuple_len` / `dead_tuple_percent`。
- **重要口径**：`pgstattuple` **不返回存活/死元组的计数**，只给长度与百分比——计数需看 `pg_stat_user_tables` 的 `n_live_tup` / `n_dead_tup`（[R-16](07-verification-record.md#r-16)）。
- **置信度**：高
- **注意**：官方页面**未**称 `pgstattuple` 需要 `ACCESS EXCLUSIVE` 锁；课程据此纠正了流传较广的说法。

#### R-18
- **影响章节**：04（4.5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/sql-createtable.html （Storage Parameters）
- **摘录**：每表存储参数包含「`autovacuum_vacuum_threshold` … Per-table value for `autovacuum_vacuum_threshold` parameter.」「`autovacuum_vacuum_scale_factor` … Per-table value …」「`autovacuum_analyze_threshold` … Per-table value …」「`autovacuum_analyze_scale_factor` … Per-table value …」
- **置信度**：高

#### R-19
- **影响章节**：04（4.1、4.2）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/download/linux/ubuntu/
- **摘录**：「Ubuntu includes PostgreSQL by default. To install PostgreSQL on Ubuntu, use the apt (or other apt-driving) command:」→ `apt install postgresql`
- **置信度**：高

#### R-20
- **影响章节**：04（4.1、4.2）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/download/linux/redhat/
- **摘录**：`dnf install postgresql-server` → `postgresql-setup --initdb` → `systemctl enable postgresql.service` → `systemctl start postgresql.service`
- **置信度**：高

### MySQL 官方文档

#### R-21
- **影响章节**：05（5.2、5.8）
- **检索日期**：2026-11-16
- **URL**：https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html （§1.3 What Is New in MySQL 8.0）
- **摘录**：「**Window functions.** MySQL now supports window functions that, for each row from a query, perform a calculation using rows related to this row.」「**Common table expressions.** MySQL now supports common table expressions, both nonrecursive and recursive.」「**JSON enhancements.** … Added the `->>` (inline path) operator … Added two JSON aggregation functions `JSON_ARRAYAGG()` and `JSON_OBJECTAGG()` … Added support in MySQL 8.0.2 for partial, in-place updates of JSON column values, which is more efficient than completely removing an existing JSON value and writing a new one in its place, as was done previously when updating any JSON column. … Added the `JSON_TABLE()` function in MySQL 8.0.4.」
- **置信度**：高
- **说明**：这是源文档"8.0 后大幅补课"的直接依据（[S-17](07-verification-record.md#s-17)）。

#### R-22
- **影响章节**：05（5.2）
- **检索日期**：2026-11-16
- **URL**：https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html （§17.3 InnoDB Multi-Versioning）
- **摘录**：「InnoDB is a multi-version storage engine. It keeps information about old versions of changed rows to support transactional features such as concurrency and rollback. This information is stored in undo tablespaces in a data structure called a rollback segment.」「It is recommend that you commit transactions regularly, including transactions that issue only consistent reads. Otherwise, InnoDB cannot discard data from the update undo logs, and the rollback segment may grow too big, filling up the undo tablespace in which it resides.」
- **置信度**：高
- **说明**：这是源文档"Undo Log 回滚段，长事务下 undo 膨胀是主要风险"的直接依据（[S-17](07-verification-record.md#s-17)）。

#### R-23
- **影响章节**：05（5.2）
- **检索日期**：2026-11-16
- **URL**：https://dev.mysql.com/doc/refman/8.0/en/window-functions.html （§14.20）
- **摘录**：「MySQL supports window functions that, for each row from a query, perform a calculation using rows related to that row.」
- **置信度**：高

#### R-24
- **影响章节**：05（5.2）
- **检索日期**：2026-11-16
- **URL**：https://dev.mysql.com/doc/refman/8.0/en/with.html （§15.2.20 WITH）
- **摘录**：「A common table expression (CTE) is a named temporary result set that exists within the scope of a single statement and that can be referred to later within that statement, possibly multiple times.」「The `cte_max_recursion_depth` system variable enforces a limit on the number of recursion levels for CTEs. The server terminates execution of any CTE that recurses more levels than the value of this variable.」「By default, `cte_max_recursion_depth` has a value of 1000, causing the CTE to terminate when it recurses past 1000 levels.」
- **置信度**：高

### MongoDB 官方文档

#### R-25
- **影响章节**：05（5.3）
- **检索日期**：2026-11-16
- **URL**：https://www.mongodb.com/docs/manual/core/transactions/
- **摘录**：「In MongoDB, an operation on a single document is atomic. The MongoDB document model supports embedding related data within a single document, which reduces the need for multi-document transactions. When your application requires atomicity across multiple documents, use transactions.」「In most cases, a distributed transaction incurs a greater performance cost over single document writes, and the availability of distributed transactions should not be a replacement for effective schema design. For many scenarios, the denormalized data model (embedded documents and arrays) will continue to be optimal for your data and use cases.」「Transactions are supported on replica sets and sharded clusters.」「To use transactions, the `featureCompatibilityVersion` for all members of the deployment must be at least: Replica Set → 4.0; Sharded Cluster → 4.2.」
- **置信度**：高
- **说明**：这是源文档"MongoDB 4.0+ 支持多文档事务，但官方明示有性能代价，事务是'后补的能力'"的直接依据（[S-19](07-verification-record.md#s-19)）。

#### R-26
- **影响章节**：05（5.3、5.8）
- **检索日期**：2026-11-16
- **URL**：https://www.mongodb.com/docs/manual/sharding/
- **摘录**：「Sharding is a method for distributing data across multiple machines. MongoDB uses sharding to support deployments with very large data sets and high throughput operations.」「A MongoDB sharded cluster consists of the following components: shard … Routing with mongos … config servers …」
- **置信度**：高

### 其他厂商与项目官方来源

#### R-27
- **影响章节**：03（3.5）、05（5.4、5.7）
- **检索日期**：2026-11-16
- **URL**：https://docs.gitlab.com/development/sidekiq.html
- **摘录**：「We use Sidekiq as our background job processor.」「GitLab stores Sidekiq jobs and their arguments in Redis.」
- **置信度**：高
- **说明**：佐证源文档"GitLab … 将队列负载迁移到 Redis"这一半（[S-12](07-verification-record.md#s-12)）。该页**未**说明"为何选择 Redis 而非数据库表"，课程因此未做此类主张。

#### R-28
- **影响章节**：05（5.4、5.8）
- **检索日期**：2026-11-16
- **URL**：https://redis.io/docs/latest/develop/get-started/
- **摘录**：「Redis can be used as a database, cache, streaming engine, message broker, and more.」
- **置信度**：中高（该页为快速上手导航页，取用其首段定位句；`redis.io/docs/` 301 跳转至 `/docs/latest/`，该路径同为导航页，正文未给出更完整的产品定义句）

#### R-29
- **影响章节**：05（5.5、5.8）
- **检索日期**：2026-11-16
- **URL**：https://www.elastic.co/elasticsearch
- **摘录**：「Elasticsearch is an open source, distributed search and analytics engine built for speed, scale, and AI applications.」（页面标题：Elasticsearch: The Official Distributed Search & Analytics Engine | Elastic）
- **置信度**：高

#### R-30
- **影响章节**：05（5.5）
- **检索日期**：2026-11-16
- **URL**：https://opensearch.org/
- **摘录**：「OpenSearch is open source that stays open: vendor-neutral, fully Apache 2.0 licensed from ingest to dashboard」「Index once, then search, retrieve, and analyze across everything you build and run.」
- **置信度**：中（首页营销文案，未取到规范化的产品定义页；该页亦未提及 Elasticsearch 的沿革关系，故课程未做"OpenSearch 源自 Elasticsearch"之类的断言）

#### R-31
- **影响章节**：03（3.1、3.6）、05（5.6、5.8）
- **检索日期**：2026-11-16
- **URL**：https://postgis.net/
- **摘录**：「PostGIS extends the capabilities of the PostgreSQL relational database by adding support for storing, indexing, and querying geospatial data.」「Spatial Data Storage: Store different types of spatial data such as points, lines, polygons, and multi-geometries, in both 2D and 3D data.」「Spatial Indexing: Quickly search and retrieve spatial data based on its location.」
- **置信度**：高

#### R-32
- **影响章节**：03（3.1、3.6）、05（5.6、5.8）
- **检索日期**：2026-11-16
- **URL**：https://github.com/pgvector/pgvector
- **摘录**：「Open-source vector similarity search for Postgres」「Supported index types are: HNSW, IVFFlat」「By default, pgvector performs exact nearest neighbor search, which provides perfect recall.」「You can add an index to use approximate nearest neighbor search, which trades some recall for speed.」
- **置信度**：高

#### R-33
- **影响章节**：03（3.1、3.6）、05（5.6、5.8）
- **检索日期**：2026-11-16
- **URL**：https://www.tigerdata.com/ （`timescale.com` 301 跳转至此）
- **摘录**：「Our capabilities let Postgres ingest it fast, store years of it cheaply, and query live and historical data together.」「Time- and key-based partitioning for fast reads and writes.」「Row storage for writes, columnar storage for analytics, with compression.」「200+ SQL functions for time-based analytics.」
- **置信度**：中（取自 Tiger Data 站点首页的产品能力描述；本次未取到 TimescaleDB 的独立产品定义页或超表（hypertable）机制说明，故课程**未**在正文中断言"超表"这一具体机制）
- **注意**：源文档把 TimescaleDB 描述为"时序"扩展（[S-04](07-verification-record.md#s-04)），官方页面的表述与之方向一致。

#### R-34
- **影响章节**：03（3.1、3.5）、05（5.3、5.6）
- **检索日期**：2026-11-16
- **URL**：https://www.citusdata.com/
- **摘录**：「Citus gives you the Postgres you love plus horizontal scale-out with schema-based and row-based sharding.」「You can start with a single Citus node, then add nodes & rebalance shards when you need to grow.」
- **置信度**：高
- **说明**：这是源文档"分片依赖 Citus 等扩展"的直接依据（[S-19](07-verification-record.md#s-19)）。

### 补充研究记录（第二轮查证：为审校中发现的证据缺口补齐）

#### R-35
- **影响章节**：03（3.3）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/sql-expressions.html （§4.2.7 Aggregate Expressions）
- **摘录**：「aggregate_name (expression [ , ... ] ) [ FILTER ( WHERE filter_clause ) ]」「If FILTER is specified, then only the input rows for which the filter_clause evaluates to true are fed to the aggregate function; other rows are discarded.」
- **官方示例**：`SELECT count(*) AS unfiltered, count(*) FILTER (WHERE i < 5) AS filtered FROM generate_series(1,10) AS s(i);`
- **置信度**：高

#### R-36
- **影响章节**：04（4.1、4.5）、06（阶段5）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/sql-vacuum.html
- **摘录**：「VACUUM FULL rewrites the entire contents of the table into a new disk file with no extra space, allowing unused space to be returned to the operating system. This form is much slower and requires an ACCESS EXCLUSIVE lock on each table while it is being processed.」「Selects "full" vacuum, which can reclaim more space, but takes much longer and exclusively locks the table. This method also requires extra disk space, since it writes a new copy of the table and doesn't release the old copy until the operation is complete.」「Plain VACUUM (without FULL) simply reclaims space and makes it available for re-use. This form of the command can operate in parallel with normal reading and writing of the table, as an exclusive lock is not obtained.」
- **置信度**：高

#### R-37
- **影响章节**：04（4.1）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/sql-createindex.html
- **摘录**：「When this option is used, PostgreSQL will build the index without taking any locks that prevent concurrent inserts, updates, or deletes on the table; whereas a standard index build locks out writes (but not reads) on the table until it's done.」「PostgreSQL supports building indexes without locking out writes. … Thus this method requires more total work than a standard index build and takes significantly longer to complete.」「Another difference is that a regular CREATE INDEX command can be performed within a transaction block, but CREATE INDEX CONCURRENTLY cannot.」
- **置信度**：高

#### R-38
- **影响章节**：04（4.6）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/ddl-constraints.html （§5.5.1 Check Constraints）
- **摘录**：「A check constraint is the most generic constraint type.」「PostgreSQL assumes that CHECK constraints' conditions are immutable, that is, they will always give the same result for the same input row.」「PostgreSQL does not support CHECK constraints that reference table data other than the new or updated row being checked.」
- **置信度**：中高（该页未出现"CHECK 约束不使用索引"的原文；正文对该点的表述已相应收窄，见 [A-06](#a-06)）

#### R-39
- **影响章节**：05（5.3）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/high-availability.html （§26.2.2 Standby Server Operation、§26.4 Hot Standby）
- **摘录**：「Servers that track changes in the primary are called standby or secondary servers.」「A standby server that cannot be connected to until it is promoted to a primary server is called a warm standby server, and one that can accept connections and serves read-only queries is called a hot standby server.」
- **置信度**：中（该页只确认了"跟踪主库的备库"与"热备可提供只读查询"；源文档所说的"流复制成熟"本课程未找到更强的定性或量化出处，正文已相应标注为源文档主张）

#### R-40
- **影响章节**：03（3.4）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/textsearch-dictionaries.html （§12.6 Dictionaries）
- **摘录**：「A text search configuration binds a parser together with a set of dictionaries to process the parser's output tokens.」
- **置信度**：中（该页给出"文本搜索配置"的定义，但未列出可用配置的完整清单；正文因此只说"`'english'` 是一个文本搜索配置"，不说"内置配置之一"的完整枚举）

#### R-41
- **影响章节**：03（3.4）
- **检索日期**：2026-11-16
- **URL**：https://www.postgresql.org/docs/current/textsearch-tables.html （§12.2.2 Creating Indexes）
- **摘录**：「We can create a GIN index (Section 12.9) to speed up text searches:」「Another approach is to create a separate tsvector column to hold the output of to_tsvector. To keep this column automatically up to date with its source data, use a stored generated column.」
- **官方示例（可逐字对照）**：
  ```sql
  CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', body));

  ALTER TABLE pgweb
  ADD COLUMN textsearchable_index_col tsvector
  GENERATED ALWAYS AS (to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))) STORED;

  CREATE INDEX textsearch_idx ON pgweb USING GIN (textsearchable_index_col);

  SELECT title
  FROM pgweb
  WHERE textsearchable_index_col @@ to_tsquery('create & table')
  ORDER BY last_mod_date DESC
  LIMIT 10;
  ```
- **置信度**：高（该页未给出 `ts_rank` 的用法示例，正文因此不展开排名函数）

---

## 未解决问题

以下问题在本次查证中**没有**得到可靠证据。它们被保留在课程正文里，但**标注为"源文档主张"**，且不作为任何论据使用。

### U-01
- **影响章节**：05（5.7、5.8）
- **源文档主张**：「2026 年基准中，读写混合 TPS 显著领先」（[S-17](07-verification-record.md#s-17)）
- **缺口内容**：该断言未点名基准名称、数据库版本、硬件、方法论或数据出处，因此无法复现或证伪。
- **检索尝试与结果**：
  1. `https://benchbase.org/` — 两次均返回 "Client network socket disconnected before secure TLS connection was established (ECONNRESET)"。
  2. `https://www.cybertec-postgresql.com/benchmarks/` — HTTP 403 Forbidden。
  3. DuckDuckGo `html` 与 `lite` 端点 — 均返回人机验证挑战（"Select all squares containing a duck"），无法取回结果列表。
  4. Bing — 两次查询分别返回与主题无关的结果（一次全为 Microsoft 站点，一次全为"about"的词典释义页）；另一次返回的"2026 基准"结果全部为无方法论的内容农场页面。
- **结论**：**未核实**。课程中该行仅以"源文档主张"呈现（[05 章 5.7](05-related-tools.md#57-三条没有证据支撑的对比)），不参与任何论证。
- **状态**：Open —— 若要闭合，需要用户指定具体基准来源，或允许在可访问基准站点（如 Benchbase）环境下重新检索。

### U-02
- **影响章节**：03（3.5）、05（5.7）
- **源文档主张**：「Stripe 和 GitLab 都公开复盘过这类问题，最终将队列负载迁移到 Redis 或专业消息中间件」（[S-12](07-verification-record.md#s-12)）
- **已核实部分**：GitLab —— 官方开发文档确认使用 Sidekiq 处理后台作业、作业与参数存于 Redis（[R-27](07-verification-record.md#r-27)）。
- **未核实部分**：Stripe —— 未找到任何公开的一手材料描述"把队列负载从 Postgres 迁走"。
- **检索尝试与结果**：
  1. `https://stripe.com/blog/choose-the-boring-database`、`https://stripe.com/blog/choose-boring-technology`、`https://stripe.dev/blog/choose-boring-technology` — 均 404。
  2. `https://stripe.dev/blog/` — 成功打开并枚举了博客索引，其中与数据库相关的条目为：*How Stripe uses graph search and state machines to auto-remediate a global database fleet*、*How Stripe's document databases supported 99.999% uptime with zero-downtime data migrations*、*How do I store inventory data in my Stripe application*、*From naive webhooks to durable sync: Queue-based and event-driven data reconciliation patterns*。逐篇打开后**均未**出现"Postgres 用作队列"或"把队列迁离 Postgres"的表述。
  3. 检索到的第三方文章 *Postgres as a Queue in 2026*（blog.rajpoot.dev，2026-05-03）观点与源文档**相反**，且为个人博客、非一手权威来源，不予采信。
- **结论**：**未核实**。课程只引用已核实的 GitLab 事实，Stripe 部分作为源文档主张保留并显式标注。
- **状态**：Open。

### U-03
- **影响章节**：03（3.5）、05（5.4、5.7、5.8）
- **源文档主张**：「当系统从几千用户增长到百万级时……」（[S-14](07-verification-record.md#s-14)）
- **缺口内容**："几千用户 → 百万用户"这一具体阈值没有可引用的一手来源；用户数与数据库负载之间也不存在通用换算关系。
- **结论**：课程将其作为**方向性提醒**呈现（单一实例的职责会随规模增长而失控），明确说明它不是精确的容量水位线（[05 章 5.7](05-related-tools.md#57-三条没有证据支撑的对比)）。正文中出现该阈值的三类位置——[03 章 3.5](03-core-functionality.md#35-明确的边界四条不要)、[05 章 5.4](05-related-tools.md#54-什么时候该引入-redis-或消息中间件)、[05 章 5.7](05-related-tools.md#57-三条没有证据支撑的对比)——以及 [05 章 5.8](05-related-tools.md#58-选择规则) 决策表末行的依据栏，四处均带有指向本条记录的提示。
- **状态**：Open（属作者经验判断，接受现状即可闭合）。

---

## 审计记录

### 审计输入与执行方式

| 项 | 值 |
| :--- | :--- |
| 审计对象 | 暂存目录 `.course-staging-20261116T092500/` 内的 7 个章节文件（规范路径与逐条校验和见该目录下的 `MANIFEST.md`；清单不晋级，故此处以路径引用而非链接） |
| 审计边界 | 暂存目录 = 唯一可读写的目标；根源文件 `source.md` 只读；`course/` 目标目录在审批前禁止写入 |
| 执行方式 | 逐文件 **review-only** post-mortem：审校者只返回发现，**不修改任何文件**；修复由本流程在暂存目录内统一执行 |
| 覆盖 | 7/7 个章节文件全部完成独立审校（其中 3 个章节因首次执行超时被重新派发并完成） |
| 随后 | 独立的跨章节一致性复核（只读），针对术语、证据 ID、链接与主张一致性 |

### A-01
- **标题**：证据标签 ID 与链接锚点错配（`R-30` 实指 `#r-17`）
- **严重度**：High
- **位置**：`02-core-concepts.md:145`、`04-usage.md:266`
- **发现**：两处 `pgstattuple` 的官方引文标注为 `R-30`，但链接锚点是 `#r-17`；`R-30` 实际是 OpenSearch 首页记录，`R-17` 才是 `pgstattuple` 记录。读者按标签回查会落到完全无关的记录。
- **处理**：两处标签统一改为 `[Research-derived · R-17](07-verification-record.md#r-17)`。这是全部七章中唯一一处被独立审校同时在 5 个章节里报告的缺陷。

### A-02
- **标题**：示例块与事实性主张缺少证据标签
- **严重度**：Medium
- **位置**：`01`（§1.2 因果链、§1.5 约定文本）、`02`（2.1/2.3 图示块、2.7 命令块）、`03`（3.1 表、3.5 架构判断、3.7 检查项）、`04`（4.1 表、4.2 安装说明、4.7 错误清单表、9 个代码块）、`05`（5.2 表、5.4/5.5/5.6 推断句、5.5 表）、`06`（整章零标签）
- **发现**：01 章承诺"每个示例都带标签""每条事实性结论都标注来源"，但全课程有 17 个代码/输出块与数十条事实性主张未标注；06 章整章一条标签都没有，而底账的 R-14 / R-16 / R-19 / R-20 已把 06 列为受影响章节。
- **处理**：
  1. 01 章 §1.5 的约定文本改为与实际做法一致，并显式补上"Source-derived ≠ 事实"这条规则。
  2. 为全部裸代码块补 `**示例标签**：…` 标记；为无标签的事实性主张补 `S-xx` / `R-xx` 链接，或在无出处时改写为不作断言的表述。
  3. 06 章补齐阶段级与实验级的标签，并新增第 5 条约定说明。

### A-03
- **标题**：BRIN 索引的措辞与官方表述不一致
- **严重度**：Medium
- **位置**：`02-core-concepts.md` 2.6 节；底账 `S-06` 的审校备注
- **发现**：源文档写 BRIN 适用于"超大有序表"，官方原文强调的是"取值与物理顺序高度相关"，**没有**把"表很大"列为前提。
- **处理**：02 章保留源文档表述但明确标注差异，并指向官方定义 [R-06](07-verification-record.md#r-06)；底账 `S-06` 的审校备注指向本条目。

### A-04
- **标题**：正文引文超出研究记录摘录（证据链断在引文中间）
- **严重度**：Medium
- **位置**：`02`（2.3/2.4/2.6/2.7）、`03`（3.2/3.4）、`04`（4.4/4.5）、`05`（5.2/5.2.1/5.3）
- **发现**：正文的引文长于对应 R-xx 记录的摘录（缺 `……` 省略标记，或直接多出整句），读者按 ID 回查时找不到被引的那部分原文。涉及 `R-02`、`R-04`、`R-05`、`R-06`、`R-07`、`R-08`、`R-13`、`R-14`、`R-17`、`R-21`、`R-24` 共 11 条记录。
- **处理**：**扩充记录而非降级正文**——把第二轮查证中实际从官方页面取得的原句补录进对应 R-xx 的摘录（含官方示例输出），并为全文检索建索引语法、`FILTER` 子句、`VACUUM FULL` 锁语义、`CREATE INDEX` 锁语义、CHECK 约束语义、只读副本语义新增 `R-35`…`R-39` 五条记录。正文中超出记录范围且确实无官方出处的断言（如"PostgreSQL 递归 CTE 没有等价深度限制"）已改写为不下结论。

### A-05
- **标题**：未解决断言的处理在章节之间不一致
- **严重度**：Medium
- **位置**：`01`（§1.5 第 3 条）、`03`（3.5）、`05`（5.4/5.7/5.8）、`06`（阶段 6）
- **发现**：底账登记了三条未解决问题（U-01 基准、U-02 Stripe、U-03 规模阈值），但 01 章只说"两处"；05 章的 5.8 决策表把 U-01 判定为未核实的"纯读点查"当作选 MySQL 的触发条件；U-03 的阈值在 05 章 5.4 / 5.8 缺少内联提示。
- **处理**：01 章统一为"三处"；05 章 5.8 的该行改为不依赖 U-01 的表述并显式注明其未核实来源；U-03 在 03、05 两处补内联提示与回链；06 章阶段 6 的完成检查同步为三条。

### A-06
- **标题**：SQL 语义、示例可运行性与事实错误
- **严重度**：High / Medium
- **位置**：`02`（2.7 `SET TRANSACTION`、2.3 缺 INSERT、2.5 缺建表、2.7 队列示例结局）、`03`（3.3 需求与 SQL 相反、窗口函数求值顺序、3.2 空白描述、3.2 `@@` 重载）、`04`（4.6 CHECK 反模式、`pgstattuple` 返回列、4.4 判读清单、`EXPLAIN ANALYZE` 副作用）、`05`（5.2.1 递归深度）、`06`（阶段 4 的 `ROLLBACK` 教学、阶段 4 的 `?` 占位符）
- **发现**：
  - 03 章 3.3 的需求写"最近 3 笔"，SQL 却按金额排名，两者相反，且 `RANK()` 在并列时可能选出多于 3 行。
  - 04 章 4.6 的反模式注释断言"CHECK 写不了 / 需要函数索引"——`data->>'amount' > 0` 根本不是合法表达式，CHECK 也能写（要绕一层类型转换）。
  - 04 章 4.5 把 `pgstattuple` 的返回列写成"存活元组数 / 死元组数"，而官方返回类型里没有这两个计数列。
  - 06 章阶段 4 要求"对 `UPDATE` 直接跑 `EXPLAIN ANALYZE` 再 `ROLLBACK`"——在自动提交下 `ROLLBACK` 无效，与 04 章明令禁止的用法冲突。
  - 02 章的 `SET TRANSACTION` 单独执行无效（必须写在事务块内）。
- **处理**：全部按审校意见改写或补齐前置条件；3.3 改为 `ROW_NUMBER()` 并把需求与 SQL 对齐；4.6 的反模式改为"约束要绕一层类型转换"；4.5 按 [R-17](07-verification-record.md#r-17) 修正返回列口径并说明计数要看 `pg_stat_user_tables`；06 章阶段 4 改为"先在事务外踩坑、再用 `BEGIN;…;ROLLBACK;` 验证"；`SET TRANSACTION` 改为事务块内写法。

### A-07
- **标题**：跨章引用与底账"影响章节"映射错误
- **严重度**：Low / Medium
- **位置**：`05`（5.1、5.3.2、5.3.3、5.8）、`07`（`S-01`/`S-05`/`S-06`/`S-10`/`S-14`/`S-19`/`S-22`/`U-03`/`R-17`）
- **发现**：`S-17`（MySQL 对比表）被用来支撑出自 MongoDB 对比表的"分片依赖 Citus"；05 章 5.1 的"三问"错配 `S-13`（OLAP 边界）而漏掉真正对应的 `S-20`；底账有 4 条记录（`S-19`/`S-22`/`U-03`/`R-17`）在正文中零引用，正文与底账双向不可追溯。
- **处理**：更正全部错配引用；在 05 章补 `S-19`/`S-22`/`U-03` 的正文入口；更新底账全部"影响章节"字段，并在字段说明处明确其含义（内容受影响 ≠ ID 出现位置）。

### A-08
- **标题**：引文字符归一化规则未声明
- **严重度**：Low
- **位置**：全课程 22 条 `S-xx` 摘录与 01–06 正文引文
- **发现**：源文档使用中文弯引号（“ ”），课程统一改用直引号（" "）。文字内容未改，属排版归一，但底账未声明该规则，字符级回查会不匹配。
- **处理**：在 [证据标签约定](#证据标签约定) 中显式声明"引文字符归一化"与"省略号即省略"两条抄录约定，不逐处改动引文。

### A-09
- **标题**：`source.md` 引文的逐字核对
- **严重度**：Low
- **位置**：`S-01`…`S-22`
- **发现**：`S-03` 的摘录把源句以句号收尾（源文以逗号续接），`S-06` 的摘录静默截断了源文结论句"**这是一个需要主动管理的权衡**"，两处均未使用本文件惯用的省略号标记。
- **处理**：两处摘录恢复为与源文逐字一致（含加粗边界），并对省略处显式标注 `……`。

### A-10
- **标题**：审校后的跨章节一致性复核（只读，独立于逐文件审校）
- **严重度**：—（复核结论）
- **范围**：术语（死元组 / 表膨胀 / 算子类 / 词元归一 / 快照）、证据 ID（S-xx / R-xx / U-xx 的定义与引用闭合）、本地链接与锚点、"两处 vs 三处"计数、"3.6 能力判定表 ↔ 5.8 选择规则"的互为镜像关系、章节编号互引
- **结论**：复核中发现并已修正——`05` 章 5.8 的"每一条都能用 3.6 反查"过宽（3.6 表里没有 MySQL / MongoDB 行），已收窄；`03` 章 3.6 缺少"且与关系数据强关联"的限定，已与 05 章 5.8 对齐；`05` 章 5.5 的"几千到几十万文档"是课程自造的量化阈值且该表无证据列，已改为源文档口径"中小规模"并补证据列；`02` 章学习目标第 4 条（读懂 `EXPLAIN ANALYZE`）在本章无对应内容，已改写为指向 04 章 4.4。

### A-11
- **标题**：审校后的跨章节一致性复核（只读，独立于逐文件审校；共 15 项发现，全部已修）
- **严重度**：High / Medium / Low（混合）
- **范围**：术语一致性、跨章技术主张矛盾、跨章引用准确性、底账"影响章节"映射、未解决断言的呈现一致性、数量口径、重复内容、操作指令一致性、标签/锚点完整性、示例 schema 命名一致性
- **主要发现与处理**：
  1. **`pgstattuple` 返回列口径在 02 与 04 之间互斥**，且 02 的版本与 R-17 的"重要口径"冲突（02 列出了并不存在的 `tuple_count` / `dead_tuple_count`）。→ 02 改为"长度与占比，不返回计数"，示意列改为官方真实列名。
  2. **02 §2.5 的示例 schema 自相矛盾**（声明 `departments`，SQL 读 `employees`；声明 `orders(id)`，SQL 用 `order_id`），且存在两个重复的 LATERAL 块。→ 统一为 `employees` / `orders(id)`，删除重复块，03 §3.3 的同类说明同步。
  3. **"两条 vs 三处 vs 四处"的计数链在 01 / 05 / 06 / 07 之间互相打架**（05 §5.7 标题写"两条"但节内三条）。→ 05 §5.7 标题改为"三条"，四处引用的锚点同步更新；U-03 的"三处"改为"三类位置 + 5.8 决策表末行"共四处；06 阶段 6 的完成检查从只提 U-01 改为三条并列。
  4. **底账"影响章节"字段有 17 条指向从未引用该 ID 的章节，且 03 §3.6 / 02 §2.8 / 04 §4.7 三张整节证据表完全未登记。** → 该字段改为"该 ID 在正文中实际出现的章节"，并由脚本按 01–06 的实际引用位置重算全部 66 条记录；字段定义同步改写为可脚本核对的口径。
  5. **`@@` 重载消歧只落在 02 §2.4**，03 章同时使用两种 `@@` 却没有提示；且 02 所说的"全文检索一节"在 02 章内并不存在。→ 03 §3.2 补消歧注，02 改为指向 [03 章 3.4](03-core-functionality.md)。
  6. **03 §3.4 的全文检索示例依赖全课程从未定义的 `articles` 表**，而该章的学习目标要求"跑通"。→ 补建表提示；03 §3.2 补 `products` 表的来源指引。
  7. **02 §2.3 的 S-03 引文未按"省略号即省略"约定标记删节**；02 内有两处重复的 ⚠️ 与重复的示例标签。→ 补齐引文、删除重复内容、区分两个同为 R-13 的标签。
  8. **`EXPLAIN ANALYZE` 的禁令强度在 04 与 06 之间不一致**，且 06 原本会让学员在练习表上真的改掉数据。→ 04 改为"一律要包在事务里（唯一例外是在可丢弃的练习表上故意演示副作用）"，06 的踩坑步骤改为在一张新建的 `explain_demo` 练习表上做。
  9. **05 §5.1 提到不存在的"第 4 问"**；**06 §6.2 的阶段图与阶段 3 的"读"清单不一致**。→ 分别改为"属于分析型负载，不在本节这三问之内"与补上 02 §2.4。
  10. **暂存清单 MANIFEST.md 的七个校验和与字节数在审校后全部过期**。→ 晋级前按最终内容重新生成清单。
- **复核结论**：修复后重跑脚本校验——本地链接与锚点 0 处失效、证据标签与锚点 0 处错配、底账 66 条记录全部有正文引用位置、无孤儿引用、无未使用记录、六章内无未标注的示例块、无重复标签。

---

## 晋级审批记录

本次晋级为**全新创建**，不涉及任何覆盖：目标目录 `course/` 在本课程开始时并不存在（已用 `ls -ld` 与 `realpath` 复核），因此无需对既有文件申请覆盖授权。

| 项 | 值 |
| :--- | :--- |
| 目标目录（canonical） | `/Users/doing/Desktop/postgresql-play/course` |
| 目标目录创建方式 | 新建目录（当前不存在） |
| 文件名白名单（仅此 7 个） | `01-positioning-and-background.md`、`02-core-concepts.md`、`03-core-functionality.md`、`04-usage.md`、`05-related-tools.md`、`06-learning-path.md`、`07-verification-record.md` |
| 每个文件的操作 | **create**（全部为新建，无 replace、无 delete） |
| 不晋级的内容 | `MANIFEST.md`（暂存清单，仅用于晋级校验，不进入课程目录） |
| 禁止触碰的路径 | 根源文件 `source.md`（只读）、`.git/**`、本清单以外的一切路径 |
| 暂存清单校验和 | `f8aedaa7505f36e87ce4b2e444968c8c62adcb611ef27dba0ee7a01fc40bad58` |
| 暂存清单校验和的定义 | SHA-256，依次输入 01–06 六个文件的完整字节，再输入 `07-verification-record.md` 中 "## 晋级审批记录" 之前的全部字节。**该定义刻意排除本审批记录与最终结果两节**，使得写入本记录本身不会使审批失效 |
| 审批生效条件 | 上述校验和必须与晋级前重算的值一致；根源文件校验和必须仍为 `c2fad4f2569fa6316d05fcbbe58d14585dd00750309c4455d42395854d763854`；目标目录规范路径必须不变；路径树中不得出现符号链接；七个目标必须都是普通文件 |

### 晋级前逐项复核结果

| 检查项 | 结果 |
| :--- | :--- |
| 根源文件 SHA-256 未变 | ✅ `c2fad4f2…3854`，与课程开始时一致 |
| 七个章节文件齐全、命名与顺序正确 | ✅ `01`…`07` 连续编号，文件名与章节顺序对应 |
| 本地 Markdown 链接全部可解析 | ✅ 脚本校验，0 处失效 |
| 锚点全部存在 | ✅ 脚本校验，0 处失效 |
| 证据标签 ID 与锚点一致 | ✅ 脚本校验，0 处错配 |
| 底账 66 条记录（S-01…S-22、R-01…R-41、U-01…U-03）均有正文引用位置 | ✅ 无孤儿引用、无未使用记录 |
| 每个事实性结论都有证据标签 | ✅ 六章逐块检查，无未标注的示例块 |
| 每章都有学习目标、证据标签、本地链接与至少一个具体示例 | ✅ |
| 审校发现全部修复 | ✅ 逐文件审校 15 项 + 跨章节复核 15 项，共 30 项，全部处理并记入 A-01…A-11 |
| 未解决断言已显式标注且未被当作事实使用 | ✅ U-01 / U-02 / U-03 在 01、03、05、06 四处呈现一致 |

### 晋级范围之外的事项（仍为 Open，需人工决定）

- **U-01**：源文档的"2026 年基准中读写混合 TPS 显著领先"未找到可核实的一手来源。
- **U-02**：源文档的"Stripe 把队列负载迁移到 Redis"未找到可核实的一手来源（GitLab 一半已由 R-27 核实）。
- **U-03**：源文档的"几千用户 → 百万级"阈值未找到可核实的一手来源。

三条均已降级为"源文档主张 + 未解决问题"并在正文标注，不影响晋级；但若日后要引用这三点，需要先闭合对应记录。


---

## 最终结果

**晋级结果**：七个课程文件已创建于 `/Users/doing/Desktop/postgresql-play/course/`，全部为新建操作，未覆盖任何既有文件。

| 文件 | 内容 |
| :--- | :--- |
| `01-positioning-and-background.md` | 定位与背景：核心命题、受众、课程地图、证据标注约定 |
| `02-core-concepts.md` | 核心概念：MVCC / 死元组 / VACUUM、JSONB + GIN、SQL 表达力、索引体系、事务隔离 |
| `03-core-functionality.md` | 核心能力：能力矩阵、可运行示例、四条边界、能力判定表 |
| `04-usage.md` | 使用：安装、连接、上手序列、执行计划、膨胀监控、建模实践、常见错误 |
| `05-related-tools.md` | 相关工具：与 MySQL / MongoDB 的分野、Redis、搜索引擎、扩展生态、选择规则 |
| `06-learning-path.md` | 学习路径：六阶段、练习与可观察完成检查 |
| `07-verification-record.md` | 验证记录：源定位 22 条、研究记录 41 条、未解决问题 3 条、审计记录 A-01…A-11、晋级审批 |

### 校验状态

| 项 | 结果 |
| :--- | :--- |
| 根源文件 `source.md` | **未被修改**（SHA-256 `c2fad4f2569fa6316d05fcbbe58d14585dd00750309c4455d42395854d763854`，与课程开始时逐字符一致） |
| 七个课程文件的逐条 SHA-256 与字节数 | 记录在 `course/../.course-staging-20261116T092500/MANIFEST.md`（暂存清单，不晋级） |
| 本地链接 / 锚点 | 0 处失效 |
| 证据标签 ↔ 底账记录 | 66 条记录全部闭合，无孤儿引用、无未使用记录 |
| 未解决断言 | 3 条，全部 Open 且已在正文降级标注 |
| 审校轮次 | 逐文件 review-only 审校 7/7 章 + 独立跨章节一致性复核 1 轮；共 30 项发现，全部处理 |

### 遗留事项

1. **U-01 / U-02 / U-03 三条未解决断言**保持 Open，需要人工提供可信来源或授权在可访问的基准环境下重新检索后才能闭合。
2. 课程正文中的所有执行计划输出、死元组数量、阈值数值等均为**示意值**，标注为 `Illustrative`；真实数值请以你自己的实例为准。
3. 本课程只覆盖 Linux 上的 PostgreSQL；容器化部署、云厂商托管服务（如 RDS）的差异不在源文档范围内，未作展开。

