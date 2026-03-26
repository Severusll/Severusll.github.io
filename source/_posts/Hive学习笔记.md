---
title: "Hive 学习笔记"
date: 2026-03-10 14:10:00
tags:
  - hive
  - 大数据
  - 学习笔记
categories:
  - 大数据
thumbnail: "/images/thumbnails/hive.png"
---

---

## 1. 我目前对 Hive 的理解

刚接触 Hive 时，我有一个误区：以为它只是“SQL 换了个地方跑”。

后来在实际使用里才慢慢理解：

- Hive 是建立在 HDFS 之上的数据仓库工具；
- HQL 看起来像 SQL，但底层是批处理计算；
- 它更适合离线分析，不是低延迟 OLTP 数据库。

我现在会把 Hive 放在这样的定位里：

1. 负责数仓分层和离线聚合；
2. 和 Spark/Flink、MySQL 各做自己擅长的事；
3. 重点是可维护、可复跑、可追溯。

---

## 2. 入门基础：数据库、表、分区

### 2.1 常用 DDL

```sql
CREATE DATABASE IF NOT EXISTS dwd_demo;
USE dwd_demo;

CREATE TABLE IF NOT EXISTS user_action_log (
  user_id        STRING,
  action_type    STRING,
  action_time    STRING,
  device_type    STRING,
  page_id        STRING
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;
```

### 2.2 内部表和外部表

| 类型 | 数据删除行为 | 适用场景 |
| :-- | :-- | :-- |
| 内部表（MANAGED） | 删表通常会删数据 | 临时中间层 |
| 外部表（EXTERNAL） | 删表一般不删底层数据 | 与共享目录/历史数据集成 |

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS ods_user_action (
  line STRING
)
PARTITIONED BY (dt STRING)
STORED AS TEXTFILE
LOCATION '/data/ods/user_action';
```

**我的经验：**接外部目录的数据，优先外部表，避免误操作时数据被连带删除。

### 2.3 分区表的意义

分区是我前期收益最高的一个点：

- 减少扫描范围；
- 让按天/按月数据管理更清晰；
- 配合调度任务更容易做增量。

---

## 3. 数据导入与分区管理

### 3.1 装载数据

```sql
LOAD DATA INPATH '/tmp/user_action_20260325.tsv'
INTO TABLE user_action_log
PARTITION (dt='2026-03-25');
```

### 3.2 手动/自动分区

```sql
-- 手动加分区
ALTER TABLE user_action_log ADD PARTITION (dt='2026-03-26');

-- 查看分区
SHOW PARTITIONS user_action_log;
```

动态分区我一开始经常漏配置，导致任务写不进去。

```sql
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT OVERWRITE TABLE user_action_log PARTITION (dt)
SELECT user_id, action_type, action_time, device_type, page_id, dt
FROM tmp_user_action_stage;
```

**踩坑记录：**没有开 `nonstrict` 时，分区字段不固定会报错，排查了半天才发现是 session 参数问题。

---

## 4. 常用查询与窗口函数

### 4.1 基础聚合

```sql
SELECT
  dt,
  action_type,
  COUNT(*) AS cnt
FROM user_action_log
WHERE dt BETWEEN '2026-03-20' AND '2026-03-26'
GROUP BY dt, action_type
ORDER BY dt, cnt DESC;
```

### 4.2 去重与留存分析常见写法

```sql
SELECT dt, COUNT(DISTINCT user_id) AS uv
FROM user_action_log
GROUP BY dt;
```

### 4.3 窗口函数

```sql
SELECT
  dt,
  user_id,
  action_time,
  ROW_NUMBER() OVER (PARTITION BY dt, user_id ORDER BY action_time DESC) AS rn
FROM user_action_log
WHERE dt='2026-03-26';
```

这个能力在“每个用户每天最后一次行为”这类需求中非常实用。

---

## 5. 建模实践：从 ODS 到 DWD 的一次整理

我在练习里采用了比较常见的分层：

- ODS：原始数据落地，尽量少改；
- DWD：清洗明细层，标准化字段；
- DWS：主题聚合层；
- ADS：应用报表层。

### 5.1 一个简化的清洗示例

```sql
INSERT OVERWRITE TABLE dwd_user_action PARTITION (dt='2026-03-26')
SELECT
  user_id,
  lower(action_type) AS action_type,
  from_unixtime(unix_timestamp(action_time, 'yyyy-MM-dd HH:mm:ss')) AS action_time,
  nvl(device_type, 'unknown') AS device_type,
  page_id
