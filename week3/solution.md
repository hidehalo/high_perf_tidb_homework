# 高性能TiDB课程作业（第三周）

分值：1 个有效 issue 100，有效 PR 根据实际效果进行相应加分，比如能节省 CPU、减少内存占用、减少 IO 次数等。

题目描述：

使用上一节可以讲的 sysbench、go-ycsb 或者 go-tpc 对 TiDB 进行压力测试， 然后对 TiDB 或 TiKV 的 CPU 、内存或 IO 进行 proﬁle，寻找潜在可以优化的地 方并提 enhance 类型的 issue 描述。

issue 描述应包含：

- 部署环境的机器配置（CPU、内存、磁盘规格型号），拓扑结构（TiDB、TiKV 各部署于哪些节点）
- 跑的 workload 情况
- 对应的 proﬁle 看到的情况
- 建议如何优化？

【可选】提 PR 进行优化：

- 按照 PR 模板提交优化 PR

**输出**：对 TiDB 或 TiKV 进行 proﬁle，写文章描述 分析的过程，对于可以优化的地方提 issue 描述，并将 issue 贴到文章中（或【可选】提 PR 进行优化，将 PR 贴到文章中）

截止时间：9.1 24:00（逾期提交不给分）

## 1. 机器配置

- CPU 3.1 GHz 四核Intel Core i5
- 内存 16 GB 1867 MHz DDR3
- 磁盘规格型号 APPLE HDD HTS541010A9E662 1 TB 5400 RPM

## 2. 拓扑结构

| IP地址             | 角色                  | 操作系统        | 核数          | 内存|
| ---               | ---                   | ---           | ---           | ---|
| 192.168.99.101    | TiDB Server           | CentOS 8.3.1  | 1vCPU         | 4GB|
| 192.168.99.102    | TiKV Server           | CentOS 8.3.1  | 4vCPU         | 4GB|
| 192.168.99.103    | PD Server             | CentOS 8.3.1  | 2vCPU         | 1GB|
| 192.168.99.103    | Prometheus Server     | -             | -             | -  |
| 192.168.99.103    | Grafana Server        | -             | -             | -  |
| 192.168.99.103    | Alertmanager Server   | -             | -             | -  |

### 2.1 TiDB&TiKV服务器配置

```yaml
server_configs:
  tidb:
    log.slow-threshold: 300
    binlog.enable: false
    binlog.ignore-error: false
  tikv:
    server.grpc-concurrency: 4
    rocksdb.max-background-jobs: 4
    raftdb.max-background-jobs: 4
    readpool.unified.max-thread-count: 4
    readpool.storage.use-unified-pool: true
    readpool.coprocessor.use-unified-pool: true
```

## 3. 使用Go-YCSB进行压测

Go-YCSB共有a-f六种负载，每种负载都由至少一种SQL命令按比例组合而成，不进行投影，请求量/时间呈正态分布。

## 3.1 数据模式及数据集

usertable (YCSB_KEY, FIELD0, FIELD1, FIELD2, FIELD3, FIELD4, FIELD5, FIELD6, FIELD7, FIELD8, FIELD9)

usertable表共有3M条记录

```sql
CREATE TABLE usertable (
  YCSB_KEY VARCHAR(64) PRIMARY KEY,
  FIELD0 VARCHAR(100),
  FIELD1 VARCHAR(100),
  FIELD2 VARCHAR(100),
  FIELD3 VARCHAR(100),
  FIELD4 VARCHAR(100),
  FIELD5 VARCHAR(100),
  FIELD6 VARCHAR(100),
  FIELD7 VARCHAR(100),
  FIELD8 VARCHAR(100),
  FIELD9 VARCHAR(100),
)
```

## 3.2 SQL模板

Go-YCSB负载有Read(点查询)、Scan(范围查询)、Insert、Update(点查询)、Delete(点查询)，以下会列出对应的SQL模板。

### 3.2.1 Read

```sql
SELECT $fields FROM $table $forceIndexKey WHERE YCSB_KEY = ?
```

### 3.2.2 Scan

```sql
SELECT $fields FROM $table $forceIndexKey WHERE YCSB_KEY >= ? LIMIT ?
```

### 3.2.3 Insert

```sql
INSERT IGNORE INTO $table ($field1, $field2, ...) VALUES (?, ?, ...)
```

### 3.2.4 Update

```sql
UPDATE $table set $field1 = ?, $field2 = ? ... WHERE YCSB_KEY = ?
```

### 3.2.5 Delete

```sql
DELETE FROM $table WHERE YCSB_KEY = ?
```

## 3.3 TiDB Profile采集与分析

### 3.3.1 Workload a

