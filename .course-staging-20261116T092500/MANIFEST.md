# 暂存清单（Staging Manifest）—— 审校后最终版

- **生成时间**：2026-11-16（审校完成后的最终内容）
- **暂存根目录（canonical）**：`/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500`
- **晋级目标目录（canonical）**：`/Users/doing/Desktop/postgresql-play/course`
- **清单状态**：**已审校（post-audit）**，对应 [07 章的晋级审批记录](07-verification-record.md#晋级审批记录)

## 只读根源文件（禁止修改）

| 路径 | SHA-256 | 字节 | 操作 |
| :--- | :--- | :--- | :--- |
| `/Users/doing/Desktop/postgresql-play/source.md` | `c2fad4f2569fa6316d05fcbbe58d14585dd00750309c4455d42395854d763854` | 8196 | read-only / 禁止修改 |

## 暂存章节文件（晋级的唯一目标边界）

| 规范路径 | SHA-256 | 字节 | 晋级操作 | 晋级后路径 |
| :--- | :--- | :--- | :--- | :--- |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/01-positioning-and-background.md` | 6ce55a7e507a155bef11d0c0a2bb09e30bf157760e3f793f4a35a865334e8109 | 9493 | create | `/Users/doing/Desktop/postgresql-play/course/01-positioning-and-background.md` |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/02-core-concepts.md` | aff7e5b18bb7ac0c8357cab31fcf1cc8bd9495d20332930411615fb65d089e14 | 26124 | create | `/Users/doing/Desktop/postgresql-play/course/02-core-concepts.md` |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/03-core-functionality.md` | 327e2846b9c7eb4e6352fc64acf4d1bbae8d8d75ea17e4bc33db4d4c5b3e4bd7 | 17542 | create | `/Users/doing/Desktop/postgresql-play/course/03-core-functionality.md` |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/04-usage.md` | aa59df6e67a7588f1bfd2add5204954b98e7524f6b81c6cefac79c3e26da0289 | 19363 | create | `/Users/doing/Desktop/postgresql-play/course/04-usage.md` |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/05-related-tools.md` | 4b23b8629b4a3c500ed240ca5084165eef58f19cd6b11f14152d13d43100b543 | 23695 | create | `/Users/doing/Desktop/postgresql-play/course/05-related-tools.md` |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/06-learning-path.md` | dce35779a2479888b514659da39595519235f00995366d10f812799b36d7041a | 12350 | create | `/Users/doing/Desktop/postgresql-play/course/06-learning-path.md` |
| `/Users/doing/Desktop/postgresql-play/.course-staging-20261116T092500/07-verification-record.md` | 6e36f109fdebf595b8f517bae6c58f674cbe4c45bfc27fbc2ef9111d59d66aa9 | 68980 | create | `/Users/doing/Desktop/postgresql-play/course/07-verification-record.md` |

## 暂存清单校验和（COURSE_DIGEST）

`f8aedaa7505f36e87ce4b2e444968c8c62adcb611ef27dba0ee7a01fc40bad58`

**定义**：SHA-256，依次输入 01–06 六个文件的完整字节，再输入 `07-verification-record.md` 中 `## 晋级审批记录` 之前的全部字节。
该定义刻意排除 07 章的"晋级审批记录"与"最终结果"两节，因此写入审批记录本身不会使审批失效。

## 禁止触碰的路径

- `/Users/doing/Desktop/postgresql-play/source.md`（根源文件）
- `/Users/doing/Desktop/postgresql-play/course/**`（晋级目标目录，在审批通过并完成复核前不得写入）
- `/Users/doing/Desktop/postgresql-play/.git/**`
- 暂存根目录以外的任何路径

## 晋级规则（逐条复核）

1. 目标目录 `course/` 原本**不存在**，七个文件全部为 **create** 操作，**不涉及任何覆盖**。
2. 仅晋级上表列出的七个文件；本清单文件**不晋级**。
3. 晋级前逐条复核：根源校验和不变、目标目录规范路径不变、路径树中无符号链接、七个目标均为普通文件、COURSE_DIGEST 与审批记录一致、文件白名单与操作与审批记录一致。
4. 任何一项不匹配即停止晋级。