FROM ods_user_action_parsed
WHERE dt='2026-03-26'
  AND user_id IS NOT NULL;
```

### 5.2 我的个人思考

以前我常常在 ODS 里就做太多业务规则，后面回溯源数据很痛苦。现在会坚持：

- ODS 保留原始；
- DWD 做统一清洗；
- 业务口径尽量在上层体现。

这样虽然前期多写一点 SQL，但长期维护轻松很多。

---

## 6. 性能优化：我常用的几条检查项

### 6.1 先看执行计划

```sql
EXPLAIN
SELECT user_id, COUNT(*) AS cnt
FROM user_action_log
WHERE dt='2026-03-26'
GROUP BY user_id;
```

我会先确认：

- 是否命中分区过滤；
- 是否有不必要的大范围 `shuffle`；
- 是否可以先过滤再聚合。

### 6.2 小文件问题

学习阶段最容易忽略的就是“小文件过多”。表面上 SQL 能跑，长期会拖垮任务性能。

常见处理思路：

- 在写入阶段控制并行度，避免产生过多碎片文件；
- 定期做小文件合并；
- 根据数据规模调整文件格式（如 ORC/Parquet）。

### 6.3 文件格式选择

| 格式 | 特点 | 适用感觉 |
| :-- | :-- | :-- |
| TextFile | 直观、便于排查 | 入门与临时层 |
| ORC | 压缩率高、查询效率好 | 数仓主力 |
| Parquet | 列式存储，生态兼容好 | 跨引擎协作 |

**我的结论：**大多数分析场景，ORC/Parquet 比 TextFile 更稳妥。

---

## 7. 错误排查记录（学习笔记最有价值的部分）

下面这几类问题，是我在练习或环境运维里反复遇到的。

### 7.1 现象：查不到当天分区数据

**排查步骤：**

1. `SHOW PARTITIONS 表名;`
2. 看 HDFS 路径是否真的有 `dt=xxxx-xx-xx`；
3. 检查是否忘记 `MSCK REPAIR TABLE`（外部表常见）；
4. 检查分区字段格式是否一致（`2026-03-26` vs `20260326`）。

**典型根因：**

- 文件到了 HDFS，但元数据没刷新；
- 分区值格式不统一，查询条件写错。

### 7.2 现象：动态分区插入失败

**常见报错方向：**

- 动态分区参数没开启；
- 分区列位置不对；
- 非严格模式未设置。

**我的修复模板：**

```sql
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
```

并确认 `INSERT` 语句中分区列在 `SELECT` 末尾输出。

### 7.3 现象：HiveServer2 连不上

我会按这个顺序排查：

1. 进程是否在：`ps -ef | grep HiveServer2`
2. 端口是否监听（常见 10000）；
3. metastore 是否正常；
4. 客户端连接串是否写对。

> 这类问题的关键是“先确认服务状态，再确认 SQL”，不要一上来怀疑查询语句。

### 7.4 现象：任务跑得非常慢

**常用定位路径：**

- 看 SQL 是否漏分区条件；
- 看是否做了高基数字段 `group by`；
- 看是否有不必要的大表 join；
- 看输入文件是否过碎；
- 看执行引擎资源是否紧张。

**我的反思：**性能问题大多不是单点故障，通常是“数据规模 + SQL 写法 + 资源配置”叠加。

---

## 8. 我给自己的 Hive 写作/开发规范

- 关键表都写注释，字段含义要清楚；
- SQL 文件按主题拆分，避免一个脚本塞全部逻辑；
- 保留关键口径说明，避免“这列到底怎么算的”无人知道；
- 任务失败要记录：错误现象、根因、修复方式；
- 每周回顾一次慢任务，把可优化 SQL 拉清单。

这套规范看起来很基础，但坚持下来，定位问题速度提升非常明显。

---

## 9. 阶段小结

目前我觉得 Hive 最值得投入的不是“背更多语法”，而是三件事：

1. 分层建模思维；
2. 执行计划和性能意识；
3. 问题排查流程化。

如果只看语法，会觉得 Hive 不难；但一旦把数据量、调度链路、稳定性都算进去，真正难的是“工程化地写 SQL”。

---

## 10. 下一步学习清单

- [ ] 系统梳理 ORC 与 Parquet 的压缩和读取差异
- [ ] 深入理解 Hive on Tez / Spark 的执行差异
- [ ] 练习复杂窗口函数与多层 CTE 改写
- [ ] 形成一份个人的 Hive 故障排查手册

后续如果遇到新问题，我会继续把排查过程更新到这篇笔记里。
