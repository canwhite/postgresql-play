# 03 · 核心能力：能做什么、不能做什么、输入输出是什么

**学习目标**

读完之后，你应当能够：

1. 对每一项核心能力说出它的**输入**、**输出**和**限制**，而不只是"它支持 XX"。
2. 至少手写并跑通三段可观察的 SQL：JSONB 条件查询、窗口函数分析、全文检索。
3. 用一句话复述 PostgreSQL 四条明确的"不擅长"，并说明每条为什么。
4. 知道自己上一条需求属于"能力范围内"还是"边界之外"。

前置：已读 [02 · 核心概念](02-core-concepts.md)。

---

## 3.1 能力总览

下表把源文档分散在各处的特性收拢成一张可查的表。"输入/输出"指这项能力在数据处理链路上吃掉什么、吐出什么。

| 能力 | 输入 | 输出 | 主要限制 | 证据 |
| :--- | :--- | :--- | :--- | :--- |
| MVCC 并发控制 | 读事务、写事务 | 互不阻塞的读/写 | 高频 `UPDATE` 产生死元组，需 VACUUM 回收 | [S-03](07-verification-record.md#s-03)、[R-01](07-verification-record.md#r-01) |
| JSONB 半结构化存储 | JSON 文本/对象 | 二进制存储、可索引的文档 | 不保留键顺序、丢弃重复键（只留最后一个） | [S-04](07-verification-record.md#s-04)、[R-04](07-verification-record.md#r-04) |
| GIN 索引 | 含多个成分的值 | 倒排的"键 → 行 ID 列表" | 插入较慢；算子类决定可用操作符 | [S-04](07-verification-record.md#s-04)、[R-05](07-verification-record.md#r-05)、[R-06](07-verification-record.md#r-06) |
| SQL 表达完整性 | 多表数据 | 一条 SQL 完成聚合/递归/分析 | 复杂查询的执行计划需要读懂（[S-10](07-verification-record.md#s-10)） | [S-05](07-verification-record.md#s-05)、[R-09](07-verification-record.md#r-09) |
| 多类索引 | 表 + 列 | 针对不同访问模式的查找结构 | 索引越多，写放大越大 | [S-06](07-verification-record.md#s-06)、[R-06](07-verification-record.md#r-06) |
| 全文检索 | 文档文本 | 预处理的 `tsvector` + 匹配结果 | 中小规模够用，大规模高并发有上限 | [S-15](07-verification-record.md#s-15)、[R-07](07-verification-record.md#r-07)、[R-08](07-verification-record.md#r-08) |
| 扩展生态 | 扩展提供的类型/算子 | 地理、向量、时序、分布式能力 | 多一个组件 = 多一条升级与运维路径 | [S-04](07-verification-record.md#s-04)、[R-31](07-verification-record.md#r-31)、[R-32](07-verification-record.md#r-32)、[R-33](07-verification-record.md#r-33)、[R-34](07-verification-record.md#r-34) |

---

## 3.2 能力一：半结构化数据（JSONB）

**输入**：一个 JSON 值。
**输出**：一份可被关系列并存、可被 GIN 索引的文档。

官方对 `jsonb` 的定义性描述已在 [02 章 2.4 节](02-core-concepts.md)给出（[R-04](07-verification-record.md#r-04)）。这里补上**可观察的行为**。

> **示例标签**：Research-derived（语法与行为依据 R-04、R-05；样例数据为 Illustrative）
>
> ```sql
> SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::json;
> -- {"bar": "baz", "balance": 7.77, "active":false}
>
> SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::jsonb;
> -- {"bar": "baz", "active": false, "balance": 7.77}
> ```
>
> 这两行**直接来自官方文档**（[R-04](07-verification-record.md#r-04)），是可以逐字对照的输出：注意 `jsonb` 重新排了键的顺序，并把 `"active":false` 规范成 `"active": false`（冒号后补空格），键之间统一用 `", "` 分隔。
>
> 重复键的处理同样来自官方文档，可以直接跑这两行对照：
>
> ```sql
> SELECT '{"a":1,"a":2}'::json;    -- json 保留所有重复键（处理函数把最后一个视为生效值）
> SELECT '{"a":1,"a":2}'::jsonb;   -- jsonb 只保留最后一个值
> ```
>
> **查询 JSONB 内部**：三种常用方式
>
> ```sql
> -- 1) 包含：走 jsonb_path_ops 索引
> SELECT id FROM products WHERE attrs @> '{"color":"red"}'::jsonb;
>
> -- 2) 提取标量
> SELECT attrs ->> 'color' AS color, attrs -> 'sizes' AS sizes FROM products;
>
> -- 3) JSONPath 存在性
> SELECT id FROM products WHERE attrs @? '$.sizes[*] ? (@ == "L")';
> ```
>
> 其中 `@>`、`@?`、`@@` 是 `jsonb_path_ops` 支持的三个操作符（[R-05](07-verification-record.md#r-05)），所以上面 1 和 3 都能用上该算子类的索引。
>
> （注意：这里说的 `@@` 是 **jsonb** 的路径匹配操作符；3.4 节中 tsvector/tsquery 的匹配操作符 `@@` 是另一个重载，两者用的不是同一种索引。）
>
> （本例用到的 `products` 表见 [02 章 2.4 节](02-core-concepts.md) 与 [04 章 4.3](04-usage.md)。）

**限制**：官方文档指出 "In general, most applications should prefer to store JSON data as `jsonb`, unless there are quite specialized needs, such as legacy assumptions about ordering of object keys."（[R-04](07-verification-record.md#r-04)）。反过来说，**如果你需要保留键顺序或原始文本格式，就不能用 `jsonb`**——那是一个真实但少见的限制。

---

## 3.3 能力二：分析型查询（窗口函数 / 递归 CTE / LATERAL / FULL OUTER JOIN）

**输入**：一张或多张表。
**输出**：不经过应用层拼装的结果集。

这一组能力的完整语法示例在 [02 章 2.5 节](02-core-concepts.md)已经逐个给出并逐条对应官方定义。这里补上一个**组合使用**的例子，因为真实分析查询往往同时用到其中两三个。

> **示例标签**：Illustrative（语法组合依据 R-09、R-10、R-35；样例数据与表结构为 Illustrative）
>
> ⚠️ 本例假设的表结构是 `departments(id, manager_id, name)`、`customers(id, name, department_id)`、`orders(id, customer_id, amount, created_at)`，与 [04 章 4.3](04-usage.md) 的示例 schema 不同（那里是 `orders(id, customer_id, product_id, amount, status, created_at)`、`customers(id, email, name, created_at)`，且没有 `departments` / `department_id`）。跑本例前请先按上述结构建表。
>
> 需求：给每个客户算出**金额最高的 3 笔**订单的金额合计，并在客户管理树上找到该客户所属的部门。
>
> ```sql
> WITH RECURSIVE org AS (
>   SELECT id, manager_id, name FROM departments WHERE id = 10
>   UNION ALL
>   SELECT d.id, d.manager_id, d.name
>   FROM departments d JOIN org o ON d.manager_id = o.id
> ),
> recent AS (
>   SELECT o.customer_id, o.id AS order_id, o.amount,
>          ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.amount DESC, o.created_at DESC) AS rnk
>   FROM orders o
> )
> SELECT c.id AS customer_id, c.name,
>       (SELECT d.name FROM org d WHERE d.id = c.department_id) AS dept,
>       SUM(r.amount) FILTER (WHERE r.rnk <= 3) AS top3_amount
> FROM customers c
> LEFT JOIN recent r ON r.customer_id = c.id
> GROUP BY c.id, c.name, c.department_id;
> ```
>
> 说明：
> - `WITH RECURSIVE` 同时给出锚点与递归项，递归项引用 `org` 自身的输出（[R-10](07-verification-record.md#r-10)）。
> - 本例的排名发生在 `recent` 这个 CTE **内部**，因此分区排名在外层 `GROUP BY` 之前就已完成（一般顺序是 `GROUP BY → HAVING → 窗口函数 → ORDER BY`，见 [R-09](07-verification-record.md#r-09)）。
> - 这里用 `ROW_NUMBER()` 而不是 `RANK()`：并列金额时 `RANK()` 会给相同的名次，`rnk <= 3` 可能选出多于 3 行；`ROW_NUMBER()` 保证恰好 3 行。
> - `FILTER (WHERE ...)` 是聚合上的过滤子句——只有 `filter_clause` 为真的输入行会被喂给聚合函数（[R-35](07-verification-record.md#r-35)）。
> - `LEFT JOIN` 保留了没有订单的客户。
>
> ⚠️ **示意输出**，具体数字取决于你的数据；本课程不主张任何具体返回值。

---

## 3.4 能力三：全文检索

**输入**：一段文档文本。
**输出**：一个 `tsvector`（预处理后的文档），以及查询 `tsquery` 对它的匹配结果。

官方定义：

> A data type tsvector is provided for storing preprocessed documents, along with a type tsquery for representing processed queries.
>
> For text search purposes, each document must be reduced to the preprocessed tsvector format. Searching and ranking are performed entirely on the tsvector representation of a document — the original text need only be retrieved when the document has been selected for display to a user.
>
> Full text searching in PostgreSQL is based on the match operator `@@`, which returns true if a tsvector (document) matches a tsquery (query).
> — [Research-derived · R-08](07-verification-record.md#r-08)

**这里有一个非常容易踩的坑，官方文档专门给了例子**：匹配操作是发生在**词元（lexeme）**上的，不是原始单词上。

> **示例标签**：Research-derived（示例与结论均直接来自官方文档 R-08）
>
> ```sql
> SELECT to_tsvector('fat cats ate fat rats') @@ to_tsquery('fat & rat');
> --  t
>
> SELECT 'fat cats ate fat rats'::tsvector @@ to_tsquery('fat & rat');
> --  f
> ```
>
> 为什么第二个是 `f`？官方原文的解释是：
>
> The elements of a tsvector are lexemes, which are assumed already normalized, so rats does not match rat.
>
> 第一行经过 `to_tsvector` 做了词元化归一（`rats` → `rat`），所以能匹配；第二行是直接把原始字符串强制转换成 `tsvector`，**跳过了归一化**，于是 `rats` 匹配不上 `rat`。
>
> ⚠️ 上面第一行的 `t` 依赖服务器的 `default_text_search_config` 是英文系配置（如 `english`）；若你的服务器用的是 `simple`（不做词干化），第一行同样会返回 `f`。要得到可复现的结果，请显式写配置：`to_tsvector('english', ...)` / `to_tsquery('english', ...)`（[R-08](07-verification-record.md#r-08)）。
>
> **结论**：全文检索必须用 `to_tsvector('english', ...)` 生成列值（或建生成列/触发器），不要直接 `::tsvector` 强转原始文本（机制依据 [R-08](07-verification-record.md#r-08)；生成列写法见 [R-41](07-verification-record.md#r-41)）。

**索引**：GIN 是首选。

> GIN indexes are the preferred text search index type. As inverted indexes, they contain an index entry for each word (lexeme), with a compressed list of matching locations.
> — [Research-derived · R-07](07-verification-record.md#r-07)

官方还说明了 GiST 的差异：GiST 是**有损的**（"the index might produce false matches"），必须回表检查实际行来排除假匹配（[R-07](07-verification-record.md#r-07)）。

> **示例标签**：Research-derived（建索引与生成列语法依据 R-41；`'english'` 这个配置名按你的语言环境选择）
>
> ⚠️ 本例假设已有一张 `articles(id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, title text, body text)`；跑本例前请先建表（字段只需 `title`、`body` 与主键 `id`）。
>
> ```sql
> -- 生成列 + GIN 索引（官方推荐的写法：写入时自动维护）
> ALTER TABLE articles ADD COLUMN tsv tsvector
>   GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) STORED;
>
> CREATE INDEX articles_tsv_gin ON articles USING gin (tsv);
>
> -- 查询
> SELECT id, title
> FROM articles
> WHERE tsv @@ to_tsquery('english', 'postgres & vacuum')
> LIMIT 10;
> ```
>
> 一个"文本搜索配置"是把解析器与一组词典绑在一起、处理解析器输出的词元的东西（[R-40](07-verification-record.md#r-40)）。官方示例用 `ORDER BY last_mod_date DESC` 排序；按相关度排序需要排名函数（本课程未就其签名建立研究记录，此处不展开）。

---

## 3.5 明确的边界：四条"不要"

源文档用了一整节讲边界，并且给出了一个反直觉的判断：

> PostgreSQL 的边界比它的能力更值得记住，因为很多问题源于**把它用在它设计上就不擅长的场景**。
> — [Source-derived · S-11](07-verification-record.md#s-11)

### ❌ 不要把它当消息队列

> 高频轮询、认领、更新任务状态，会产生海量死元组，让 autovacuum 彻底跟不上。Stripe 和 GitLab 都公开复盘过这类问题，最终将队列负载迁移到 Redis 或专业消息中间件。
> — [Source-derived · S-12](07-verification-record.md#s-12)

**机制**（已在 [02 章 2.3 节](02-core-concepts.md)推导过）：认领任务 = 高频 `UPDATE` 状态字段 = 每个 `UPDATE` 产生一个死元组 = 表膨胀 = autovacuum 追不上。

**外部佐证（部分）**：GitLab 的官方开发文档确认其后台作业系统用 Sidekiq，作业与参数存在 Redis 中：

> We use Sidekiq as our background job processor.
>
> GitLab stores Sidekiq jobs and their arguments in Redis.
> — [Research-derived · R-27](07-verification-record.md#r-27)

⚠️ **但请注意证据边界**：源文档同句中的 **Stripe 部分，本课程未能找到可核实的一手材料**（检索尝试与失败原因见 [07 章未解决问题 U-02](07-verification-record.md#u-02)）。因此本课程只把"GitLab 用 Redis 跑后台作业"当作**已核实**的事实陈述，把"Stripe 迁移队列负载"当作**源文档主张**保留，不作为论据使用。

### ❌ 不要指望它替代 OLAP 数据仓库

> PostgreSQL 适合**中大规模**数据，但**不适合 PB 级、超高频、超廉价存储的大数据分析场景**。列式压缩、大规模并行处理不是它的主战场。
> — [Source-derived · S-13](07-verification-record.md#s-13)

这条属于**架构定位**而非某个具体功能开关：PostgreSQL 是行式存储、单机（或通过 Citus 分布式扩展，见 [R-34](07-verification-record.md#r-34)）执行；列式压缩与大规模并行处理是另一类系统的设计目标（[S-13](07-verification-record.md#s-13)）。

### ❌ 不要把它当作唯一的"万能锤"

> 当系统从几千用户增长到百万级时，把缓存、队列、分析、向量搜索全部塞进一个 PostgreSQL 实例，会从务实的选择变成架构隐患。单实例的写放大、索引膨胀和 VACUUM 压力会逐渐成为瓶颈。
> — [Source-derived · S-14](07-verification-record.md#s-14)

⚠️ 注意这里的"几千 → 百万"是源文档作者的经验阈值，**本课程没有找到任何可引用的一手来源支撑这个具体数字**（检索结论见 [07 章未解决问题 U-03](07-verification-record.md#u-03)）。请把它当作一个**方向性的提醒**（单一实例的职责会随规模增长而失控），而不是一条精确的容量水位线。具体的替代方案见 [05 章](05-related-tools.md)。

### ⚠️ 全文检索有能力上限

> 内置的 `tsvector` 全文检索在中小规模够用，但面对大规模、高并发的搜索需求，专用搜索引擎（如 Elasticsearch/OpenSearch）在功能深度和性能上仍然更合适。
> — [Source-derived · S-15](07-verification-record.md#s-15)

对照官方文档可以确认前半句的机制基础：内置全文检索的匹配发生在 `tsvector` 上，且 GIN 索引"每个词元一条索引项"（[R-07](07-verification-record.md#r-07)、[R-08](07-verification-record.md#r-08)）——这是典型的倒排检索结构；而 GIN 的写入成本本身也不低（[R-05](07-verification-record.md#r-05)：插入慢、批量插入建议先删索引再重建）。规模上去之后，索引体积与写入成本都会显著上升。

---

## 3.6 能力判定表：把它用在这一行的哪一侧

| 你的需求 | 判定 | 依据 |
| :--- | :--- | :--- |
| 强一致的事务型写入 + 复杂关联查询 | ✅ 在能力范围内 | [S-05](07-verification-record.md#s-05)、[R-13](07-verification-record.md#r-13) |
| 半结构化字段 + 需要按内部键过滤 | ✅ 在能力范围内 | [S-04](07-verification-record.md#s-04)、[R-05](07-verification-record.md#r-05) |
| 层次结构 / 图遍历式查询 | ✅ 在能力范围内（递归 CTE） | [S-05](07-verification-record.md#s-05)、[R-10](07-verification-record.md#r-10) |
| 地理 / 向量 / 时序（且与关系数据强关联） | ✅ 在能力范围内（需装扩展） | [S-04](07-verification-record.md#s-04)、[R-31](07-verification-record.md#r-31)、[R-32](07-verification-record.md#r-32)、[R-33](07-verification-record.md#r-33) |
| 中小规模站内搜索 | ✅ 够用 | [S-15](07-verification-record.md#s-15)、[R-07](07-verification-record.md#r-07) |
| 高频更新的任务队列 / 计数器 | ❌ 边界之外 | [S-12](07-verification-record.md#s-12)、[R-02](07-verification-record.md#r-02) |
| PB 级列式分析 / 大规模并行 | ❌ 边界之外 | [S-13](07-verification-record.md#s-13) |
| 大规模高并发全文检索 | ⚠️ 到达上限，考虑专用引擎 | [S-15](07-verification-record.md#s-15) |
| 一个实例同时扛缓存 + 队列 + 分析 + 向量 | ⚠️ 随规模增长成为隐患 | [S-14](07-verification-record.md#s-14) |

---

## 3.7 本节完成检查

- [ ] 我能对 JSONB、窗口函数、全文检索分别说出"输入 / 输出 / 限制"。
- [ ] 我亲手跑过 3.2 的 `json` vs `jsonb` 对比，并能指出两处差异（键顺序、重复键）。
- [ ] 我能解释为什么 `SELECT '...text...'::tsvector @@ to_tsquery('fat & rat')` 返回 `f`，而经过 `to_tsvector` 就返回 `t`。
- [ ] 我能不看原文说出四条边界，并给每条指出机制层面的原因。
- [ ] 我知道源文档中哪些断言**没有**得到外部证据支持（见 3.5 与 [07 章](07-verification-record.md#未解决问题)）。
