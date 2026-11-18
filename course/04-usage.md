# 04 · 使用：正确的方式比"会写 SQL"更重要

**学习目标**

读完之后，你应当能够：

1. 在一台 Linux 机器上装好 PostgreSQL 并连上去。
2. 完整走一遍"建库 → 建表 → 写入 → 查询 → 加索引 → 事务边界"的上手序列。
3. 用 `EXPLAIN ANALYZE` 判断一条查询到底有没有走索引。
4. 用系统视图和 `pgstattuple` 观察死元组与表膨胀，并知道该调哪个 autovacuum 参数。
5. 面对"该建索引还是该改查询"做出有依据的选择，而不是凭感觉。

前置：已读 [02 · 核心概念](02-core-concepts.md) 与 [03 · 核心能力](03-core-functionality.md)。

---

## 4.1 前置条件

| 需要什么 | 说明 | 证据 |
| :--- | :--- | :--- |
| 一台 Linux 机器 | 源文档的安装路径写的是 Linux 系统包管理器 | [S-10](07-verification-record.md#s-10) |
| 系统包管理器 | Debian/Ubuntu 用 `apt`，Red Hat 系用 `dnf` | [R-19](07-verification-record.md#r-19)、[R-20](07-verification-record.md#r-20) |
| `psql` 客户端 | 连接服务器用的交互式客户端。Debian/Ubuntu 装 `postgresql` 即可用；Red Hat 系的服务端包名是 `postgresql-server`，客户端包名以你的发行版为准（本课程未就包内容建立研究记录） | [R-19](07-verification-record.md#r-19)、[R-20](07-verification-record.md#r-20) |
| 一个能停写的窗口 | `VACUUM FULL` 会重写整表并取 `ACCESS EXCLUSIVE` 锁，必须停写；普通 `CREATE INDEX` 只阻塞写入、不阻塞读，`CREATE INDEX CONCURRENTLY` 连写入都不阻塞 | [R-36](07-verification-record.md#r-36)、[R-37](07-verification-record.md#r-37) |

**不需要**：ORM 框架、Grafana、连接池。你在本章里看到的所有 SQL 都能用 `psql` 直接执行。

---

## 4.2 安装与启动

源文档给的入门路径非常简单：

> 在 Linux 上通过系统包管理器即可安装，默认端口通常是 `5432`。
> — [Source-derived · S-10](07-verification-record.md#s-10)

**Debian / Ubuntu**

> **示例标签**：Research-derived（命令直接来自官方下载页 R-19）
>
> ```sh
> apt install postgresql
> ```
>
> 官方页面同时说明："Ubuntu includes PostgreSQL by default."

**Red Hat / Rocky / AlmaLinux**

> **示例标签**：Research-derived（命令直接来自官方下载页 R-20）
>
> ```sh
> dnf install postgresql-server
> postgresql-setup --initdb
> systemctl enable postgresql.service
> systemctl start postgresql.service
> ```
>
> 注意 Red Hat 系要多跑三条命令：`postgresql-setup --initdb` 初始化数据目录，`systemctl enable` + `systemctl start` 管理服务（命令见 [R-20](07-verification-record.md#r-20)）。Debian/Ubuntu 官方页面只给出 `apt install postgresql` 一条命令（[R-19](07-verification-record.md#r-19)）；建集群与开机自启的行为该页未说明，实际以你的环境为准。

**macOS / Homebrew**

> **示例标签**：Illustrative（Homebrew 的 PostgreSQL 服务名随安装版本变化；以下以 `postgresql@16` 为例）
>
> ```sh
> # 安装 PostgreSQL 16
> brew install postgresql@16
>
> # 后台启动，并设为登录后自动启动
> brew services start postgresql@16
>
> # 连接默认数据库；默认角色通常是当前 macOS 用户
> psql postgres
>
> # 停止后台服务
> brew services stop postgresql@16
> ```
>
> 如果安装的是未带版本号的 `postgresql`，把上述命令中的 `postgresql@16` 换成 `postgresql`。可用 `brew services list` 确认实际服务名和运行状态。

### 端口

> The TCP port the server listens on; 5432 by default. Note that the same port number is used for all IP addresses the server listens on. This parameter can only be set at server start.
> — [Research-derived · R-15](07-verification-record.md#r-15)

⚠️ 两条实用推论：

1. `port` **只能在服务启动时设置**，改 `postgresql.conf` 后需要重启才生效。
2. 所有 IP 地址共用同一个端口，所以本机连和远程连是同一个端口号——排查连接问题时不要被这个搞混。

### 连接数上限

> The default is typically 100 connections, but might be less if your kernel settings will not support it (as determined during initdb).
> — [Research-derived · R-15](07-verification-record.md#r-15)

100 是个小数字。如果你的应用用连接池，注意池大小 + 其他客户端不要把它撑爆。

---

## 4.3 首次上手序列

> 入门路径很直接：建库、建表、写入、查询、加索引、理解事务边界。真正需要花时间的是**执行计划（`EXPLAIN ANALYZE`）和索引策略**，而不是 SQL 语法本身。
> — [Source-derived · S-10](07-verification-record.md#s-10)

下面把这条路径完整走一遍。

### 第 1 步：连接并确认身份

> **示例标签**：Illustrative（连接命令为常见写法；本课程未就认证配置建立研究记录，实际以你的 `pg_hba.conf` 为准）

```sh
# Debian/Ubuntu 默认安装下，最省事的是用系统用户直接进（走本地 socket）
sudo -u postgres psql

# 或者显式指定主机与端口（能否连上取决于 pg_hba.conf 对该角色的认证方式）
psql -U postgres -h 127.0.0.1 -p 5432
```

进入 `psql` 后：

> **示例标签**：Illustrative（内置函数；输出取决于你的实例）

```sql
SELECT version();
SELECT current_database(), current_user;
```

### 第 2 步：建库

> **示例标签**：Illustrative（命令序列；库名为示意）

```sql
CREATE DATABASE shop;
\connect shop
```

### 第 3 步：建表

> **示例标签**：Illustrative（DDL 为示意设计；类型选择依据 S-04 的"固定列优先"原则与 R-04 的 jsonb 事实）
>
> ```sql
> CREATE TABLE customers (
>   id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>   email text NOT NULL UNIQUE,
>   name  text NOT NULL,
>   created_at timestamptz NOT NULL DEFAULT now()
> );
>
> CREATE TABLE products (
>   id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>   name  text NOT NULL,
>   price numeric(10,2) NOT NULL,
>   attrs jsonb NOT NULL DEFAULT '{}'::jsonb
> );
>
> CREATE TABLE orders (
>   id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>   customer_id bigint NOT NULL REFERENCES customers(id),
>   product_id  bigint NOT NULL REFERENCES products(id),
>   amount      numeric(12,2) NOT NULL,
>   status      text NOT NULL DEFAULT 'new',
>   created_at  timestamptz NOT NULL DEFAULT now()
> );
> ```

**注意建模选择**：`status` 是固定列而不是 JSONB 字段，`amount` 用 `numeric` 而不是 `float`——因为它要参与金额计算。`attrs` 才是留给"每个品类不同"的动态属性。

### 第 4 步：写入

> **示例标签**：Illustrative（业务数据为构造值）

```sql
INSERT INTO customers (email, name) VALUES ('a@example.com', '张三');
INSERT INTO products (name, price, attrs) VALUES ('红色 T 恤', 99.00, '{"color":"red"}');
INSERT INTO orders (customer_id, product_id, amount) VALUES (1, 1, 99.00);
```

### 第 5 步：查询

> **示例标签**：Illustrative（窗口函数语法依据 R-09；数据为构造值）

```sql
-- 关联 + 窗口函数
SELECT c.id, c.name, o.id AS order_id, o.amount,
       RANK() OVER (PARTITION BY c.id ORDER BY o.amount DESC) AS rnk
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

### 第 6 步：加索引

> **示例标签**：Illustrative（B-Tree 建索引语法为通用语法；GIN 与算子类依据 R-05）

```sql
-- 高频过滤列
CREATE INDEX orders_customer_id_idx ON orders (customer_id);
CREATE INDEX orders_status_idx ON orders (status);

-- JSONB 内部键查询
CREATE INDEX products_attrs_gin ON products USING gin (attrs jsonb_path_ops);
```

### 第 7 步：理解事务边界

> **示例标签**：Illustrative（事务边界示意；数据为构造值）

```sql
BEGIN;
UPDATE orders SET status = 'paid' WHERE id = 1;
-- 这里可以检查 SELECT 结果
COMMIT;   -- 或 ROLLBACK;
```

⚠️ `BEGIN` 到 `COMMIT`/`ROLLBACK` 之间就是事务边界：**这段区间内的写操作要么全部生效，要么全部不生效**。断连时未提交的事务不会保留下来——写操作前先想清楚这段是否要留在线上（本课程未就断连时的后端行为建立研究记录，此处只作原则性提醒）。

---

## 4.4 读执行计划：`EXPLAIN ANALYZE`

这是源文档点名"真正需要花时间"的两件事之一。

> **示例标签**：Research-derived（`ANALYZE` / `TIMING` 选项语义依据 R-14；执行计划输出形状为示意）

```sql
EXPLAIN (ANALYZE, TIMING TRUE, BUFFERS)
SELECT o.id, o.amount
FROM orders o
WHERE o.customer_id = 1;
```

官方对 `ANALYZE` 的说明：

> The `ANALYZE` option causes the statement to be actually executed, not only planned. Then actual run time statistics are added to the display, including the total elapsed time expended within each plan node (in milliseconds) and the total number of rows it actually returned. This is useful for seeing whether the planner's estimates are close to reality.
> — [Research-derived · R-14](07-verification-record.md#r-14)

⚠️ **最重要的一条警告**：

> Keep in mind that the statement is actually executed when the `ANALYZE` option is used. Although `EXPLAIN` will discard any output that a `SELECT` would return, other side effects of the statement will happen as usual. If you wish to use `EXPLAIN ANALYZE` on an `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `CREATE TABLE AS`, or `EXECUTE` statement without letting the command affect your data, use this approach:
>
> ```sql
> BEGIN;
> EXPLAIN ANALYZE ...;
> ROLLBACK;
> ```
> — [Research-derived · R-14](07-verification-record.md#r-14)

**所以：对写语句做 `EXPLAIN ANALYZE`，一律要包在 `BEGIN; ... ROLLBACK;` 里**（唯一的例外是在可丢弃的练习表上故意演示副作用）。这是本课程最有用的一条实操规则。

关于 `TIMING` 选项：

> The overhead of repeatedly reading the system clock can slow down the query significantly on some systems, so it may be useful to set this parameter to `FALSE` when only actual row counts, and not exact times, are needed. Run time of the entire statement is always measured, even when node-level timing is turned off with this option.
> — [Research-derived · R-14](07-verification-record.md#r-14)

> **示例标签**：Illustrative（执行计划输出格式为示意；判读规则依据 R-14）
>
> 判读时按这个顺序看：
>
> 1. **`Seq Scan` 还是 `Index Scan`** —— `Seq Scan` 不代表"错"，小表全扫常常更快；但**大表上出现 `Seq Scan` + `Rows Removed by Filter: 100000`** 基本就是缺索引的信号。
> 2. **`actual time=... rows=...`** —— 实际值 vs 估算值差距很大，说明统计信息过期，跑一次 `ANALYZE`。
> 3. **`Planning Time` vs `Execution Time`** —— 计划时间占比高，通常说明这条语句参与连接/展开的关系太多（或分区太多），先减少参与连接的关系，而不是笼统地"拆分 SQL"。
> 4. **是否"回表"要看 `Heap Fetch`** —— 计划里出现 `Heap Fetch` 节点、且索引扫描取回的行数远大于堆块访问数时，才是回表过多；`Buffers: shared hit=... read=...` 表示数据块在共享缓冲区命中还是从磁盘读，不等于回表次数（且需显式 `EXPLAIN (ANALYZE, BUFFERS)` 才会输出）。

---

## 4.5 观察 VACUUM 与表膨胀

这一节直接对应源文档的第二条重点：

> 必须建立 VACUUM 意识 …… 任何高频写入的表都需要关注表膨胀和 autovacuum 的状态。
> — [Source-derived · S-09](07-verification-record.md#s-09)

### 日常巡检查询

> **示例标签**：Research-derived（字段与语义依据 R-16；查询语句为组装）
>
> ```sql
> SELECT relname,
>        n_live_tup, n_dead_tup,
>        last_vacuum, last_autovacuum,
>        last_analyze, last_autoanalyze
> FROM pg_stat_user_tables
> ORDER BY n_dead_tup DESC
> LIMIT 20;
> ```
>
> 字段含义（官方定义）：
>
> | 字段 | 官方定义 |
> | :--- | :--- |
> | `n_live_tup` | Estimated number of live rows |
> | `n_dead_tup` | Estimated number of dead rows |
> | `last_vacuum` | 该表最近一次被**手动** vacuum 的时间（不含 `VACUUM FULL`） |
> | `last_autovacuum` | 该表最近一次被 **autovacuum 守护进程** vacuum 的时间 |
> | `last_analyze` | 该表最近一次被**手动** analyze 的时间 |
> | `last_autoanalyze` | 该表最近一次被 **autovacuum 守护进程** analyze 的时间 |

### 需要精确物理数据时

> **示例标签**：Research-derived（扩展与函数依据 R-17；返回列口径见下）

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders');
```

`pgstattuple` 返回关系物理长度、**存活元组总长度与占比**、**死元组总长度与占比**、空闲空间与可用空间等——注意它给的是**长度与百分比，不是存活/死元组的计数**（计数要看 `pg_stat_user_tables` 的 `n_live_tup` / `n_dead_tup`）。官方提醒：

> `pgstattuple` acquires only a read lock on the relation. So the results do not reflect an instantaneous snapshot; concurrent updates will affect them.
> — [Research-derived · R-17](07-verification-record.md#r-17)

⚠️ 顺带纠正一个流传很广的说法：`pgstattuple` **不需要** `ACCESS EXCLUSIVE` 锁，它只取读锁。但代价是结果不是瞬时快照，且它会全表扫描——生产大表上用 `pgstattuple_approx` 更稳（官方原文：「pgstattuple_approx is a faster alternative to pgstattuple that returns approximate results.」）。注意它的列名差异：存活元组一列叫 `approx_tuple_count`，而死元组一列仍叫 `dead_tuple_count`（[R-17](07-verification-record.md#r-17)）。

### 调整 autovacuum

autovacuum 是**按阈值触发**的：

> vacuum threshold = Minimum(vacuum max threshold, vacuum base threshold + vacuum scale factor * number of tuples)
> — [Research-derived · R-03](07-verification-record.md#r-03)

对一张高频更新的表，你通常要**调小比例阈值**（让 autovacuum 更早介入），而不是加大延迟：

> **示例标签**：Research-derived（参数名依据 [R-03](07-verification-record.md#r-03)、[R-18](07-verification-record.md#r-18)；`ALTER TABLE ... SET (...)` 为 PostgreSQL 通用语法；四个阈值为 Illustrative 构造值）
>
> ```sql
> ALTER TABLE orders SET (
>   autovacuum_vacuum_threshold  = 1000,
>   autovacuum_vacuum_scale_factor = 0.05,
>   autovacuum_analyze_threshold   = 500,
>   autovacuum_analyze_scale_factor = 0.02
> );
> ```
>
> 这些都是**每表存储参数（storage parameters）**，可以逐表覆盖全局设置：
>
> The default thresholds and scale factors are taken from postgresql.conf, but it is possible to override them (and many other autovacuum control parameters) on a per-table basis.
> — [Research-derived · R-03](07-verification-record.md#r-03)
>
> ⚠️ 数值是示意，不是推荐值。正确取值取决于你的行更新频率和可接受的膨胀比例。

### `VACUUM` 还是 `VACUUM FULL`？

官方给的判断很清楚：

> The autovacuum daemon attempts to work this way, and in fact will never issue `VACUUM FULL`.
>
> Thus, moderately-frequent standard `VACUUM` runs are a better approach than infrequent `VACUUM FULL` runs for maintaining heavily-updated tables.
> — [Research-derived · R-03](07-verification-record.md#r-03)

**默认选 `VACUUM`。** 官方原文说得很直白：

> Plain `VACUUM` (without `FULL`) simply reclaims space and makes it available for re-use. This form of the command can operate in parallel with normal reading and writing of the table, as an exclusive lock is not obtained.
>
> `VACUUM FULL` rewrites the entire contents of the table into a new disk file with no extra space, allowing unused space to be returned to the operating system. This form is much slower and requires an `ACCESS EXCLUSIVE` lock on each table while it is being processed.
> — [Research-derived · R-36](07-verification-record.md#r-36)

也就是说：`VACUUM` 回收空间供后续复用但不把文件还给操作系统，也不阻塞正常的读写；`VACUUM FULL` 会重写整表并**持有强锁**（`ACCESS EXCLUSIVE`），应该只在确实需要把磁盘还给操作系统、且能停写时用。

---

## 4.6 建模实践：固定列优先

源文档把这条列为"被反复强调的最佳实践"：

> 一个被反复强调的最佳实践是：**能作为固定字段的，就不要塞进 JSONB**。把高频过滤、排序、约束的字段保留为独立列，只把真正多变、稀疏的属性放入 JSONB。否则你会得到一张"看起来灵活、查起来全表扫描"的表，JSONB 的索引优势也会被滥用稀释。
> — [Source-derived · S-08](07-verification-record.md#s-08)

> **示例标签**：Illustrative（对照设计为示意；CHECK 约束的语义依据 [R-38](07-verification-record.md#r-38)）
>
> ❌ 反模式：什么都塞 JSONB
>
> ```sql
> CREATE TABLE orders_bad (
>   id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>   data jsonb NOT NULL
> );
> -- 查询要写成 data->>'status' = 'paid'
> -- 排序要写成 (data->>'amount')::numeric
> -- 约束要绕一层类型转换：CHECK ((data->>'amount')::numeric > 0)
> -- 而且 PostgreSQL 假定 CHECK 约束的条件是不可变的，只在插入/更新行时检查
> ```
>
> ✅ 正确姿势：固定列 + 动态属性
>
> ```sql
> CREATE TABLE orders_good (
>   id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>   status text NOT NULL DEFAULT 'new',
>   amount numeric(12,2) NOT NULL CHECK (amount > 0),
>   attrs  jsonb NOT NULL DEFAULT '{}'::jsonb
> );
> CREATE INDEX orders_good_status_idx ON orders_good (status);
> CREATE INDEX orders_good_attrs_gin ON orders_good USING gin (attrs jsonb_path_ops);
> ```
>
> 判断标准很简单：**这个字段会被用来过滤、排序、或需要数据库层约束吗？** 会 → 给它独立列。不会，且形状多变 → 放 JSONB。

---

## 4.7 常见错误清单

| 症状 | 最可能的原因 | 怎么查 | 证据 |
| :--- | :--- | :--- | :--- |
| 查询变慢，且 `n_dead_tup` 很高 | 死元组累积，autovacuum 追不上 | `pg_stat_user_tables` + `pgstattuple` | [S-09](07-verification-record.md#s-09)、[R-16](07-verification-record.md#r-16) |
| `EXPLAIN` 里估算行数与实际差很多 | 统计信息过期 | `ANALYZE 表名;` 后重看 | [R-14](07-verification-record.md#r-14) |
| 大表上出现 `Seq Scan` + 大量 `Rows Removed by Filter` | 缺索引或索引没被用上 | `EXPLAIN (ANALYZE)` | [R-14](07-verification-record.md#r-14) |
| JSONB 查询走了全表扫描 | 用了 `jsonb_path_ops` 索引但查询操作符不被支持 | 核对操作符是否在 `@>`/`@?`/`@@` 内 | [R-05](07-verification-record.md#r-05) |
| 全文检索命中不对 | 直接 `::tsvector` 强转，未做词元归一 | 改用 `to_tsvector('english', ...)` | [R-08](07-verification-record.md#r-08) |
| `EXPLAIN ANALYZE` 之后数据变了 | 对写语句直接跑 `EXPLAIN ANALYZE` | 包在 `BEGIN; ... ROLLBACK;` 里 | [R-14](07-verification-record.md#r-14) |
| 连接数耗尽 | `max_connections` 默认约 100 | 检查连接池配置 | [R-15](07-verification-record.md#r-15) |

---

## 4.8 本节完成检查

- [ ] 我在一台 Linux 机器上完成了安装，并能连上 `psql`。
- [ ] 我走完了 4.3 的七个步骤（建库 → 建表 → 写入 → 查询 → 加索引 → 事务）。
- [ ] 我用 `EXPLAIN (ANALYZE, TIMING TRUE)` 看过一条查询，并能说出它是否走了索引。
- [ ] 我知道为什么对 `UPDATE` 跑 `EXPLAIN ANALYZE` 必须包 `ROLLBACK`。
- [ ] 我跑过 `pg_stat_user_tables` 的死元组巡检，并知道 `n_dead_tup` 大意味着什么。
- [ ] 我能说出 `VACUUM` 与 `VACUUM FULL` 的取舍，以及为什么默认选前者。
- [ ] 我能对一张自己的表指出：哪些字段该是固定列，哪些该进 JSONB。
