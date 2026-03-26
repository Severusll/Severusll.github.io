---
title: "MySQL 学习笔记"
date: 2026-03-01 11:40:00
tags:
  - mysql
  - 数据库
  - 学习笔记
categories:
  - 数据库
thumbnail: "/images/thumbnails/mysql.png"
---

---

## 1. 我是怎么开始系统学 MySQL 的

一开始我对 MySQL 的理解很“工具化”：

- 会 `select`、`insert`、`update`、`delete`；
- 能把功能做出来；
- 但遇到慢查询、数据不一致、锁等待时就很被动。

后来我意识到，数据库不是“写几条 SQL”这么简单。它其实有三层：

1. **语法层**：SQL 能写出来；
2. **设计层**：表结构、索引、约束是否合理；
3. **运行层**：事务、锁、执行计划、磁盘与内存的取舍。

我的学习方式也跟着变了：每学一个点，都问自己两个问题：

- 如果线上出问题，我能不能定位？
- 如果让我重新设计一次，我会不会做出更好的选择？

---

## 2. SQL 基础：先把“正确”写出来

### 2.1 DDL / DML / DQL 的基本分工

- `DDL`：定义结构（库、表、索引）
- `DML`：修改数据（增删改）
- `DQL`：查询数据

```sql
-- 建库
CREATE DATABASE IF NOT EXISTS study_mysql
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_general_ci;

USE study_mysql;

-- 建表
CREATE TABLE IF NOT EXISTS user_account (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(64) NOT NULL,
  email VARCHAR(128) NOT NULL,
  status TINYINT NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_email (email)
);
```

### 2.2 我踩过的基础坑

| 坑点 | 当时的做法 | 后来修正 |
| :-- | :-- | :-- |
| 字符集混用 | 表有的 `utf8`，有的 `utf8mb4` | 统一 `utf8mb4` |
| 时间字段随便存 | 用字符串存时间 | 改为 `DATETIME` / `TIMESTAMP` |
| 约束缺失 | 业务校验，不加唯一约束 | 关键字段加 `UNIQUE` |

**我的反思：**“先跑起来”没问题，但数据库字段一旦上线，迁移成本会越来越高。结构设计要在早期就认真一点。

---

## 3. 查询进阶：结果对不对，速度快不快

### 3.1 常用查询套路

```sql
-- 条件筛选 + 排序 + 分页
SELECT id, username, created_at
FROM user_account
WHERE status = 1
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- 聚合统计
SELECT status, COUNT(*) AS total
FROM user_account
GROUP BY status
HAVING total > 10;
```

### 3.2 JOIN 的理解

我以前把 `JOIN` 当语法题，后来才发现它是“关系设计能力”的体现。

```sql
SELECT o.id AS order_id, u.username, o.total_amount
FROM orders o
JOIN user_account u ON o.user_id = u.id
WHERE o.pay_status = 'PAID';
```

- `INNER JOIN`：只保留两边都匹配的数据；
- `LEFT JOIN`：以左表为准，右表没有则 `NULL`。

**个人提醒：**写 `JOIN` 时，我会先确认“谁是主视角表”，避免结果集变形后才发现重复或漏数。

### 3.3 子查询与窗口函数（MySQL 8）

```sql
SELECT username, score,
       DENSE_RANK() OVER (ORDER BY score DESC) AS rk
FROM exam_result;
```

窗口函数对“排名、分组内比较、累计值”很实用，语义比老式子查询清晰很多。

---

## 4. 表设计：比 SQL 技巧更重要

### 4.1 我现在遵循的建表习惯

- 每张核心业务表都有主键（一般 `BIGINT` 自增或雪花 ID）；
- 统一审计字段：`created_at`、`updated_at`；
- 可枚举状态字段用整型或短字符串，并在文档里列出含义；
- 尽量避免“一个字段存多个值”的反范式设计。

### 4.2 范式与反范式

我对范式的理解：

- 范式提升数据一致性；
- 反范式提升查询性能；
- 真正方案不是“站队”，而是根据读写比例和业务稳定性做平衡。

