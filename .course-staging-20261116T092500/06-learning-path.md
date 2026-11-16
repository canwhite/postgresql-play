# 06 · 学习路径：从零到能独立做选型判断

**学习目标**

读完之后，你应当能够：

1. 按顺序走完六个阶段，每阶段都有明确的产出物。
2. 为每个阶段说出**可观察的完成检查**——也就是"做完了"能被别人验证，而不是"我觉得会了"。
3. 知道哪些阶段可以跳过、哪些绝对不能跳。

---

## 6.1 前置知识清单

进入第 1 阶段之前，你应当已经具备：

| 前置 | 最低要求 | 为什么需要 |
| :--- | :--- | :--- |
| SQL 基础 | 会写 `SELECT ... FROM ... WHERE ...`、`JOIN`、`GROUP BY`、`ORDER BY` | 第 2、3 阶段的练习全靠它 |
| 终端与包管理器 | 能在 Linux 上装软件、看进程、翻日志 | 第 2 阶段要装 PostgreSQL |
| 基本数据建模 | 知道"表 / 行 / 列 / 主键 / 索引"是什么 | 第 2 阶段建表、第 3 阶段建索引时要用 |
| 一门后端语言 | 任选（Python / Go / Java / Node…） | 可选：阶段 4 若想把第 1 步写成参数化查询再执行，才用得上 |

**不需要**预先懂：MVCC、执行计划、索引算法、扩展机制——这些正是本课程要教的东西。

**如果你完全没有 SQL 基础**：先补 SQL 基础再回来，否则第 1 阶段的练习会变成抄写而非理解。

---

## 6.2 六个阶段

> **示例标签**：Illustrative（阶段与章节的对应关系依据本课程各章的实际标题）

```
阶段 1  建立判断框架      →  01 + 02 章
阶段 2  装上并跑起来      →  04.1–04.3
阶段 3  亲手验证机制      →  03 章（3.2 JSONB / 3.4 全文检索）+ 02 章 2.4（GIN 与算子类）
阶段 4  学会看执行计划    →  04.4
阶段 5  学会看膨胀与运维  →  04.5 + 02 章 2.3
阶段 6  做一次真实选型    →  05 章
```

---

### 阶段 1 · 建立判断框架

**读**：[01 · 定位与背景](01-positioning-and-background.md) → [02 · 核心概念](02-core-concepts.md)

**练**：把 01 章 1.6 的电商场景写下来，用自己的话回答"如果只允许一个数据库，你选什么？"先写下你的**第一反应**，不要急着看答案。

**可观察的完成检查**：

- [ ] 你能默写出 PostgreSQL 的核心定位，以及它的代价（不看原文）。
- [ ] 你能画出一个从 `UPDATE` 到"表膨胀"的因果链，并且链条上有 `VACUUM` 和 `autovacuum` 两个词。
- [ ] 你能说出 `json` 与 `jsonb` 的两个区别。
- [ ] 你能说出 B-Tree / GIN / GiST / BRIN 各自适合的场景。
- [ ] 你的答案里**没有**出现"PostgreSQL 更快/更慢"这种无法定位的表述。

---

### 阶段 2 · 装上并跑起来

**读**：[04 · 使用](04-usage.md) 的 4.1–4.3

**练**：

1. 在你的机器上安装 PostgreSQL（Debian/Ubuntu 或 Red Hat 系皆可）。
2. 连上 `psql`，执行 `SELECT version();`。
3. 建一个 `practice` 库，建三张表：`customers` / `products` / `orders`（结构见 04.3）。
4. 插入至少 20 条订单数据。

**可观察的完成检查**：

