---
title: "Spark 学习笔记"
date: 2026-03-21 16:00:00
tags:
  - spark
  - 大数据
  - 学习笔记
categories:
  - 大数据
thumbnail: "/images/thumbnails/spark.png"
---

> 这篇笔记记录我学习 Spark 的过程。  
> 我希望自己不仅会写作业式代码，还能看懂执行过程，遇到失败时能快速定位原因。

---

## 1. 我现在怎么理解 Spark

一开始我把 Spark 看成一个更快的 MapReduce。这个理解不算错，但太浅了。

后来我慢慢形成了一个更实用的认知：

- Spark 是一个通用分布式计算引擎；
- Spark SQL、DataFrame、Structured Streaming 都是它的使用入口；
- 真正难点不在 API，而在数据规模、分区策略、shuffle 和资源管理。

我现在给自己的目标是：

1. 写出来的任务先保证正确；
2. 能解释任务为什么慢；
3. 能通过日志和 UI 进行基本排障。

---

## 2. 本地开发最小样例（PySpark）

我常用这个最小模板验证环境和逻辑：

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = (
    SparkSession.builder
    .appName("spark-learning-demo")
    .master("local[*]")
    .getOrCreate()
)

data = [
    ("u1", "click", 1),
    ("u1", "pay", 3),
    ("u2", "click", 2),
    ("u3", "click", 1),
]

df = spark.createDataFrame(data, ["user_id", "action", "cnt"])

result = (
    df.groupBy("user_id")
      .agg(F.sum("cnt").alias("total_cnt"))
      .orderBy(F.col("total_cnt").desc())
)

result.show()
spark.stop()
```

这个例子很小，但适合反复练习：`读取 -> 转换 -> 聚合 -> 输出`。

---

## 3. RDD、DataFrame、Dataset 的使用边界

### 3.1 我自己的选择顺序

- 优先 `DataFrame` / `Spark SQL`；
- 需要底层控制时再考虑 `RDD`；
- `Dataset` 更常见于 Scala/Java 体系。

### 3.2 一个常见误区

我以前会直接上 `RDD`，觉得更灵活。后来发现：

- DataFrame 的优化空间更大（Catalyst + Tungsten）；
- 代码表达更接近业务语义；
- 调优和排查时，执行计划更可读。

所以我现在除非必须，不会先走 RDD 路线。

---

## 4. Spark SQL 与执行计划

### 4.1 先写清楚 SQL

```sql
SELECT
  dt,
  user_id,
  COUNT(*) AS action_cnt
FROM dwd_user_action
WHERE dt BETWEEN '2026-03-20' AND '2026-03-26'
GROUP BY dt, user_id;
```

### 4.2 再看执行计划

```python
spark.sql("""
SELECT dt, user_id, COUNT(*) AS action_cnt
FROM dwd_user_action
WHERE dt BETWEEN '2026-03-20' AND '2026-03-26'
GROUP BY dt, user_id
""").explain(True)
```

我主要关注：

- 是否有 `Exchange`（通常意味着 shuffle）；
- 过滤条件有没有下推；
- join 走的是 Broadcast 还是 SortMerge；
- 计划是否出现明显的全量扫描。

**我的体会：**解释计划不是可选项，任务慢的时候它几乎是第一入口。

---

## 5. Shuffle、分区与数据倾斜

### 5.1 为什么 shuffle 常常是性能分界点

shuffle 会带来：

- 网络传输；
- 磁盘 IO；
- 内存压力与任务长尾。

如果上游分区设计不好，后面的聚合和 join 会明显变慢。

### 5.2 我常用的检查项

- `spark.sql.shuffle.partitions` 是否过大或过小；
- 某些 key 是否特别热（单个分区数据过大）；
- 任务是否出现明显 straggler（个别 task 特别慢）。

### 5.3 数据倾斜的一个处理思路

当 key 倾斜明显时，我会尝试：

1. 对热点 key 做拆分（加盐）；
2. 非热点数据正常聚合；
3. 最后再做一次汇总。

这不是万能方案，但在练习里确实能明显改善长尾。

---

## 6. Join 优化：我常踩的坑和修正

### 6.1 广播 join

小表 join 大表时，优先考虑广播小表：

```python
from pyspark.sql.functions import broadcast