**个人思考：**我过去喜欢一步到位做“完美抽象”，后来发现业务还在变，表拆得太细反而拖慢迭代。现在会先保证正确，再逐步演进。

---

## 5. 索引：不是加了就快，而是加对了才快

### 5.1 常见索引类型（InnoDB 语境）

- 主键索引（聚簇索引）
- 二级索引（普通索引、唯一索引）
- 联合索引

```sql
CREATE INDEX idx_status_created_at
ON user_account (status, created_at);
```

### 5.2 联合索引的最左前缀

如果索引是 `(a, b, c)`，通常可用：

- `a`
- `a, b`
- `a, b, c`

不一定能高效用到：

- `b`
- `b, c`

### 5.3 用 EXPLAIN 看执行计划

```sql
EXPLAIN SELECT id, username
FROM user_account
WHERE status = 1
ORDER BY created_at DESC
LIMIT 20;
```

我主要关注这几列：

- `type`：访问方式（是否全表扫描）
- `key`：是否命中期望索引
- `rows`：预估扫描行数
- `Extra`：是否出现 `Using filesort` / `Using temporary`

**我的经验：**慢查询优化通常不是“神奇 SQL 技巧”，而是“索引设计 + 查询改写 + 数据分布”一起考虑。

---

## 6. 事务与隔离级别：理解一致性成本

### 6.1 ACID 的直观理解

- `A` 原子性：要么都成功，要么都失败；
- `C` 一致性：事务前后，数据约束不被破坏；
- `I` 隔离性：并发事务互不干扰（程度不同）；
- `D` 持久性：提交后结果可恢复。

### 6.2 隔离级别与常见现象

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :-- | :--: | :--: | :--: |
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed | 不会 | 可能 | 可能 |
| Repeatable Read (MySQL 默认) | 不会 | 不会 | 可能（InnoDB 通过 MVCC/Next-Key Lock 降低） |
| Serializable | 不会 | 不会 | 不会 |

### 6.3 一个我印象很深的误区

我以前以为“开了事务就万无一失”，后来才知道：

- 事务解决的是一致性，不直接解决所有并发竞争；
- 锁范围、索引命中、SQL 写法都会影响并发性能；
- 长事务非常危险，会占用资源并放大锁冲突。

---

## 7. 锁与并发：写得出，不等于扛得住

### 7.1 先理解现象，再谈优化

并发问题在我看来通常有三种表现：

1. 等锁（请求变慢）；
2. 死锁（事务互相等待）；
3. 结果异常（覆盖更新、读到旧值）。

### 7.2 实用建议（给未来的自己）

- 更新条件尽量命中索引，缩小锁范围；
- 事务内只放必要 SQL，不做耗时操作；
- 访问多表时固定顺序，减少死锁概率；
- 对热点更新考虑乐观锁（版本号）或队列化处理。

---

## 8. 我常用的排查路径

当我怀疑数据库问题时，通常按这个顺序看：

1. 先看慢查询日志（是不是 SQL 本身慢）；
2. 用 `EXPLAIN` 看执行计划（索引是否命中）；
3. 看锁等待和死锁日志（是不是并发冲突）；
4. 对照业务高峰时段（是不是流量和批任务叠加）；
5. 最后再考虑参数调优（避免一上来就“拍脑袋调配置”）。

这个顺序帮我避免了很多无效操作。

---

## 9. 小结：我现在对 MySQL 的理解

目前我把 MySQL 学习分成三句话：

- **先写对**：数据正确、约束清晰；
- **再写快**：索引合理、查询可解释；
- **最后写稳**：事务边界清楚、并发问题可控。

数据库学习很像“慢工”：短期看不明显，长期收益很大。每次遇到线上或练习里的异常，我都尽量写一条复盘，积累自己的判断力。

---

## 10. 下一步学习清单

- [ ] 继续深入 InnoDB 存储结构（页、B+ 树、回表、覆盖索引）
- [ ] 系统学习 MVCC 与 undo log / redo log
- [ ] 实操主从复制与读写分离基础
- [ ] 练习常见 SQL 优化题，形成自己的优化 checklist

如果后面我有新的踩坑记录，会持续追加到这篇笔记里。
