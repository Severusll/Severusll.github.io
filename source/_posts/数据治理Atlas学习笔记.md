---
title: "数据治理Atlas学习笔记"
date: 2026-04-02 18:19:04
tags:
  - atlas
  - 大数据
  - 学习笔记
categories:
  - 大数据
thumbnail: "/images/thumbnails/atlas.png"
---

本文记录我在当前大数据集群中安装与使用 Atlas 2.1.0 的完整过程，包含环境规划、Solr/Atlas 安装、核心配置、启动顺序与元数据同步实践。

## 1. Atlas 概述

Apache Atlas 是一套开源的元数据治理平台，核心目标是帮助团队建立统一的数据资产目录，并提供分类、检索、血缘分析和协作治理能力。

我目前对 Atlas 的理解是：

- Atlas 不直接替代 Hive/HBase/Kafka，而是作为元数据管理中心；
- 它可以把库、表、字段、任务之间的关系串起来，形成可追溯的数据链路；
- 对数仓团队最实用的能力是元数据检索和数据血缘。

数据字典的价值：

- 可查看 Hive 库/表的业务含义；
- 可查看字段定义、口径说明、上下游依赖；
- 降低团队沟通成本，减少口径不一致问题。

## 2. 安装环境准备

Atlas 常见有两种部署方式：

1. 集成内置 HBase + Solr；
2. 集成外部 HBase + Solr。

在企业集群里通常采用第 2 种方案，便于统一运维与资源管理。本文使用外部 HBase + Solr。

### 2.1 集群服务规划

| 服务名称 | 子服务 | master | slave1 | slave2 |
| :--- | :--- | :---: | :---: | :---: |
| HDFS | NameNode | √ |  |  |
| HDFS | DataNode | √ | √ | √ |
| HDFS | SecondaryNameNode |  | √ |  |
| YARN | ResourceManager | √ |  |  |
| YARN | NodeManager | √ | √ | √ |
| YARN | JobHistoryServer | √ |  |  |
| ZooKeeper | QuorumPeerMain | √ | √ | √ |
| Kafka | Kafka Broker | √ | √ | √ |
| HBase | HMaster | √ |  |  |
| HBase | HRegionServer | √ | √ | √ |
| Solr | Solr 节点 | √ | √ | √ |
| Hive | HiveServer2 / Metastore | √ |  |  |
| MySQL | MySQL | √ |  |  |
| Atlas | Atlas Server | √ |  |  |

### 2.2 基础约定

- Solr 端口：8983；
- Atlas 端口：21000。

## 3. 安装 Solr 7.7.3

### 3.1 创建 solr 系统用户（所有节点）

```bash
sudo useradd solr
echo solr | sudo passwd --stdin solr
```

### 3.2 安装并重命名（以 master 节点为例）

```bash
cd /usr/local
tar -zxvf solr-7.7.3.tgz
mv solr-7.7.3 solr
sudo chown -R solr:solr /usr/local/solr
```

### 3.3 修改 Solr 配置

编辑 /usr/local/solr/bin/solr.in.sh，设置 ZooKeeper 地址：

```bash
ZK_HOST="master:2181,slave1:2181,slave2:2181"
```

### 3.4 分发到其他节点

可按你现有分发脚本或 rsync/scp 方式同步到 slave1、slave2，并保持目录为 /usr/local/solr。

### 3.5 启动 Solr 集群（所有节点）

```bash
sudo -i -u solr /usr/local/solr/bin/solr start
```

出现 **Happy Searching!** 说明启动成功。

### 3.6 访问 Web 页面

- 默认端口：8983；
- 访问示例：http://master:8983/solr。

## 4. 安装 Atlas 2.1.0

### 4.1 上传与解压

```bash
cd /usr/local
tar -zxvf apache-atlas-2.1.0-server.tar.gz
mv apache-atlas-2.1.0 atlas
```

## 5. Atlas 核心配置

以下配置文件均位于 /usr/local/atlas/conf。

### 5.1 Atlas 集成 HBase

在 atlas-application.properties 中配置：

```properties
atlas.graph.storage.hostname=master:2181,slave1:2181,slave2:2181
```

在 atlas-env.sh 中增加：

```bash
export HBASE_CONF_DIR=/usr/local/hbase/conf
```

### 5.2 Atlas 集成 Solr

在 atlas-application.properties 中配置：

```properties
atlas.graph.index.search.backend=solr
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.zookeeper-url=master:2181,slave1:2181,slave2:2181
```

创建 Solr Collection（在 master 上执行）：

```bash
sudo -i -u solr /usr/local/solr/bin/solr create -c vertex_index -d /usr/local/atlas/conf/solr -shards 3 -replicationFactor 2
sudo -i -u solr /usr/local/solr/bin/solr create -c edge_index -d /usr/local/atlas/conf/solr -shards 3 -replicationFactor 2
sudo -i -u solr /usr/local/solr/bin/solr create -c fulltext_index -d /usr/local/atlas/conf/solr -shards 3 -replicationFactor 2
```

