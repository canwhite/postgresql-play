# 02 · 核心概念：把"全能"拆成可理解的几块

**学习目标**

读完之后，你应当能够：

1. 用一段话说清 MVCC 在 PostgreSQL 里到底做了什么，并解释它为什么必然带来 VACUUM 负担。
2. 说出 `json` 与 `jsonb` 的区别，以及 GIN 索引"索引的是什么"。
3. 知道 B-Tree / GIN / GiST / BRIN 各自适合什么场景，以及索引为什么会拖慢写入。
4. 知道执行计划（`EXPLAIN ANALYZE`）为什么是真正需要花时间的部分，以及它在哪一章展开（[04 章 4.4](04-usage.md)）。

前置：已读 [01 · 定位与背景](01-positioning-and-background.md)。本节的示例需要你有一台可连的 PostgreSQL 实例；没有的话可以只读概念，跳过"动手"标记的片段。

---

## 2.1 心智模型：一个内核，三层代价

源文档把 PostgreSQL 的整体性归结为"正确性"与"可扩展性"两个词（[S-02](07-verification-record.md#s-02)）。落到可操作的层面，这个内核可以拆成三层，每一层都自带一份账单：

> **示例标签**：Illustrative（本图为课程自建的讲解框架；其中"JSONB + GIN"依据 [R-04](07-verification-record.md#r-04)、[R-05](07-verification-record.md#r-05)，"扩展层"依据 [S-04](07-verification-record.md#s-04)）

```
┌──────────────────────────────────────────────────────┐
│  第 3 层 · 扩展层                                      │
│  JSONB + GIN / PostGIS / pgvector / TimescaleDB / Citus  │
│  账单：更多运维对象、更多升级路径、写放大                 │
├──────────────────────────────────────────────────────┤
│  第 2 层 · 表达层                                      │
│  窗口函数 / 递归 CTE / LATERAL / FULL OUTER JOIN         │
│  账单：执行计划复杂度上升，需要读懂执行计划                │
├──────────────────────────────────────────────────────┤
│  第 1 层 · 并发层（地基）                               │
│  MVCC：读不阻塞写，写不阻塞读                              │
│  账单：死元组 → 表膨胀 → 必须有 VACUUM                   │
└──────────────────────────────────────────────────────┘
```

**越往下越难绕过。** 第 3 层你可以不用（不装扩展就行），第 2 层你可以避开（不写复杂 SQL 就行），但第 1 层是所有写操作的地基——你没法"不用 MVCC"，因为 PostgreSQL 的并发控制是建立在 MVCC 之上的（[R-01](07-verification-record.md#r-01)）。

---

## 2.2 MVCC：读不阻塞写，写不阻塞读

源文档给出了这个特性的准确表述：

> PostgreSQL 的核心是 **MVCC（多版本并发控制）**，实现了"读不阻塞写，写不阻塞读"。
> — [Source-derived · S-03](07-verification-record.md#s-03)

官方文档给出了同样的机制解释：

> The main advantage of using the MVCC model of concurrency control rather than locking is that in MVCC locks acquired for querying (reading) data do not conflict with locks acquired for writing data, and so reading never blocks writing and writing never blocks reading.
> — [Research-derived · R-01](07-verification-record.md#r-01)

关键在于**"不冲突"是怎么做到的**：不是靠"读的时候给读事务一个共享锁、写的时候给写事务一个排他锁"，而是靠**每个事务看到自己那个时刻的快照**。写事务改一行，产生的不是一个"新值盖住旧值"，而是"多出一行版本"；读事务按自己的快照，挑它应该看到的那一版。

> **示例标签**：Research-derived（机制依据 R-01；下例为 Illustrative 的操作序列）
>
> 会话 A（读者）与会话 B（写者）同时进行：
>
> ```sql
> -- 会话 A
> BEGIN;
> SELECT * FROM accounts WHERE id = 1;      -- 读到 id=1，余额 100
> -- 此时会话 B 执行 UPDATE accounts SET balance = 200 WHERE id = 1; 并 COMMIT;
> -- 会话 B 的写不会被会话 A 阻塞
> -- 第二次再读，读到的是B提交之后的新版本
> SELECT * FROM accounts WHERE id = 1;      -- 会话 A 在 Read Committed 下重新读，会看到 200
> COMMIT;
> ```
>
> 这就是"读不阻塞写，写不阻塞读"的含义：**写者不需要等读者提交，读者在快照里也不会看到未提交的中间状态。**

**这里有一个重要的前提**：上面第二段 `SELECT` 在 Read Committed 下能看到 200，是因为 Read Committed 是 PostgreSQL 的默认隔离级别，并且"每条命令开始时取一个新快照"：

> Read Committed is the default isolation level in PostgreSQL.
> — [Research-derived · R-13](07-verification-record.md#r-13)

官方原文说明了这个行为：

> In effect, a SELECT query sees a snapshot of the database as of the instant the query begins to run.
>
> Also note that two successive SELECT commands can see different data, even though they are within a single transaction, if other transactions commit changes after the first SELECT starts and before the second SELECT starts.
> — [Research-derived · R-13](07-verification-record.md#r-13)

隔离级别本身在 2.7 节展开。

---

## 2.3 死元组与 VACUUM：MVCC 的账单

上一节的"多出一行版本"不是免费的。官方文档把机制说得很直白：

> In PostgreSQL, an UPDATE or DELETE of a row does not immediately remove the old version of the row.
>
> But eventually, an outdated or deleted row version is no longer of interest to any transaction. The space it occupies must then be reclaimed for reuse by new rows, to avoid unbounded growth of disk space requirements. This is done by running VACUUM.
> — [Research-derived · R-02](07-verification-record.md#r-02)

源文档用"死元组"这个中文术语描述同一件事，并直接给出了它的能力边界：

> 但它的实现方式很关键：**更新一行数据时，它不会原地覆盖，而是写入一个全新的行版本，旧版本留在数据块中成为"死元组"，靠后台的 `VACUUM` 进程回收。这直接决定了它的能力边界：它天生适合读多写少或读写均衡的负载，但极不适合作为高频更新的队列或任务表，因为高频 `UPDATE` 会产生海量死元组，拖垮 autovacuum。**
> — [Source-derived · S-03](07-verification-record.md#s-03)

**这段话是整篇源文档里因果链最完整的一句**，也是 [03 章能力边界](03-core-functionality.md)和 [05 章选型规则](05-related-tools.md)的共同依据。请把它拆成三段记住：

> **示例标签**：Illustrative（链条依据 [S-03](07-verification-record.md#s-03)、[R-02](07-verification-record.md#r-02)、[R-03](07-verification-record.md#r-03)）

```
高频 UPDATE
   ↓
每个 UPDATE 产生一个新版本 + 一个死元组（空间不立即回收）
   ↓
死元组累积 → 表膨胀 → autovacuum 追不上 → 空间持续增长
```

官方文档对 autovacuum 的定位是：

> PostgreSQL has an optional but highly recommended feature called autovacuum, whose purpose is to automate the execution of VACUUM and ANALYZE commands.
>
> In the default configuration, autovacuuming is enabled and the related configuration parameters are appropriately set.
> — [Research-derived · R-03](07-verification-record.md#r-03)

也就是说 autovacuum **默认开启**，但它是**按阈值触发**的，而不是"保证及时"。触发公式是：

> vacuum threshold = Minimum(vacuum max threshold, vacuum base threshold + vacuum scale factor * number of tuples)
> — [Research-derived · R-03](07-verification-record.md#r-03)

> **示例标签**：Research-derived（机制依据 R-02、R-03；下面的 SQL 与数值是示意）
>
> ⚠️ 注意第 1 步的 `INSERT` 不能省——空表上的 `UPDATE` 命中 0 行，不会产生任何行版本，也就没有死元组可观察。
>
> ```sql
> -- 1) 建一张会被高频更新的表
> CREATE TABLE counters (id int PRIMARY KEY, n int NOT NULL DEFAULT 0);
> INSERT INTO counters (id) VALUES (1);
>
> -- 2) 模拟队列式高频更新
> UPDATE counters SET n = n + 1 WHERE id = 1;   -- 重复执行数千次
> ```
>
> 观察死元组累积：
>
> ```sql
> SELECT relname, n_live_tup, n_dead_tup,
>        last_autovacuum, last_autoanalyze
> FROM pg_stat_user_tables
> WHERE relname = 'counters';
> ```
>
> `n_live_tup` 是存活行数估计，`n_dead_tup` 是死行数估计；`last_autovacuum` / `last_autoanalyze` 记录自动维护最近一次发生的时间。四个字段的官方定义见 [R-16](07-verification-record.md#r-16)。
>
> 想要更精确的物理层数字（死元组占用多少字节、空闲空间多少），可以启用扩展：
>
> ```sql
> CREATE EXTENSION IF NOT EXISTS pgstattuple;
> SELECT * FROM pgstattuple('counters');
> ```
>
> `pgstattuple` 会返回物理长度、存活元组比例、死元组数量与长度、空闲空间等。注意官方文档的提醒：**它只获取读锁，结果不是瞬时快照**（并发更新会影响结果）。
> — [Research-derived · R-17](07-verification-record.md#r-17)
>
> 示意输出（非实测，仅示意列的形状）：
>
> ```
>  table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count | dead_tuple_len | dead_tuple_percent | free_space | free_percent
> ```

**所以"VACUUM 意识"到底意味着什么？** 源文档的表述是：

> 因为 MVCC 的设计，**任何高频写入的表都需要关注表膨胀和 autovacuum 的状态**。这不是一个可以完全交给默认配置忽略的细节；在写入负载较高的系统中，调整 autovacuum 参数是运维的一部分。
> — [Source-derived · S-09](07-verification-record.md#s-09)

它是运维的一部分——但要记清楚它的作用范围：autovacuum 负责"回收空间"，不负责"把表变回最小尺寸"。官方文档区分得很清楚：

> The autovacuum daemon attempts to work this way, and in fact will never issue VACUUM FULL.
>
> Thus, moderately-frequent standard VACUUM runs are a better approach than infrequent VACUUM FULL runs for maintaining heavily-updated tables.
> — [Research-derived · R-03](07-verification-record.md#r-03)

---

## 2.4 JSONB 与 GIN：半结构化数据不用再开一个数据库

源文档把 JSONB 称为 PostgreSQL "最显著的差异化能力"：

> PostgreSQL 最显著的差异化能力是 **JSONB**。它以二进制格式存储，支持 **GIN 索引**，可以高效查询 JSON 内部的任意键值。这意味着你可以在一张表里同时使用严格的关系列和灵活的 JSONB 列，**不必为了半结构化数据再引入一个 MongoDB**。
> — [Source-derived · S-04](07-verification-record.md#s-04)

两句话，三个可查证的事实：

**事实一：`json` 与 `jsonb` 的差别是存储格式。**

> The `json` data type stores an exact copy of the input text, which processing functions must reparse on each execution; while `jsonb` data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead, but significantly faster to process, since no reparsing is needed. `jsonb` also supports indexing, which can be a significant advantage.
>
> In general, most applications should prefer to store JSON data as `jsonb`, unless there are quite specialized needs, such as legacy assumptions about ordering of object keys.
> — [Research-derived · R-04](07-verification-record.md#r-04)

注意一个容易被忽略的副作用：`jsonb` **不保留键的顺序，也不保留重复键**（只保留最后一个）。

**事实二：GIN 索引"索引的是键，不是值"。**

> GIN always stores and searches for keys, not item values per se.
>
> A GIN index stores a set of (key, posting list) pairs, where a posting list is a set of row IDs in which the key occurs. Each key value is stored only once, so a GIN index is very compact for cases where the same key appears many times.
> — [Research-derived · R-05](07-verification-record.md#r-05)

这是 GIN 的设计灵魂：它把"一个复合值里的每个元素"抽出来做倒排，所以同一个键只存一次，索引非常紧凑。

**事实三：`jsonb` 有两个算子类（operator class），支持的操作符不同。**

| 算子类 | 默认 | 可用操作符 |
| :--- | :--- | :--- |
| `jsonb_ops` | 是 | `@>`、`@?`、`@@`、`?`、`?|`、`?&` |
| `jsonb_path_ops` | 否 | `@>`、`@?`、`@@` |

四个常用符号可以这样记：

| 操作符 | 大白话 | 示例 |
| :--- | :--- | :--- |
| `@>` | 左边 JSON 是否包含右边这部分 | `attrs @> '{"color":"red"}'` |
| `?` | 顶层是否有这个键；若左边是 JSON 数组，则是否有这个字符串元素 | `attrs ? 'color'` |
| `@?` | 按 JSONPath 查找，只要找到一个匹配项就为真 | `attrs @? '$.sizes[*] ? (@ == "M")'` |
| `@@` | 执行 JSONPath 的判断表达式，结果为真才匹配 | `attrs @@ '$.color == "red"'` |

`?` 默认只看当前 JSON 的最外层；嵌套字段可用 `@>`，或用 `@?` / `@@` 写路径条件。

> Of the two operator classes for type jsonb, `jsonb_ops` is the default. `jsonb_path_ops` supports fewer operators but offers better performance for those operators.
> — [Research-derived · R-05](07-verification-record.md#r-05)

> **示例标签**：Research-derived（语法与算子类依据 R-04、R-05；样例数据为 Illustrative）
>
> （注意：这里的 `@@` 是 **jsonb** 的路径匹配操作符，与 [03 章 3.4 节](03-core-functionality.md) 中 tsvector/tsquery 的匹配操作符 `@@` 是两个不同的重载，索引互不相通。）
>
> ```sql
> CREATE TABLE products (
>   id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>   name  text NOT NULL,
>   price numeric(10,2) NOT NULL,
>   attrs jsonb NOT NULL DEFAULT '{}'::jsonb
> );
>
> -- 只关心"包含某个键值"的查询，用 jsonb_path_ops（更省空间、查询更快）
> CREATE INDEX products_attrs_gin ON products USING gin (attrs jsonb_path_ops);
>
> INSERT INTO products (name, price, attrs) VALUES
>   ('红色 T 恤', 99.00, '{"color":"red","sizes":["S","M","L"]}'),
>   ('蓝色帽子', 59.00, '{"color":"blue","sizes":["M"]}');
>
> -- 包含运算符 @> 走 GIN 索引
> SELECT id, name FROM products WHERE attrs @> '{"color":"red"}'::jsonb;
> ```
>
> 示意输出：
>
> ```
>  id |   name
> ----+----------
>   1 | 红色 T 恤
> ```
>
> ⚠️ 只有当 `jsonb_path_ops` 支持的操作符（`@>`、`@?`、`@@`）才能用上这个索引。要用 `?`、`?|`、`?&` 这类"键存在性"查询，得建 `jsonb_ops`（默认算子类）索引。

---

## 2.5 SQL 表达力：窗口函数、递归 CTE、LATERAL、FULL OUTER JOIN

源文档的判断是：

> 当你的查询涉及多表聚合、层次化递归或复杂的分析逻辑时，PostgreSQL 往往能用一条相对清晰的 SQL 表达，而不是把逻辑拆散到应用层。
> — [Source-derived · S-05](07-verification-record.md#s-05)

这四项都是 PostgreSQL 的原生能力，且都有官方定义。

**窗口函数**

> Window functions provide the ability to perform calculations across sets of rows that are related to the current query row.
>
> Aggregate functions act as window functions only when an OVER clause follows the call; otherwise they act as plain aggregates and return a single row for the entire set.
> — [Research-derived · R-09](07-verification-record.md#r-09)

**递归 CTE**

> The optional RECURSIVE modifier changes WITH from a mere syntactic convenience into a feature that accomplishes things not otherwise possible in standard SQL. Using RECURSIVE, a WITH query can refer to its own output.
> — [Research-derived · R-10](07-verification-record.md#r-10)

**LATERAL**

> Subqueries appearing in FROM can be preceded by the key word LATERAL. This allows them to reference columns provided by preceding FROM items. (Without LATERAL, each subquery is evaluated independently and so cannot cross-reference any other FROM item.)
> — [Research-derived · R-11](07-verification-record.md#r-11)

**FULL OUTER JOIN**

> FULL OUTER JOIN returns all the joined rows, plus one row for each unmatched left-hand row (extended with nulls on the right), plus one row for each unmatched right-hand row (extended with nulls on the left).
> — [Research-derived · R-12](07-verification-record.md#r-12)

> **示例标签**：Illustrative（语法依据 R-09、R-10、R-11、R-12；样例数据与表结构为 Illustrative）
>
> ⚠️ 本例假设的表结构是 `employees(id, manager_id, name)`、`customers(id, name, department_id)`、`orders(id, customer_id, amount, created_at)`，与 [04 章 4.3](04-usage.md) 的示例 schema 不同（那里是 `orders(id, customer_id, product_id, amount, status, created_at)`、`customers(id, email, name, created_at)`，且没有 `employees` / `department_id`）。跑本例前请先按上述结构建表。
>
> **窗口函数**——给每个客户按订单金额排名：
>
> ```sql
> -- 原表什么都没变，加了个rnk列
> SELECT customer_id, id AS order_id, amount,
>        RANK() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rnk
> FROM orders;
> ```
>
> **递归 CTE(common table expression)**——沿组织树向下展开（自引用需要 `RECURSIVE`）：
>
> ```sql
> WITH RECURSIVE org AS (
>   -- ① 锚点（anchor）：递归的起点
>   SELECT id, manager_id, name
>   FROM employees
>   WHERE id = 1                       -- 锚点：起点
>   -- UNION ALL 把锚点的结果集和递归项的结果集拼起来。这是"上下"两个分支的关系，UNION会去重
>   UNION ALL
>   -- ② 递归项（recursive term）：拿上一步的结果去找下一层
>   SELECT e.id, e.manager_id, e.name
>   FROM employees e
>   JOIN org o ON e.manager_id = o.id  -- 递归项：引用自身输出
> )
> SELECT * FROM org;
> ```
>
> **FULL OUTER JOIN**——两张表都要保留未匹配行（缺失侧补 NULL）：
>
> ```sql
> SELECT c.id AS customer_id, o.id
> FROM customers c
> FULL OUTER JOIN orders o ON o.customer_id = c.id;
> ```
>
> **LATERAL**——对每个客户取金额最大的 3 笔订单（子查询引用同一 FROM 列表中**前面**的项）：
>
> ```sql
> SELECT c.id, t.order_id, t.amount
> FROM customers c
> CROSS JOIN LATERAL (
>   SELECT o.id AS order_id, o.amount
>   FROM orders o
>   WHERE o.customer_id = c.id
>   ORDER BY o.amount DESC
>   LIMIT 3
> ) t;
> ```

---

## 2.6 索引体系：对症下药，代价是写放大

源文档列举了非 B-Tree 的几类索引：

> 除了常规 B-Tree，它的 **GIN**（JSONB、数组、全文检索）、**GiST**（地理空间）、**BRIN**（超大有序表）等索引类型，让特定场景的查询可以做到"对症下药"。但索引越多，写放大的代价也越大，这是一个需要主动管理的权衡。
> — [Source-derived · S-06](07-verification-record.md#s-06)

逐个对照官方定义：

| 索引类型 | 官方定义要点 | 证据 |
| :--- | :--- | :--- |
| B-Tree | 处理可排序数据上的等值与范围查询；也可用于 `LIKE 'foo%'` 这类锚定前缀的模式匹配，但不能用于 `LIKE '%bar'` | [R-06](07-verification-record.md#r-06) |
| Hash | 存 32 位哈希码，**只能处理简单等值比较** | [R-06](07-verification-record.md#r-06) |
| GiST | 不是单一索引，而是一种可承载多种索引策略的**基础设施**；标准发行版含二维几何算子类，支持最近邻搜索 | [R-06](07-verification-record.md#r-06) |
| SP-GiST | 类似 GiST，支持四叉树、k-d 树、基数树等非平衡结构 | [R-06](07-verification-record.md#r-06) |
| GIN | "倒排索引"，适合**含多个组成部分的值**，如数组 | [R-06](07-verification-record.md#r-06) |
| BRIN | 存储**连续物理块范围**上的值摘要，只对"取值与物理顺序高度相关"的列最有效 | [R-06](07-verification-record.md#r-06) |

> BRIN indexes (a shorthand for Block Range INdexes) store summaries about the values stored in consecutive physical block ranges of a table. Thus, they are most effective for columns whose values are well-correlated with the physical order of the table rows.
> — [Research-derived · R-06](07-verification-record.md#r-06)

⚠️ **一处需要修正源文档措辞的地方**：源文档写 BRIN 适用于"超大有序表"（[S-06](07-verification-record.md#s-06)）。官方原文强调的是"取值与物理顺序相关"，**没有**把"表很大"写成前提条件。这是本课程在审校中记录的一处措辞修正（见 [07 章审计记录 A-03](07-verification-record.md#a-03)、[R-06](07-verification-record.md#r-06)）。

**写放大为什么必然发生？** 因为索引本身也是要维护的数据结构。每次 `INSERT`/`UPDATE`/`DELETE` 都要同步更新所有相关索引，所以索引越多，单次写入越慢、空间占用越大。GIN 尤其敏感：

> Insertion into a GIN index can be slow due to the likelihood of many keys being inserted for each item.
>
> For bulk insertions into a table it is advisable to drop the GIN index and recreate it after finishing bulk insertion.
> — [Research-derived · R-05](07-verification-record.md#r-05)

**全文检索用的是同一套 GIN 机制**：

> GIN indexes are the preferred text search index type. As inverted indexes, they contain an index entry for each word (lexeme), with a compressed list of matching locations.
> — [Research-derived · R-07](07-verification-record.md#r-07)

---

## 2.7 事务与隔离级别

源文档把"理解事务边界"列在入门路径里：

> 入门路径很直接：建库、建表、写入、查询、加索引、理解事务边界。真正需要花时间的是**执行计划（`EXPLAIN ANALYZE`）和索引策略**，而不是 SQL 语法本身。
> — [Source-derived · S-10](07-verification-record.md#s-10)

SQL 标准定义了四级隔离级别，其中最严格的是 Serializable：

> The SQL standard defines four levels of transaction isolation. The most strict is Serializable, which is defined by the standard in a paragraph which says that any concurrent execution of a set of Serializable transactions is guaranteed to produce the same effect as running them one at a time in some order.
>
> In PostgreSQL, you can request any of the four standard transaction isolation levels, but internally only three distinct isolation levels are implemented, i.e., PostgreSQL's Read Uncommitted mode behaves like Read Committed.
> — [Research-derived · R-13](07-verification-record.md#r-13)

| 级别 | 官方描述要点 | 证据 |
| :--- | :--- | :--- |
| Read Uncommitted | 在 PostgreSQL 内部**行为等同 Read Committed**（只实现了三种不同的隔离级别） | [R-13](07-verification-record.md#r-13) |
| Read Committed | **默认级别**。语句看到的是该语句开始时的快照 | [R-13](07-verification-record.md#r-13) |
| Repeatable Read | 事务开始后已提交的数据可见；事务执行期间并发事务的提交**不可见**。实现上等同快照隔离 | [R-13](07-verification-record.md#r-13) |
| Serializable | 最严格，通过在快照隔离之上加序列化异常检查实现 | [R-13](07-verification-record.md#r-13) |

要用非默认级别（`SET TRANSACTION` 必须写在事务块内）：

> **示例标签**：Research-derived（命令与语义依据 R-13）
>
> ```sql
> BEGIN;
> SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
> -- 此后的事务内语句按 SERIALIZABLE 执行
> COMMIT;
> ```

> **示例标签**：Illustrative（场景为课程构造；Read Committed 的语义依据 R-13）
>
> 一个"队列式认领"在 Read Committed 下的典型问题：
>
> ```sql
> -- 会话 A 和会话 B 都执行
> BEGIN;
> UPDATE jobs SET status = 'running'
>  WHERE id = (SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1)
>  RETURNING id;
> -- 会话 A 抢到 id=1；会话 B 的 SELECT 子查询看不见 A 未提交的更新，
> -- 会话 B 会再次看到 id=1 为 pending，然后它的 UPDATE 会阻塞在行锁上。
> -- A 提交后，B 解除阻塞并重新检查条件——此时条件只剩 id = 1，仍然成立，
> -- 于是 B 又把 id=1 更新了一遍（又产生一个死元组）：两个会话都"认领"了同一个任务。
> COMMIT;
> ```
>
> 这个例子说明：**"用一张表当队列"的正确性问题和性能问题是两个独立问题**。把队列表搬去 Redis 解决的是性能问题（以及高频 `UPDATE` 带来的死元组问题）（[S-12](07-verification-record.md#s-12)、[S-03](07-verification-record.md#s-03)）；但如果你的业务真的需要"恰好认领一次"的严格语义，光换存储并不够，还得把认领逻辑本身写对。

---

## 2.8 术语速查表

| 术语 | 一句话解释 | 证据 |
| :--- | :--- | :--- |
| MVCC | 多版本并发控制：读不阻塞写，写不阻塞读 | [R-01](07-verification-record.md#r-01) |
| 死元组 | `UPDATE`/`DELETE` 留下的旧行版本，空间待回收 | [R-02](07-verification-record.md#r-02) |
| 表膨胀 | 死元组累积导致表占用的空间远大于有效数据 | [R-02](07-verification-record.md#r-02) |
| autovacuum | 默认开启的后台守护进程，自动执行 `VACUUM` / `ANALYZE` | [R-03](07-verification-record.md#r-03) |
| vacuum threshold | `Minimum(最大阈值, 基数阈值 + 比例阈值 × 行数)`，决定何时触发 vacuum | [R-03](07-verification-record.md#r-03) |
| JSONB | 以分解二进制格式存储的 JSON 类型，支持索引 | [R-04](07-verification-record.md#r-04) |
| 算子类 | 决定索引支持哪些操作符的集合，如 `jsonb_ops` / `jsonb_path_ops` | [R-05](07-verification-record.md#r-05) |
| 倒排索引 | 把"值里包含哪些键"反过来存成"键 → 行 ID 列表"，GIN 的核心机制 | [R-05](07-verification-record.md#r-05) |
| tsvector | 预处理后的文档表示，元素是已归一化的词元（lexeme） | [R-08](07-verification-record.md#r-08) |
| 写放大 | 每类索引都要在写入时同步维护，索引越多写越慢 | [S-06](07-verification-record.md#s-06) |
| 快照 | 语句/事务在某一时刻看到的数据库状态视图 | [R-13](07-verification-record.md#r-13) |

---

## 2.9 本节完成检查

- [ ] 我能解释：为什么 PostgreSQL 的 `UPDATE` 会产生死元组，以及这和 autovacuum 的关系。
- [ ] 我能说出 `json` 和 `jsonb` 的至少两个区别，并解释为什么官方建议默认用 `jsonb`。
- [ ] 我能解释 GIN 为什么"很紧凑"，以及 `jsonb_ops` 和 `jsonb_path_ops` 该怎么选。
- [ ] 我能说出 B-Tree / GIN / GiST / BRIN 各自适合的场景，并解释写放大从哪来。
- [ ] 我能说出 PostgreSQL 的默认隔离级别，并解释"用表当队列"为什么同时有正确性和性能两个问题。