```bash
# Yahoo! Cloud System Benchmark
# Workload A: Update heavy workload
#   Application example: Session store recording recent actions
#                        
#   Read/update ratio: 50/50
#   Default data size: 1 KB records (10 fields, 100 bytes each, plus key)
#   Request distribution: zipfian

recordcount=1000
operationcount=1000
workload=core

readallfields=true

readproportion=0.5
updateproportion=0.5
scanproportion=0
insertproportion=0

requestdistribution=uniform
```

### 3.3.1.1 TiDB Profile

![YCSB workload a TiDB profile](./profiles/ycsb/tidb/a.svg)

### 3.3.1.2 分析

### 3.3.1.3 结论

### 3.3.2 Workload b

```bash
# Yahoo! Cloud System Benchmark
# Workload B: Read mostly workload
#   Application example: photo tagging; add a tag is an update, but most operations are to read tags
#                        
#   Read/update ratio: 95/5
#   Default data size: 1 KB records (10 fields, 100 bytes each, plus key)
#   Request distribution: zipfian

recordcount=1000
operationcount=1000
workload=core

readallfields=true

readproportion=0.95
updateproportion=0.05
scanproportion=0
insertproportion=0

requestdistribution=uniform
```

### 3.3.2.1 TiDB Profile

![YCSB workload b TiDB profile](./profiles/ycsb/tidb/b.svg)

### 3.3.2.2 分析

### 3.3.2.3 结论

### 3.3.3 Workload c

```bash
# Yahoo! Cloud System Benchmark
# Workload C: Read only
#   Application example: user profile cache, where profiles are constructed elsewhere (e.g., Hadoop)
#                        
#   Read/update ratio: 100/0
#   Default data size: 1 KB records (10 fields, 100 bytes each, plus key)
#   Request distribution: zipfian

recordcount=1000
operationcount=1000
workload=core

readallfields=true

readproportion=1
updateproportion=0
scanproportion=0
insertproportion=0

requestdistribution=uniform
```

### 3.3.3.1 TiDB Profile

![YCSB workload c TiDB profile](./profiles/ycsb/tidb/c.svg)

### 3.3.3.2 分析

### 3.3.3.3 结论

### 3.3.4 Workload d

```bash
# Yahoo! Cloud System Benchmark
# Workload D: Read latest workload
#   Application example: user status updates; people want to read the latest
#                        
#   Read/update/insert ratio: 95/0/5
#   Default data size: 1 KB records (10 fields, 100 bytes each, plus key)
#   Request distribution: latest

# The insert order for this is hashed, not ordered. The "latest" items may be 
# scattered around the keyspace if they are keyed by userid.timestamp. A workload
# which orders items purely by time, and demands the latest, is very different than 
# workload here (which we believe is more typical of how people build systems.)

recordcount=1000
operationcount=1000
workload=core

readallfields=true

readproportion=0.95
updateproportion=0
scanproportion=0
insertproportion=0.05

requestdistribution=latest
```

### 3.3.4.1 TiDB Profile

![YCSB workload d TiDB profile](./profiles/ycsb/tidb/d.svg)

### 3.3.4.2 分析

### 3.3.4.3 结论

### 3.3.5 Workload e

```bash
# Yahoo! Cloud System Benchmark
# Workload E: Short ranges
#   Application example: threaded conversations, where each scan is for the posts in a given thread (assumed to be clustered by thread id)
#                        
#   Scan/insert ratio: 95/5
#   Default data size: 1 KB records (10 fields, 100 bytes each, plus key)
#   Request distribution: zipfian

# The insert order is hashed, not ordered. Although the scans are ordered, it does not necessarily
# follow that the data is inserted in order. For example, posts for thread 342 may not be inserted contiguously, but
# instead interspersed with posts from lots of other threads. The way the YCSB client works is that it will pick a start
# key, and then request a number of records; this works fine even for hashed insertion.

recordcount=1000
operationcount=1000
workload=core

readallfields=true

readproportion=0
updateproportion=0
scanproportion=0.95
insertproportion=0.05

requestdistribution=uniform

maxscanlength=1

scanlengthdistribution=uniform
```

### 3.3.5.1 TiDB Profile

![YCSB workload e TiDB profile](./profiles/ycsb/tidb/e.svg)

### 3.3.5.2 分析

### 3.3.5.3 结论

### 3.3.6 Workload f

```bash
# Yahoo! Cloud System Benchmark
# Workload F: Read-modify-write workload
#   Application example: user database, where user records are read and modified by the user or to record user activity.
#                        
#   Read/read-modify-write ratio: 50/50
#   Default data size: 1 KB records (10 fields, 100 bytes each, plus key)
#   Request distribution: zipfian

recordcount=1000
operationcount=1000
workload=core

readallfields=true

readproportion=0.5
updateproportion=0
scanproportion=0
insertproportion=0
readmodifywriteproportion=0.5

requestdistribution=uniform
```

### 3.3.6.1 TiDB Profile

![YCSB workload f TiDB profile](./profiles/ycsb/tidb/f.svg)

### 3.3.6.2 分析

### 3.3.6.3 结论