### 5.3 Atlas 集成 Kafka

在 atlas-application.properties 中配置：

```properties
atlas.notification.embedded=false
atlas.kafka.data=/usr/local/kafka/data
atlas.kafka.zookeeper.connect=master:2181,slave1:2181,slave2:2181/kafka
atlas.kafka.bootstrap.servers=master:9092,slave1:9092,slave2:9092
```

### 5.4 Atlas Server 参数

在 atlas-application.properties 中配置：

```properties
#########  Server Properties  #########
atlas.rest.address=http://master:21000
atlas.server.run.setup.on.start=false

#########  Entity Audit Configs  #########
atlas.audit.hbase.tablename=apache_atlas_entity_audit
atlas.audit.zookeeper.session.timeout.ms=1000
atlas.audit.hbase.zookeeper.quorum=master:2181,slave1:2181,slave2:2181
```

### 5.5 开启性能日志（可选）

编辑 atlas-log4j.xml，放开 perf_appender 与 org.apache.atlas.perf 对应 logger 注释。

### 5.6 Atlas 集成 Hive Hook

1. 在 atlas-application.properties 中增加：

```properties
atlas.hook.hive.synchronous=false
atlas.hook.hive.numRetries=3
atlas.hook.hive.queueSize=10000
atlas.cluster.name=primary
```

2. 在 /usr/local/hive/conf/hive-site.xml 增加：

```xml
<property>
  <name>hive.exec.post.hooks</name>
  <value>org.apache.atlas.hive.hook.HiveHook</value>
</property>
```

3. 安装 Hive Hook：

```bash
tar -zxvf apache-atlas-2.1.0-hive-hook.tar.gz
cp -r apache-atlas-hive-hook-2.1.0/* /usr/local/atlas/
```

4. 修改 /usr/local/hive/conf/hive-env.sh（若无则由模板复制）：

```bash
export HIVE_AUX_JARS_PATH=/usr/local/atlas/hook/hive
```

5. 拷贝 Atlas 配置到 Hive：

```bash
cp /usr/local/atlas/conf/atlas-application.properties /usr/local/hive/conf/
```

## 6. Atlas 启动与验证

### 6.1 依赖服务启动顺序

```text
Hadoop(HDFS/YARN) -> ZooKeeper -> Kafka -> HBase -> Solr -> Atlas
```

### 6.2 启动命令

```bash
# 1) Hadoop
start-all.sh

# 2) ZooKeeper
zk-cluster.sh start

# 3) Kafka
kf.sh start

# 4) HBase
start-hbase.sh

# 5) Solr（所有节点）
sudo -i -u solr /usr/local/solr/bin/solr start

# 6) Atlas（master）
cd /usr/local/atlas
python2 bin/atlas_start.py
```

停止 Atlas：

```bash
cd /usr/local/atlas
python2 bin/atlas_stop.py
```

### 6.3 Web UI 访问

- 访问地址：http://master:21000 
- 注意：等待时间大概2分钟。 
- 默认账户：admin 
- 默认密码：admin 
![Atlas Web UI 界面](/images/atlas-UI.png)


### 6.4 常见排查

```bash
# 查看 Atlas 日志
tail -f /usr/local/atlas/logs/application.log

# 查看 Solr 端口
netstat -anp | grep 8983

# 查看 Atlas 端口
netstat -anp | grep 21000
```

## 7. Atlas 使用实践

### 7.1 Hive 元数据初次导入

执行 Hive 全量导入脚本：

```bash
/usr/local/atlas/hook-bin/import-hive.sh
```

按提示输入：

- 用户名：admin
- 密码：admin

出现 **Hive Meta Data import was successful!!!** 表示导入成功。

### 7.2 Hive 元数据增量同步

增量同步由 Hive Hook 自动完成，无需人工干预：

- 执行 DDL（建库、建表、加字段）会同步结构变更；
- 执行 DML（如 insert into ... select ...）会逐步形成血缘关系。

### 7.3 血缘为什么一开始看不到

Atlas 的血缘依赖来自实际执行的 SQL。若仅完成安装但尚未跑过关联 SQL，图谱中通常不会出现完整上下游。

例如执行以下语句后，可观察 table_b 到 table_a 的血缘：

```sql
INSERT INTO table_a
SELECT *
FROM table_b;
```

## 8. 小结

这次 Atlas 落地的关键点是三件事：

1. 路径、用户、主机名保持全局一致（/usr/local、hadoop、master/slave1/slave2）；
2. 先打通 Atlas 与 HBase/Solr/Kafka/Hive Hook 的配置链路；
3. 通过一次全量导入 + 后续 SQL 增量，逐步完善元数据与血缘图谱。

后续我会继续补充：分类体系（Classification）、术语（Glossary）和权限治理（Ranger 集成）实战。