result = big_df.join(broadcast(dim_df), on="item_id", how="left")
```

### 6.2 常见坑

| 问题 | 表现 | 修正方向 |
| :-- | :-- | :-- |
| join 前不做过滤 | 数据量过大，shuffle 激增 | 先过滤再 join |
| 字段类型不一致 | 结果异常或计划退化 | 显式 cast 统一类型 |
| 盲目 `repartition` | 额外开销，反而更慢 | 根据数据规模和 key 决定 |

### 6.3 我的经验

join 之前先回答两个问题：

- 能不能先减数据量？
- 能不能让一个表足够小，触发广播？

这两个问题能挡住很多低效写法。

---

## 7. Cache/Persist：不是所有 DataFrame 都要缓存

我以前写任务有个坏习惯：到处 `cache()`。后来才发现缓存也有成本。

### 7.1 什么时候适合缓存

- 同一个中间结果被重复使用；
- 计算链条长，反复触发代价大；
- 内存资源可以承受。

### 7.2 一个常用模式

```python
mid_df = heavy_df.filter("dt >= '2026-03-20'").select("user_id", "item_id", "score")
mid_df.persist()

mid_df.groupBy("user_id").count().show()
mid_df.groupBy("item_id").count().show()

mid_df.unpersist()
```

**我的反思：**缓存要有证据，不要凭感觉。

---

## 8. 错误排查记录（最像学习笔记的部分）

下面这些问题我都实际遇到过，或者在练习中复现过。

### 8.1 现象：`Java heap space` / executor OOM

**排查路径：**

1. 看失败 stage 的输入数据量；
2. 看是否有超大分区或倾斜 key；
3. 看是否有 `collect()`、`toPandas()` 这类高风险操作；
4. 看 executor memory 和 cores 配比是否失衡。

**修复方向：**

- 避免把大结果拉回 driver；
- 调整分区和聚合策略；
- 必要时增加 executor 内存并优化并发度。

### 8.2 现象：任务卡在某个 stage 很久

**常见根因：**

- 数据倾斜导致少数 task 极慢；
- 下游依赖外部存储写入慢；
- 不合理的 wide transformation 过多。

**我的处理顺序：**先看 Spark UI 的 stage/task 分布，再回到代码定位算子。

### 8.3 现象：序列化相关错误（Task not serializable）

我早期在闭包里引用了不可序列化对象，导致任务失败。

**经验总结：**

- 避免在算子里引用复杂外部对象；
- 函数尽量纯净；
- 必要时用广播变量传小型只读配置。

### 8.4 现象：写 Hive 分区表后查不到数据

这个问题在 Hive 和 Spark 混用场景很常见。

**排查要点：**

- 分区字段值是否写对；
- 写入路径和表 location 是否一致；
- metastore 是否同步更新；
- 查询条件格式是否和分区格式一致。

---

## 9. 我给自己的 Spark 学习规范

- 每个作业先写目标输入输出，不直接开写代码；
- 关键 DataFrame 打印 schema，防止隐式类型问题；
- 每次性能问题至少保留一次 `explain` 结果；
- 失败任务记录四件事：现象、根因、修复、预防；
- 对重要参数改动做备注，避免下次重复踩坑。

这套规范执行后，我最明显的变化是排查速度变快了。

---

## 10. 阶段总结

目前我对 Spark 的学习重点有三个：

1. 任务正确性（口径和结果）；
2. 任务可解释性（计划和指标）；
3. 任务稳定性（失败可复现、可定位、可修复）。

我现在不再追求一次把代码写得很复杂，而是先把链路跑通，再一点点优化瓶颈。

---