- [ ] `psql -c "SELECT version()"` 有输出且无报错。
- [ ] `\l` 能看到你建的 `practice` 库。
- [ ] `\d orders` 能看到你建的表结构。
- [ ] `SELECT count(*) FROM orders;` 返回 ≥ 20。
- [ ] 你能说出服务监听在哪个端口，以及 `port` 参数改动后需要做什么（重启）。（依据 [Research-derived · R-15](07-verification-record.md#r-15)）

---

### 阶段 3 · 亲手验证机制

**读**：[03 · 核心能力](03-core-functionality.md) 的 3.2 / 3.4 → [02 · 核心概念](02-core-concepts.md) 的 2.4

**练**（三个实验，逐个做，不要跳过）：

**实验 A：JSONB 的行为差异**

> **示例标签**：Research-derived（键顺序与重复键行为依据 R-04；样例数据为 Illustrative）

```sql
SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::json;
SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::jsonb;
```
预期：两行输出的键顺序不同。再跑一次 `jsonb` 对重复键的处理：
```sql
SELECT '{"a":1,"a":2}'::json;
SELECT '{"a":1,"a":2}'::jsonb;
```

**实验 B：全文检索的词元陷阱**

> **示例标签**：Research-derived（示例与结论依据 R-08）

```sql
SELECT to_tsvector('english', 'fat cats ate fat rats') @@ to_tsquery('english', 'fat & rat');   -- 期望 t
SELECT 'fat cats ate fat rats'::tsvector @@ to_tsquery('english', 'fat & rat');                 -- 期望 f
```
预期：第一行 `t`，第二行 `f`。想清楚为什么。
⚠️ 显式写出 `'english'` 是为了保证结果可复现：单参数形式用的是服务器的 `default_text_search_config`，若它不是英文系配置（例如 `simple`），第一行也会返回 `f`（[R-08](07-verification-record.md#r-08)）。

**实验 C：GIN 索引是否被用上**

> **示例标签**：Illustrative（算子类与可索引操作符依据 R-05；表结构、样例数据与执行计划输出均为示意）

```sql
CREATE TABLE t_docs (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY, attrs jsonb);
INSERT INTO t_docs (attrs) SELECT jsonb_build_object('k', i::text, 'n', i) FROM generate_series(1,5000) i;
ANALYZE t_docs;
CREATE INDEX t_docs_gin ON t_docs USING gin (attrs jsonb_path_ops);
EXPLAIN (ANALYZE) SELECT * FROM t_docs WHERE attrs @> '{"k":"42"}'::jsonb;
```
预期：执行计划里出现 `Bitmap Index Scan` 且没有大量 `Rows Removed by Filter`。删掉索引再跑一次，对比差异。
⚠️ 如果这张 5000 行的表被规划成 `Seq Scan`，那说明表确实太小、全表扫描更便宜（见 [04 章 4.4](04-usage.md)：`Seq Scan` 不一定代表"错"）——把行数放大到十万级再对比一次。

**可观察的完成检查**：

- [ ] 实验 A 的两次输出**键顺序不同**，你能指出差异在哪。
- [ ] 实验 B 第二行返回 `f`，且你能用"词元归一化"解释原因。
- [ ] 实验 C 在建索引后执行计划中出现 `Index Scan` 或 `Bitmap Index Scan`；删索引后出现 `Seq Scan`。
- [ ] 你能说出 `jsonb_path_ops` 与 `jsonb_ops` 的区别，并说明为什么实验 C 用前者。（依据 [Research-derived · R-05](07-verification-record.md#r-05)）

---

### 阶段 4 · 学会看执行计划

**读**：[04 · 使用](04-usage.md) 的 4.4

**练**：

1. 在 `orders` 表上跑一条带 `WHERE customer_id = 1` 的查询（把 `1` 换成你库里真实存在的 `customer_id`；`?` 是客户端库的占位符，在 `psql` 里不能直接用），用 `EXPLAIN (ANALYZE, TIMING TRUE)`。
2. 没有索引时看一次输出；建索引后再看一次。
3. **故意踩一次坑**：在一张**可丢弃的练习表**上（例如 `CREATE TABLE explain_demo (id int, v int)`），先在事务外对一条 `UPDATE` 直接跑一次 `EXPLAIN ANALYZE`，观察数据**真的被改了**（这就是坑：官方原文说 `ANALYZE` 会让语句"照常产生副作用"）；再按 `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;` 的形式做一次，验证这次数据没变。（依据 [Research-derived · R-14](07-verification-record.md#r-14)）

**可观察的完成检查**：

- [ ] 你能指出建索引前后执行计划的差异（`Seq Scan` ↔ `Index Scan`/`Bitmap Index Scan`）。
- [ ] 你能指出 `Planning Time` 与 `Execution Time` 哪一项更大，以及这意味着什么。
- [ ] 你能说出"对写语句跑 `EXPLAIN ANALYZE` 必须包在 `BEGIN; ... ROLLBACK;` 里"的原因，并现场演示：事务外那次数据确实变了，包在事务里那次数据没变。
- [ ] 你运行过一次 `ANALYZE 表名;` 并观察到估算行数与实际行数变接近。

---

### 阶段 5 · 学会看膨胀与运维

**读**：[04 · 使用](04-usage.md) 的 4.5 → [02 · 核心概念](02-core-concepts.md) 的 2.3

**练**：

1. 建一张会高频更新的表（实验 C 的表复用即可）。
2. 对它执行上千次 `UPDATE`。
3. 跑 `pg_stat_user_tables` 看 `n_dead_tup`；启用 `pgstattuple` 看精确值。
4. 执行 `VACUUM`，再看一次。
5. 用 `ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = ...)` 调整阈值。

**可观察的完成检查**：

- [ ] 你能展示 `n_dead_tup` 在高频 `UPDATE` 后**明显大于 0**（先看 `last_autovacuum`：若它已经晚于你的 UPDATE 批次，说明 autovacuum 抢先回收过，这一项要重做或先 `ALTER TABLE t_docs SET (autovacuum_enabled = false)`）。注意 `n_dead_tup` 是**估计值**（官方定义见 [R-16](07-verification-record.md#r-16)）。
- [ ] 你能展示 `VACUUM` 之后 `n_dead_tup` 显著下降。
- [ ] 你能说出 `pgstattuple` 只取读锁、结果不是瞬时快照（官方文档明确说明，见 [R-17](07-verification-record.md#r-17)）。
- [ ] 你能说出 autovacuum 的触发公式，并解释为什么"高写入表要调小 scale_factor"（见 [R-03](07-verification-record.md#r-03)）。
- [ ] 你能说出 `VACUUM` 与 `VACUUM FULL` 的取舍（见 [R-36](07-verification-record.md#r-36)）。

---

### 阶段 6 · 做一次真实选型

**读**：[05 · 相关工具](05-related-tools.md)

**练**：拿一个你真实负责（或熟悉的）系统，用 5.8 的选择规则表逐项过一遍，写出结论：

> **示例标签**：Illustrative（选项清单依据 05 章 5.8 选择规则）

```
需求：____________________
数据模型：关系 / 文档 / 混合
写入形态：低频写 / 高频更新 / 追加
检索需求：精确过滤 / 全文 / 向量 / 分析

结论：PostgreSQL / MySQL / MongoDB / PostgreSQL + 某扩展 / 多组件组合
理由：____________________
被排除的选项及原因：____________________
如果规模扩大 10 倍，最先出问题的环节是：____________________
```

**可观察的完成检查**：

- [ ] 你的结论**至少引用一条具体证据**（可以引用本课程章节或 [07 章](07-verification-record.md)的记录编号）。
- [ ] 你能说出被排除选项的具体原因，而不是"不合适"三个字。
- [ ] 你能指出"规模扩大 10 倍时最先出问题的环节"，并说明你会用什么指标观测它（对应阶段 5 的 `n_dead_tup` 等）。
- [ ] 你转述源文档的**基准性能对比、Stripe 队列迁移案例、规模阈值**这三处断言时，**没有**把它们说成已核实的事实（见 [05 章 5.7](05-related-tools.md#57-三条没有证据支撑的对比) 与 [07 章未解决问题](07-verification-record.md#未解决问题)）。

---

## 6.3 节奏建议

| 你的情况 | 建议路径 |
| :--- | :--- |
| 团队没人懂 PostgreSQL 运维 | 阶段 1→2→3→4→5→6 全走，阶段 5 至少花双倍时间 |
| 你只是要用 PostgreSQL 写业务 | 阶段 1→2→3→4 必做，阶段 5 至少做完"练 1–3"，阶段 6 快速过一遍 |
| 你要做技术选型 | 阶段 1→3→6 为主，阶段 2/4/5 至少各做一次动手 |
| 你已经有运维经验 | 阶段 1、2、6 快速过，重点做阶段 4、5 |
| 你在评估要不要换掉 MongoDB | 重点做阶段 3 + 阶段 6，并读 [05 章 5.3](05-related-tools.md#53-postgresql-vs-mongodb) |

**时间的现实估计**：阶段 1 约 1 小时，阶段 2 约 1 小时，阶段 3 约 2 小时，阶段 4 约 1.5 小时，阶段 5 约 2 小时，阶段 6 约 1 小时（不含查资料时间）。按推荐顺序走完约 8–9 小时动手量；若按上表给阶段 5 加倍，总量约 10–11 小时。

---

## 6.4 课程完成的判定标准

完成本课程的标志不是"读完了七章"，而是下面这些**全部为真**：

- [ ] 我能不看笔记画出 `UPDATE → 死元组 → 表膨胀 → VACUUM` 的链条，并说清 autovacuum 在其中的角色。
- [ ] 我能对一条真实查询给出"要不要建索引、用什么索引、怎么验证"的完整答案。
- [ ] 我能独立完成一次选型判断，并说明每个结论背后的证据。
- [ ] 我知道源文档中哪些断言未经外部核实，转述时不会把它们当作事实。
- [ ] 我能对一个跑了半年的 PostgreSQL 实例做一次健康巡检（死元组、autovacuum 状态、执行计划抽样）。

---

## 6.5 本节完成检查

- [ ] 我已把六个阶段拆成了具体的日程，而不是"有空就学"。
- [ ] 我知道自己可以跳过哪些阶段、哪些必须做。
- [ ] 我确认了每个阶段的"可观察的完成检查"是可执行的（能在终端里跑出一条命令并看到结果），不是主观感受。
