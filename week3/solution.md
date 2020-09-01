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

### 2.1 服务器配置

```yaml
server_configs:
  tidb:
    binlog.enable: false
    binlog.ignore-error: false
    enable-streaming: true
    log.enable-slow-log: false
    log.slow-threshold: 300
    oom-use-tmp-storage: false
    performance.tcp-keep-alive: true
    prepared-plan-cache.enabled: true
  tikv:
    backup.num-threads: 1
    pessimistic-txn.pipelined: true
    raftdb.max-background-jobs: 4
    readpool.coprocessor.use-unified-pool: true
    readpool.storage.use-unified-pool: true
    readpool.unified.max-thread-count: 4
    rocksdb.max-background-jobs: 4
    server.grpc-concurrency: 4
  pd:
    schedule.leader-schedule-limit: 2
    schedule.region-schedule-limit: 1024
    schedule.replica-schedule-limit: 32
```

## 3. 使用Go-YCSB进行压测及分析

Go-YCSB共有a-f六种负载，每种负载都由至少一种SQL命令按比例组合而成，不进行投影，请求量/时间呈正态分布。以下测试均采用256个线程模拟并发，本次报告的性能分析对象是TiDB。

```bash
./bin/go-ycsb run mysql -P workloads/workloadx
  -p operationcount=1000000
  -p mysql.host=192.168.99.101
  -p mysql.port=4000 --threads 256
```

### 3.1 数据模式及数据集大小

#### 3.1.1 数据模式

usertable(<u style="color:red;">YCSB_KEY</u>, FIELD0, FIELD1, FIELD2, FIELD3, FIELD4, FIELD5, FIELD6, FIELD7, FIELD8, FIELD9)

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
  PRIMARY KEY (`YCSB_KEY`)
);
```

#### 3.1.2 数据集大小

```sql
mysql> select count(1) from usertable;
```

```bash
+----------+
| count(1) | 
+----------+
|   999936 |
+----------+
```

### 3.2 Workload涉及的SQL

Go-YCSB负载有`Read`(点查询)、`Scan`(范围查询)、`Insert`、`Update`(点查询)、`Delete`(点查询)，以下会列出对应的SQL模板。

#### 3.2.1 Read

```sql
SELECT $fields FROM $table $forceIndexKey WHERE YCSB_KEY = ?
```

#### 3.2.2 Scan

```sql
SELECT $fields FROM $table $forceIndexKey WHERE YCSB_KEY >= ? LIMIT ?
```

#### 3.2.3 Insert

```sql
INSERT IGNORE INTO $table ($field1, $field2, ...) VALUES (?, ?, ...)
```

#### 3.2.4 Update

```sql
UPDATE $table set $field1 = ?, $field2 = ? ... WHERE YCSB_KEY = ?
```

#### 3.2.5 Delete

```sql
DELETE FROM $table WHERE YCSB_KEY = ?
```

### 3.3 TiDB profile采集与分析

#### 3.3.1 Workload a

##### 3.3.1.1 负载配置

```bash
"threadcount"="256"
"mysql.host"="192.168.99.101"
"updateproportion"="0.5"
"insertproportion"="0"
"mysql.port"="4000"
"workload"="core"
"requestdistribution"="uniform"
"dotransactions"="true"
"recordcount"="1000"
"operationcount"="1000000"
"scanproportion"="0"
"readproportion"="0.5"
"readallfields"="true"
```

##### 3.3.1.2 压测结果

```bash
READ   - Takes(s): 215.8, Count: 115432, OPS: 535.0, Avg(us): 115576, Min(us): 1795, Max(us): 5059776, 99th(us): 357000, 99.9th(us): 1072000, 99.99th(us): 2871000
UPDATE - Takes(s): 211.6, Count: 114645, OPS: 541.9, Avg(us): 347580, Min(us): 8232, Max(us): 13621982, 99th(us): 1335000, 99.9th(us): 4250000, 99.99th(us): 7361000
```

##### 3.3.1.3 CPU

```bash
go tool pprof -nodecount 20 -cum -tree profile

Duration: 2mins, Total samples = 76.57s (63.73%)
Showing nodes accounting for 17.33s, 22.63% of 76.57s total
Dropped 1095 nodes (cum <= 0.38s)
Showing top 20 nodes out of 293
----------------------------------------------------------+-------------
      flat  flat%   sum%        cum   cum%   calls calls% + context 	 	 
----------------------------------------------------------+-------------
         0     0%     0%     51.76s 67.60%                | github.com/pingcap/tidb/server.(*Server).onConn
                                            51.76s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
----------------------------------------------------------+-------------
                                            51.76s   100% |   github.com/pingcap/tidb/server.(*Server).onConn
     0.16s  0.21%  0.21%     51.76s 67.60%                | github.com/pingcap/tidb/server.(*clientConn).Run
                                            48.70s 94.09% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
                                             1.34s  2.59% |   syscall.Syscall
----------------------------------------------------------+-------------
                                            48.70s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
     0.10s  0.13%  0.34%     48.70s 63.60%                | github.com/pingcap/tidb/server.(*clientConn).dispatch
                                            47.72s 97.99% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
----------------------------------------------------------+-------------
                                            47.72s   100% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
     0.08s   0.1%  0.44%     47.72s 62.32%                | github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
                                            30.77s 64.48% |   github.com/pingcap/tidb/server.(*TiDBStatement).Execute
                                            11.97s 25.08% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                                4s  8.38% |   net.(*conn).Write
----------------------------------------------------------+-------------
                                            30.77s   100% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
         0     0%  0.44%     30.77s 40.19%                | github.com/pingcap/tidb/server.(*TiDBStatement).Execute
                                            30.76s   100% |   github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
----------------------------------------------------------+-------------
                                            30.76s   100% |   github.com/pingcap/tidb/server.(*TiDBStatement).Execute
     0.07s 0.091%  0.54%     30.76s 40.17%                | github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
                                            26.58s 86.41% |   github.com/pingcap/tidb/session.(*session).CachedPlanExec
                                             1.87s  6.08% |   github.com/pingcap/tidb/session.runStmt
----------------------------------------------------------+-------------
                                            25.18s 93.05% |   github.com/pingcap/tidb/session.(*session).CachedPlanExec
                                             1.87s  6.91% |   github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
     0.06s 0.078%  0.61%     27.06s 35.34%                | github.com/pingcap/tidb/session.runStmt
                                            12.45s 46.01% |   github.com/pingcap/tidb/executor.(*ExecStmt).Exec
                                            12.24s 45.23% |   github.com/pingcap/tidb/session.finishStmt
----------------------------------------------------------+-------------
                                            26.58s   100% |   github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
     0.03s 0.039%  0.65%     26.58s 34.71%                | github.com/pingcap/tidb/session.(*session).CachedPlanExec
                                            25.18s 94.73% |   github.com/pingcap/tidb/session.runStmt
----------------------------------------------------------+-------------
                                            14.48s 85.33% |   syscall.write
                                             1.34s  7.90% |   github.com/pingcap/tidb/server.(*clientConn).Run
    16.35s 21.35% 22.01%     16.97s 22.16%                | syscall.Syscall
----------------------------------------------------------+-------------
                                            14.11s 95.60% |   net.(*netFD).Write
                                             0.47s  3.18% |   github.com/pingcap/tidb/session.(*session).doCommitWithRetry
     0.02s 0.026% 22.03%     14.76s 19.28%                | internal/poll.(*FD).Write
                                            14.50s 98.24% |   syscall.Write
----------------------------------------------------------+-------------
                                            14.50s   100% |   internal/poll.(*FD).Write
         0     0% 22.03%     14.50s 18.94%                | syscall.Write
                                            14.50s   100% |   syscall.write
----------------------------------------------------------+-------------
                                            14.50s   100% |   syscall.Write
     0.02s 0.026% 22.06%     14.50s 18.94%                | syscall.write
                                            14.48s 99.86% |   syscall.Syscall
----------------------------------------------------------+-------------
                                             5.33s 37.56% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                                4s 28.19% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
     0.03s 0.039% 22.10%     14.19s 18.53%                | net.(*conn).Write
                                            14.16s 99.79% |   net.(*netFD).Write
----------------------------------------------------------+-------------
                                            14.16s   100% |   net.(*conn).Write
     0.05s 0.065% 22.16%     14.16s 18.49%                | net.(*netFD).Write
                                            14.11s 99.65% |   internal/poll.(*FD).Write
----------------------------------------------------------+-------------
                                            12.45s 88.36% |   github.com/pingcap/tidb/session.runStmt
                                             1.64s 11.64% |   github.com/pingcap/tidb/session.(*session).doCommitWithRetry
     0.12s  0.16% 22.32%     14.09s 18.40%                | github.com/pingcap/tidb/executor.(*ExecStmt).Exec
                                            10.45s 74.17% |   github.com/pingcap/tidb/executor.Next
----------------------------------------------------------+-------------
                                            10.45s 76.73% |   github.com/pingcap/tidb/executor.(*ExecStmt).Exec
                                             3.15s 23.13% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
     0.09s  0.12% 22.44%     13.62s 17.79%                | github.com/pingcap/tidb/executor.Next
----------------------------------------------------------+-------------
                                            12.24s   100% |   github.com/pingcap/tidb/session.runStmt
     0.01s 0.013% 22.45%     12.24s 15.99%                | github.com/pingcap/tidb/session.finishStmt
                                            12.23s 99.92% |   github.com/pingcap/tidb/session.(*session).CommitTxn
----------------------------------------------------------+-------------
                                            12.23s   100% |   github.com/pingcap/tidb/session.finishStmt
     0.05s 0.065% 22.52%     12.23s 15.97%                | github.com/pingcap/tidb/session.(*session).CommitTxn
                                            12.02s 98.28% |   github.com/pingcap/tidb/session.(*session).doCommitWithRetry
----------------------------------------------------------+-------------
                                            12.02s   100% |   github.com/pingcap/tidb/session.(*session).CommitTxn
     0.06s 0.078% 22.59%     12.02s 15.70%                | github.com/pingcap/tidb/session.(*session).doCommitWithRetry
                                             1.64s 13.64% |   github.com/pingcap/tidb/executor.(*ExecStmt).Exec
                                             0.47s  3.91% |   internal/poll.(*FD).Write
----------------------------------------------------------+-------------
                                            11.97s   100% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
     0.03s 0.039% 22.63%     11.97s 15.63%                | github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                             5.33s 44.53% |   net.(*conn).Write
                                             3.15s 26.32% |   github.com/pingcap/tidb/executor.Next
----------------------------------------------------------+-------------
```

```bash
go tool pprof -nodecount 20 -flat -text profile

Duration: 2mins, Total samples = 76.57s (63.73%)
Showing nodes accounting for 35.62s, 46.52% of 76.57s total
Dropped 1095 nodes (cum <= 0.38s)
Showing top 20 nodes out of 293
      flat  flat%   sum%        cum   cum%
    16.35s 21.35% 21.35%     16.97s 22.16%  syscall.Syscall
     2.49s  3.25% 24.60%      5.18s  6.77%  runtime.scanobject
     2.16s  2.82% 27.43%      2.40s  3.13%  runtime.findObject
     2.08s  2.72% 30.14%     10.39s 13.57%  runtime.mallocgc
     1.52s  1.99% 32.13%      1.52s  1.99%  runtime.nextFreeFast
     1.42s  1.85% 33.98%      2.01s  2.63%  runtime.heapBitsSetType
     1.26s  1.65% 35.63%      1.26s  1.65%  runtime.memclrNoHeapPointers
     1.22s  1.59% 37.22%      7.76s 10.13%  runtime.newobject
     1.12s  1.46% 38.68%      1.12s  1.46%  runtime.memmove
     0.79s  1.03% 39.72%      0.80s  1.04%  runtime.(*itabTableType).find
     0.73s  0.95% 40.67%      2.36s  3.08%  runtime.selectgo
     0.70s  0.91% 41.58%      0.75s  0.98%  context.(*valueCtx).Value
     0.59s  0.77% 42.35%      0.59s  0.77%  runtime.lock
     0.54s  0.71% 43.06%      1.34s  1.75%  runtime.getitab
     0.54s  0.71% 43.76%      0.63s  0.82%  runtime.step
     0.45s  0.59% 44.35%      0.48s  0.63%  time.now
     0.43s  0.56% 44.91%      0.75s  0.98%  runtime.mapaccess2
     0.41s  0.54% 45.45%      0.41s  0.54%  github.com/prometheus/client_golang/prometheus.(*histogram).observe
     0.41s  0.54% 45.98%      0.41s  0.54%  runtime.duffcopy
     0.41s  0.54% 46.52%      2.17s  2.83%  runtime.gentraceback
```

从profile中可以得到以下信息：

1. TiDB的CPU使用率较高
2. syscall.Syscall消耗了大量的CPU时间
3. TiKV事务的提交冲突导致消耗了较多CPU时间，并引起了syscall.Syscall的开销
4. GC消耗了较多的CPU时间
5. 网络IO消耗了较多的CPU时间，并引起了syscall.Syscall的开销

##### 3.3.1.4 内存

```bash
go tool pprof -nodecount 20 -flat -text -alloc_space heap

Showing nodes accounting for 8.26GB, 61.25% of 13.49GB total
Dropped 1151 nodes (cum <= 0.07GB)
Showing top 20 nodes out of 259
      flat  flat%   sum%        cum   cum%
    3.03GB 22.47% 22.47%     3.03GB 22.47%  github.com/pingcap/tidb/statistics/handle.statsCache.copy
    0.68GB  5.05% 27.52%     0.68GB  5.05%  github.com/pingcap/tidb/util/chunk.newFixedLenColumn
    0.61GB  4.52% 32.04%     0.61GB  4.52%  github.com/pingcap/tidb/util/chunk.newVarLenColumn
    0.45GB  3.34% 35.37%     0.45GB  3.35%  fmt.Sprintf
    0.45GB  3.32% 38.70%     0.45GB  3.32%  github.com/prometheus/client_golang/prometheus.(*histogram).Write
    0.39GB  2.88% 41.58%     4.89GB 36.25%  github.com/pingcap/tidb/statistics/handle.(*Handle).HandleAutoAnalyze
    0.36GB  2.71% 44.28%     0.73GB  5.39%  github.com/pingcap/tidb/statistics.PseudoTable
    0.30GB  2.26% 46.54%     0.32GB  2.34%  github.com/pingcap/tidb/executor.(*indexWorker).extractTaskHandles
    0.30GB  2.22% 48.77%     0.30GB  2.22%  github.com/pingcap/tidb/infoschema.(*infoSchema).SchemaTables
    0.19GB  1.43% 50.19%     0.19GB  1.43%  github.com/pingcap/tidb/executor.colNames2ResultFields
    0.17GB  1.25% 51.44%     1.36GB 10.11%  github.com/pingcap/tidb/util/chunk.New
    0.17GB  1.24% 52.68%     0.17GB  1.24%  github.com/pingcap/tidb/kv/memdb.newArenaBlock
    0.17GB  1.23% 53.91%     0.19GB  1.41%  github.com/pingcap/parser.yyParse
    0.16GB  1.18% 55.09%     0.16GB  1.18%  google.golang.org/grpc.(*parser).recvMsg
    0.16GB  1.16% 56.25%     0.16GB  1.19%  github.com/pingcap/tidb/util/chunk.(*Column).AppendBytes
    0.14GB  1.04% 57.29%     0.16GB  1.18%  google.golang.org/grpc/internal/transport.(*http2Client).Write
    0.14GB  1.04% 58.33%     0.23GB  1.68%  github.com/pingcap/tidb/planner/core.(*PlanBuilder).buildDataSource
    0.13GB     1% 59.32%     0.18GB  1.37%  github.com/pingcap/tidb/executor.ResetContextOfStmt
    0.13GB  0.99% 60.31%     0.13GB  0.99%  github.com/pingcap/tidb/expression.(*Column).Clone
    0.13GB  0.94% 61.25%     0.36GB  2.69%  github.com/pingcap/tidb/statistics.NewHistogram
```

```bash
go tool pprof -nodecount 20 -cum -text -alloc_space heap

Showing nodes accounting for 3534.82MB, 25.60% of 13808.86MB total
Dropped 1151 nodes (cum <= 69.04MB)
Showing top 20 nodes out of 259
      flat  flat%   sum%        cum   cum%
         0     0%     0%  5006.36MB 36.25%  github.com/pingcap/tidb/domain.(*Domain).autoAnalyzeWorker
  398.02MB  2.88%  2.88%  5006.36MB 36.25%  github.com/pingcap/tidb/statistics/handle.(*Handle).HandleAutoAnalyze
         0     0%  2.88%  3688.60MB 26.71%  github.com/pingcap/tidb/statistics/handle.(*Handle).GetPartitionStats
         0     0%  2.88%  3688.60MB 26.71%  github.com/pingcap/tidb/statistics/handle.(*Handle).GetTableStats
   13.67MB 0.099%  2.98%  3116.30MB 22.57%  github.com/pingcap/tidb/statistics/handle.statsCache.update
 3102.63MB 22.47% 25.45%  3102.63MB 22.47%  github.com/pingcap/tidb/statistics/handle.statsCache.copy
         0     0% 25.45%  2190.07MB 15.86%  github.com/pingcap/tidb/server.(*Server).onConn
         0     0% 25.45%  2183.02MB 15.81%  github.com/pingcap/tidb/server.(*clientConn).Run
         0     0% 25.45%  2175.02MB 15.75%  github.com/pingcap/tidb/server.(*clientConn).dispatch
   11.50MB 0.083% 25.53%  2034.20MB 14.73%  github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
       1MB 0.0072% 25.54%  1984.03MB 14.37%  github.com/pingcap/tidb/session.execRestrictedSQL
         0     0% 25.54%  1983.53MB 14.36%  github.com/pingcap/tidb/session.(*session).ExecRestrictedSQL
         0     0% 25.54%  1983.53MB 14.36%  github.com/pingcap/tidb/session.(*session).ExecRestrictedSQLWithContext
         0     0% 25.54%  1966.64MB 14.24%  github.com/pingcap/tidb/session.(*session).execute
         0     0% 25.54%  1965.14MB 14.23%  github.com/pingcap/tidb/session.(*session).Execute
         0     0% 25.54%  1933.84MB 14.00%  github.com/pingcap/tidb/executor.Next
       2MB 0.014% 25.55%  1808.17MB 13.09%  github.com/pingcap/tidb/session.runStmt
       6MB 0.043% 25.60%  1750.68MB 12.68%  github.com/pingcap/tidb/server.(*TiDBStatement).Execute
         0     0% 25.60%  1745.18MB 12.64%  github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
         0     0% 25.60%  1562.48MB 11.32%  github.com/pingcap/tidb/domain.(*Domain).loadStatsWorker
```

```bash
go tool pprof -nodecount 20 -flat -text -inuse_space heap

Showing nodes accounting for 54.94MB, 65.85% of 83.43MB total
Showing top 20 nodes out of 269
      flat  flat%   sum%        cum   cum%
      10MB 11.99% 11.99%    16.50MB 19.78%  github.com/pingcap/tidb/planner/core.buildSchemaFromFields
    7.74MB  9.27% 21.26%     7.74MB  9.27%  github.com/pingcap/tidb/util/arena.NewAllocator
       6MB  7.19% 28.45%        6MB  7.19%  github.com/pingcap/tidb/planner/core.colInfoToColumn
    4.06MB  4.87% 33.32%     4.06MB  4.87%  bufio.NewReaderSize
    3.50MB  4.20% 37.52%     4.50MB  5.39%  github.com/pingcap/parser.yyParse
    2.54MB  3.04% 40.56%     2.54MB  3.04%  bufio.NewWriterSize
    2.54MB  3.04% 43.61%     2.54MB  3.04%  github.com/pingcap/parser.New
    2.50MB  3.00% 46.60%    11.50MB 13.79%  github.com/pingcap/tidb/planner/core.buildPointUpdatePlan
       2MB  2.40% 49.00%        2MB  2.40%  github.com/pingcap/tidb/expression.foldConstant
    1.55MB  1.86% 50.86%     1.55MB  1.86%  bytes.makeSlice
    1.50MB  1.80% 52.66%        2MB  2.40%  github.com/pingcap/tidb/executor.ResetContextOfStmt
    1.50MB  1.80% 54.46%     2.50MB  3.00%  github.com/pingcap/tidb/expression.(*castAsStringFunctionClass).getFunction
    1.50MB  1.80% 56.26%     1.50MB  1.80%  github.com/pingcap/tidb/expression.ParamMarkerExpression
    1.50MB  1.80% 58.06%        8MB  9.59%  github.com/pingcap/tidb/server.(*TiDBContext).Prepare
    1.50MB  1.80% 59.85%        6MB  7.19%  github.com/pingcap/tidb/expression.BuildCastFunction
       1MB  1.20% 61.06%        1MB  1.20%  github.com/pingcap/tidb/sessionctx/variable.(*SessionVars).SetSystemVar
       1MB  1.20% 62.26%        1MB  1.20%  github.com/pingcap/tidb/util/rowcodec.(*row).toBytes
       1MB  1.20% 63.46%        1MB  1.20%  github.com/pingcap/tidb/planner/core.getIndexValues
       1MB  1.20% 64.66%        1MB  1.20%  github.com/pingcap/parser/model.NewExtraHandleColInfo
       1MB  1.20% 65.85%        1MB  1.20%  github.com/pingcap/tidb/statistics/handle.tableDeltaMap.update
```

从profile中可以得到以下信息：

1. handle.(*Handle).HandleAutoAnalyze引起了大量的内存申请需求

##### 3.3.1.5 IO

![Workload a disk performance](./profiles/ycsb/tidb/workloada/a_disk_perf.png)

![Workload a tikv summart](./profiles/ycsb/tidb/workloada/a_tikv_summary.png)

![Workload a tikv network](./profiles/ycsb/tidb/workloada/a_tikv_net.png)

根据以上监控数据可以得出：

1. TiKV结点的CPU负载很高
2. 压测中途磁盘IO出现了猛烈的下降，可能是由于`Update`引起的
3. 网络吞吐没有引起程序瓶颈
4. 磁盘写入IOPS较高

##### 3.3.1.6 结论

- Workload a是50%的`Read`操作和50%的`Update`操作组成的负载
- 在当前拓扑下，等待磁盘IO的开销占了不小比重，可能主要是由于设备磁盘性能的不足
- `Update`较重的负载下，TiDB的统计分析模块引起了CPU和内存非常大的开销

---

#### 3.3.2 Workload c

##### 3.3.2.1 负载配置

```bash
"updateproportion"="0"
"workload"="core"
"recordcount"="1000"
"readallfields"="true"
"threadcount"="256"
"requestdistribution"="uniform"
"insertproportion"="0"
"scanproportion"="0"
"mysql.port"="4000"
"dotransactions"="true"
"operationcount"="1000000"
"mysql.host"="192.168.99.101"
"readproportion"="1"
```

##### 3.3.2.2 压测结果

```bash
READ   - Takes(s): 399.5, Count: 999936, OPS: 2502.9, Avg(us): 97998, Min(us): 1923, Max(us): 20546032, 99th(us): 163000, 99.9th(us): 5863000, 99.99th(us): 20444000
```

##### 3.3.2.3 CPU

```bash
go tool pprof -nodecount 30 -cum -tree profile

Duration: 2mins, Total samples = 81.13s (67.53%)
Showing nodes accounting for 31.54s, 38.88% of 81.13s total
Dropped 803 nodes (cum <= 0.41s)
Showing top 30 nodes out of 218
----------------------------------------------------------+-------------
      flat  flat%   sum%        cum   cum%   calls calls% + context 	 	 
----------------------------------------------------------+-------------
         0     0%     0%     63.28s 78.00%                | github.com/pingcap/tidb/server.(*Server).onConn
                                            63.28s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
----------------------------------------------------------+-------------
                                            63.28s   100% |   github.com/pingcap/tidb/server.(*Server).onConn
     0.12s  0.15%  0.15%     63.28s 78.00%                | github.com/pingcap/tidb/server.(*clientConn).Run
                                            50.77s 80.23% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
                                            12.05s 19.04% |   github.com/pingcap/tidb/server.(*clientConn).readPacket
----------------------------------------------------------+-------------
                                            50.77s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
     0.09s  0.11%  0.26%     50.77s 62.58%                | github.com/pingcap/tidb/server.(*clientConn).dispatch
                                            50.13s 98.74% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
----------------------------------------------------------+-------------
                                            50.13s   100% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
     0.08s 0.099%  0.36%     50.13s 61.79%                | github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
                                            37.26s 74.33% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                            12.42s 24.78% |   github.com/pingcap/tidb/server.(*TiDBStatement).Execute
----------------------------------------------------------+-------------
                                            37.26s   100% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
     0.06s 0.074%  0.43%     37.26s 45.93%                | github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                            19.36s 51.96% |   github.com/pingcap/tidb/server.(*clientConn).writeChunks
                                            15.41s 41.36% |   github.com/pingcap/tidb/server.(*clientConn).flush
----------------------------------------------------------+-------------
                                            17.56s 59.49% |   syscall.write
                                            11.96s 40.51% |   syscall.Read
    29.06s 35.82% 36.25%     29.52s 36.39%                | syscall.Syscall
----------------------------------------------------------+-------------
                                            19.36s   100% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
     0.25s  0.31% 36.56%     19.36s 23.86%                | github.com/pingcap/tidb/server.(*clientConn).writeChunks
                                            13.52s 69.83% |   github.com/pingcap/tidb/server.(*tidbResultSet).Next
----------------------------------------------------------+-------------
                                            15.23s 84.14% |   bufio.(*Writer).Flush
     0.01s 0.012% 36.57%     18.10s 22.31%                | net.(*conn).Write
                                            18.09s 99.94% |   net.(*netFD).Write
----------------------------------------------------------+-------------
                                            18.09s   100% |   net.(*conn).Write
     0.19s  0.23% 36.81%     18.09s 22.30%                | net.(*netFD).Write
                                            17.90s 98.95% |   internal/poll.(*FD).Write
----------------------------------------------------------+-------------
                                            17.90s   100% |   net.(*netFD).Write
         0     0% 36.81%     17.90s 22.06%                | internal/poll.(*FD).Write
                                            17.60s 98.32% |   syscall.Write
----------------------------------------------------------+-------------
                                            17.60s   100% |   internal/poll.(*FD).Write
     0.02s 0.025% 36.83%     17.60s 21.69%                | syscall.Write
                                            17.58s 99.89% |   syscall.write
----------------------------------------------------------+-------------
                                            17.58s   100% |   syscall.Write
     0.02s 0.025% 36.85%     17.58s 21.67%                | syscall.write
                                            17.56s 99.89% |   syscall.Syscall
----------------------------------------------------------+-------------
                                            15.38s 99.61% |   github.com/pingcap/tidb/server.(*packetIO).flush
     0.15s  0.18% 37.04%     15.44s 19.03%                | bufio.(*Writer).Flush
                                            15.23s 98.64% |   net.(*conn).Write
----------------------------------------------------------+-------------
                                            15.41s   100% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
     0.03s 0.037% 37.08%     15.41s 18.99%                | github.com/pingcap/tidb/server.(*clientConn).flush
                                            15.38s 99.81% |   github.com/pingcap/tidb/server.(*packetIO).flush
----------------------------------------------------------+-------------
                                            15.38s   100% |   github.com/pingcap/tidb/server.(*clientConn).flush
         0     0% 37.08%     15.38s 18.96%                | github.com/pingcap/tidb/server.(*packetIO).flush
                                            15.38s   100% |   bufio.(*Writer).Flush
----------------------------------------------------------+-------------
                                            13.52s   100% |   github.com/pingcap/tidb/server.(*clientConn).writeChunks
     0.04s 0.049% 37.13%     13.52s 16.66%                | github.com/pingcap/tidb/server.(*tidbResultSet).Next
                                            13.48s 99.70% |   github.com/pingcap/tidb/executor.(*recordSet).Next
----------------------------------------------------------+-------------
                                            13.48s 99.85% |   github.com/pingcap/tidb/server.(*tidbResultSet).Next
     0.43s  0.53% 37.66%     13.50s 16.64%                | github.com/pingcap/tidb/executor.(*recordSet).Next
                                            12.68s 93.93% |   github.com/pingcap/tidb/executor.Next
----------------------------------------------------------+-------------
                                            11.44s 87.00% |   github.com/pingcap/tidb/server.(*packetIO).readPacket
     0.01s 0.012% 37.67%     13.15s 16.21%                | io.ReadFull
                                            13.14s 99.92% |   io.ReadAtLeast
----------------------------------------------------------+-------------
                                            13.14s   100% |   io.ReadFull
     0.07s 0.086% 37.75%     13.14s 16.20%                | io.ReadAtLeast
                                            12.82s 97.56% |   bufio.(*Reader).Read
----------------------------------------------------------+-------------
                                            12.82s   100% |   io.ReadAtLeast
     0.17s  0.21% 37.96%     12.82s 15.80%                | bufio.(*Reader).Read
                                            12.56s 97.97% |   net.(*conn).Read
----------------------------------------------------------+-------------
                                            12.68s 99.92% |   github.com/pingcap/tidb/executor.(*recordSet).Next
     0.24s   0.3% 38.26%     12.69s 15.64%                | github.com/pingcap/tidb/executor.Next
                                            12.20s 96.14% |   github.com/pingcap/tidb/executor.(*PointGetExecutor).Next
----------------------------------------------------------+-------------
                                            12.56s   100% |   bufio.(*Reader).Read
     0.03s 0.037% 38.30%     12.56s 15.48%                | net.(*conn).Read
                                            12.53s 99.76% |   net.(*netFD).Read
----------------------------------------------------------+-------------
                                            12.53s   100% |   net.(*conn).Read
     0.01s 0.012% 38.31%     12.53s 15.44%                | net.(*netFD).Read
                                            12.52s 99.92% |   internal/poll.(*FD).Read
----------------------------------------------------------+-------------
                                            12.52s   100% |   net.(*netFD).Read
     0.06s 0.074% 38.38%     12.52s 15.43%                | internal/poll.(*FD).Read
                                            12.09s 96.57% |   syscall.Read
----------------------------------------------------------+-------------
                                            12.42s   100% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
     0.02s 0.025% 38.41%     12.42s 15.31%                | github.com/pingcap/tidb/server.(*TiDBStatement).Execute
                                            12.32s 99.19% |   github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
----------------------------------------------------------+-------------
                                            12.32s   100% |   github.com/pingcap/tidb/server.(*TiDBStatement).Execute
     0.04s 0.049% 38.46%     12.32s 15.19%                | github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
----------------------------------------------------------+-------------
                                            12.20s   100% |   github.com/pingcap/tidb/executor.Next
     0.26s  0.32% 38.78%     12.20s 15.04%                | github.com/pingcap/tidb/executor.(*PointGetExecutor).Next
----------------------------------------------------------+-------------
                                            12.09s   100% |   internal/poll.(*FD).Read
     0.07s 0.086% 38.86%     12.09s 14.90%                | syscall.Read
                                            11.96s 98.92% |   syscall.Syscall
----------------------------------------------------------+-------------
                                            12.05s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
         0     0% 38.86%     12.05s 14.85%                | github.com/pingcap/tidb/server.(*clientConn).readPacket
                                            12.05s   100% |   github.com/pingcap/tidb/server.(*packetIO).readPacket
----------------------------------------------------------+-------------
                                            12.05s   100% |   github.com/pingcap/tidb/server.(*clientConn).readPacket
     0.01s 0.012% 38.88%     12.05s 14.85%                | github.com/pingcap/tidb/server.(*packetIO).readPacket
                                            11.44s 94.94% |   io.ReadFull
----------------------------------------------------------+-------------
```

```bash
go tool pprof -nodecount 20 -flat -text profile

Duration: 2mins, Total samples = 81.13s (67.53%)
Showing nodes accounting for 45.60s, 56.21% of 81.13s total
Dropped 803 nodes (cum <= 0.41s)
Showing top 20 nodes out of 218
      flat  flat%   sum%        cum   cum%
    29.06s 35.82% 35.82%     29.52s 36.39%  syscall.Syscall
     2.29s  2.82% 38.64%      2.29s  2.82%  runtime.memmove
     1.81s  2.23% 40.87%      3.27s  4.03%  runtime.scanobject
     1.70s  2.10% 42.97%      8.96s 11.04%  runtime.mallocgc
     1.11s  1.37% 44.34%      1.11s  1.37%  runtime.memclrNoHeapPointers
     1.05s  1.29% 45.63%      1.35s  1.66%  runtime.heapBitsSetType
     0.94s  1.16% 46.79%      0.94s  1.16%  runtime.nextFreeFast
     0.86s  1.06% 47.85%      6.62s  8.16%  runtime.newobject
     0.80s  0.99% 48.84%      0.90s  1.11%  runtime.findObject
     0.79s  0.97% 49.81%      0.85s  1.05%  time.now
     0.76s  0.94% 50.75%      2.26s  2.79%  runtime.selectgo
     0.68s  0.84% 51.58%      0.68s  0.84%  runtime.lock
     0.56s  0.69% 52.27%      0.58s  0.71%  runtime.(*itabTableType).find
     0.49s   0.6% 52.88%      0.51s  0.63%  context.(*valueCtx).Value
     0.48s  0.59% 53.47%      0.86s  1.06%  runtime.mapaccess2
     0.46s  0.57% 54.04%      2.37s  2.92%  github.com/pingcap/tidb/server.(*ColumnInfo).Dump
     0.46s  0.57% 54.60%      1.04s  1.28%  runtime.getitab
     0.45s  0.55% 55.16%      0.97s  1.20%  runtime.schedule
     0.43s  0.53% 55.69%     13.50s 16.64%  github.com/pingcap/tidb/executor.(*recordSet).Next
     0.42s  0.52% 56.21%      0.53s  0.65%  runtime.deferreturn
```

从profile中可以得到以下信息：

1. TiDB的CPU使用率较高
2. syscall.Syscall消耗了大量的CPU时间
3. syscall.Syscall主要由server.(*clientConn).writeResultset、server.(*packetIO).readPacket引起的
4. GC开销很小

##### 3.3.2.4内存

```bash
go tool pprof -nodecount 20 -flat -text -alloc_space heap

Showing nodes accounting for 20.59GB, 54.99% of 37.44GB total
Dropped 1312 nodes (cum <= 0.19GB)
Showing top 20 nodes out of 285
      flat  flat%   sum%        cum   cum%
    3.54GB  9.45%  9.45%     3.54GB  9.45%  github.com/pingcap/tidb/statistics/handle.statsCache.copy
    2.34GB  6.26% 15.71%     2.34GB  6.26%  github.com/pingcap/tidb/util/chunk.newFixedLenColumn
    1.98GB  5.30% 21.01%     1.98GB  5.30%  github.com/pingcap/tidb/util/chunk.newVarLenColumn
    1.72GB  4.58% 25.59%     1.72GB  4.58%  github.com/prometheus/client_golang/prometheus.(*histogram).Write
    1.55GB  4.15% 29.74%     1.56GB  4.16%  fmt.Sprintf
    1.51GB  4.04% 33.78%     8.30GB 22.17%  github.com/pingcap/tidb/statistics/handle.(*Handle).HandleAutoAnalyze
    1.16GB  3.11% 36.89%     1.20GB  3.21%  github.com/pingcap/tidb/executor.(*indexWorker).extractTaskHandles
    1.11GB  2.95% 39.84%     1.11GB  2.95%  github.com/pingcap/tidb/infoschema.(*infoSchema).SchemaTables
    0.65GB  1.73% 41.57%     0.65GB  1.73%  github.com/pingcap/tidb/executor.colNames2ResultFields
    0.57GB  1.52% 43.09%     0.64GB  1.72%  github.com/pingcap/parser.yyParse
    0.50GB  1.33% 44.43%     0.50GB  1.34%  google.golang.org/grpc.(*parser).recvMsg
    0.49GB  1.31% 45.74%     0.49GB  1.31%  github.com/pingcap/tidb/expression.(*Column).Clone
    0.49GB  1.30% 47.04%     0.49GB  1.30%  github.com/pingcap/tidb/kv/memdb.newArenaBlock
    0.46GB  1.24% 48.28%     0.81GB  2.15%  github.com/pingcap/tidb/planner/core.(*PlanBuilder).buildDataSource
    0.46GB  1.24% 49.52%     0.65GB  1.73%  github.com/pingcap/tidb/executor.ResetContextOfStmt
    0.45GB  1.20% 50.71%     0.51GB  1.35%  google.golang.org/grpc/internal/transport.(*http2Client).Write
    0.42GB  1.13% 51.85%     0.43GB  1.14%  github.com/pingcap/tidb/util/chunk.(*Column).AppendBytes
    0.41GB  1.10% 52.95%     0.41GB  1.10%  time.NewTimer
    0.40GB  1.07% 54.02%     2.41GB  6.43%  github.com/prometheus/client_golang/prometheus.processMetric
    0.36GB  0.97% 54.99%     0.73GB  1.94%  github.com/pingcap/tidb/statistics.PseudoTable
```

```bash
go tool pprof -nodecount 20 -cum -text -alloc_space heap

Showing nodes accounting for 1679.58MB, 4.38% of 38336.78MB total
Dropped 1312 nodes (cum <= 191.68MB)
Showing top 20 nodes out of 285
      flat  flat%   sum%        cum   cum%
         0     0%     0%  8498.97MB 22.17%  github.com/pingcap/tidb/domain.(*Domain).autoAnalyzeWorker
 1549.07MB  4.04%  4.04%  8498.97MB 22.17%  github.com/pingcap/tidb/statistics/handle.(*Handle).HandleAutoAnalyze
    3.50MB 0.0091%  4.05%  7142.33MB 18.63%  github.com/pingcap/tidb/session.execRestrictedSQL
         0     0%  4.05%  7140.33MB 18.63%  github.com/pingcap/tidb/session.(*session).ExecRestrictedSQL
         0     0%  4.05%  7140.33MB 18.63%  github.com/pingcap/tidb/session.(*session).ExecRestrictedSQLWithContext
         0     0%  4.05%  7050.64MB 18.39%  github.com/pingcap/tidb/session.(*session).execute
         0     0%  4.05%  7046.64MB 18.38%  github.com/pingcap/tidb/session.(*session).Execute
         0     0%  4.05%  6448.49MB 16.82%  github.com/pingcap/tidb/server.(*Server).onConn
         0     0%  4.05%  6435.89MB 16.79%  github.com/pingcap/tidb/server.(*clientConn).Run
         0     0%  4.05%  6407.89MB 16.71%  github.com/pingcap/tidb/server.(*clientConn).dispatch
      38MB 0.099%  4.15%  6219.97MB 16.22%  github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
         0     0%  4.15%  6154.93MB 16.05%  github.com/pingcap/tidb/executor.Next
         0     0%  4.15%  5701.55MB 14.87%  github.com/pingcap/tidb/domain.(*Domain).loadStatsWorker
       1MB 0.0026%  4.15%  5487.45MB 14.31%  github.com/pingcap/tidb/statistics/handle.(*Handle).Update
       7MB 0.018%  4.17%  5476.38MB 14.28%  github.com/pingcap/tidb/session.runStmt
      17MB 0.044%  4.21%  5118.88MB 13.35%  github.com/pingcap/tidb/server.(*TiDBStatement).Execute
         0     0%  4.21%  5103.37MB 13.31%  github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
         0     0%  4.21%  4592.25MB 11.98%  github.com/pingcap/tidb/planner.Optimize
      27MB  0.07%  4.28%  4526.77MB 11.81%  github.com/pingcap/tidb/executor.(*ExecStmt).Exec
      37MB 0.097%  4.38%  4465.19MB 11.65%  github.com/pingcap/tidb/executor.(*Compiler).Compile
```

```bash
go tool pprof -nodecount 20 -flat -text -inuse_space heap

Showing nodes accounting for 30595.18kB, 82.13% of 37253.83kB total
Showing top 20 nodes out of 230
      flat  flat%   sum%        cum   cum%
 6866.17kB 18.43% 18.43%  6866.17kB 18.43%  github.com/pingcap/tidb/util/arena.NewAllocator
 5200.42kB 13.96% 32.39%  5200.42kB 13.96%  bufio.NewReaderSize
 3632.45kB  9.75% 42.14%  3632.45kB  9.75%  github.com/pingcap/parser.New
 2600.21kB  6.98% 49.12%  2600.21kB  6.98%  bufio.NewWriterSize
 1536.16kB  4.12% 53.24%  1536.16kB  4.12%  github.com/pingcap/parser.yyParse
 1024.31kB  2.75% 55.99%  1024.31kB  2.75%  github.com/pingcap/tidb/planner/core.getIndexValues
 1024.22kB  2.75% 58.74%  1024.22kB  2.75%  fmt.Sprintf
 1024.15kB  2.75% 61.49%  2048.26kB  5.50%  github.com/pingcap/tidb/statistics.PseudoTable
 1024.11kB  2.75% 64.24%  1024.11kB  2.75%  github.com/pingcap/tidb/util/chunk.newFixedLenColumn
 1024.05kB  2.75% 66.99%  1024.05kB  2.75%  container/list.(*List).insertValue
  902.59kB  2.42% 69.41%  1447.25kB  3.88%  compress/flate.NewWriter
  544.67kB  1.46% 70.87%   544.67kB  1.46%  compress/flate.(*compressor).initDeflate
  544.67kB  1.46% 72.34%   544.67kB  1.46%  google.golang.org/grpc/internal/transport.newBufWriter
  536.37kB  1.44% 73.78%   536.37kB  1.44%  bytes.makeSlice
  528.17kB  1.42% 75.19%   528.17kB  1.42%  github.com/pingcap/tidb/types.BinaryLiteral.ToString
  524.09kB  1.41% 76.60%   524.09kB  1.41%  golang.org/x/net/http2.(*Framer).WriteDataPadded
  516.01kB  1.39% 77.99%   516.01kB  1.39%  github.com/pingcap/tidb/statistics.NewCMSketch
  514.38kB  1.38% 79.37%   514.38kB  1.38%  github.com/pingcap/tidb/sessionctx/variable.(*SessionVars).SetSystemVar
     514kB  1.38% 80.75%      514kB  1.38%  github.com/pingcap/tidb/statistics.(*Histogram).AppendBucket
     514kB  1.38% 82.13%      514kB  1.38%  sync.(*Map).Store
```

从profile中可以得到以下信息：

1. handle.(*Handle).HandleAutoAnalyze引起了大量内存申请的需求

##### 3.3.2.5 IO

![Workload c disk performance](./profiles/ycsb/tidb/workloadc/c_disk_perf.png)

![Workload c tikv summart](./profiles/ycsb/tidb/workloadc/c_tikv_summary.png)

![Workload c tikv network](./profiles/ycsb/tidb/workloadc/c_tikv_net.png)

根据以上监控数据可以得出：

1. TiKV磁盘IOPS与磁盘带宽显著提高
2. TiKV的网络吞吐引起了程序瓶颈
3. TiKV的CPU负载较高

##### 3.3.2.6 结论

- Workload c是100%的`Read`操作的负载
- TiDB查询QPS相较于workload a大幅提升
- TiKV节点网络带宽降低，网络IOPS下降，但原因不明
- handle.statsCache.copy引起了大量的内存申请需求

---

#### 3.3.3 Workload e

##### 3.3.3.1 负载配置

```bash
"updateproportion"="0"
"workload"="core"
"requestdistribution"="uniform"
"maxscanlength"="1"
"scanproportion"="0.95"
"operationcount"="1000000"
"threadcount"="256"
"readallfields"="true"
"scanlengthdistribution"="uniform"
"dotransactions"="true"
"readproportion"="0"
"recordcount"="1000"
"mysql.host"="192.168.99.101"
"mysql.port"="4000"
"insertproportion"="0.05"
```

##### 3.3.3.2 压测结果

```bash
INSERT - Takes(s): 1201.7, Count: 49917, OPS: 41.5, Avg(us): 214754, Min(us): 2764, Max(us): 20166731, 99th(us): 389000, 99.9th(us): 8160000, 99.99th(us): 20053000
SCAN   - Takes(s): 1201.7, Count: 950019, OPS: 790.6, Avg(us): 307257, Min(us): 4079, Max(us): 21248821, 99th(us): 523000, 99.9th(us): 10371000, 99.99th(us): 20184000
```

##### 3.3.3.3 CPU

```bash
go tool pprof -nodecount 20 -cum -tree profile

Duration: 2.01mins, Total samples = 96.74s (80.39%)
Showing nodes accounting for 5.80s, 6.00% of 96.74s total
Dropped 1485 nodes (cum <= 0.48s)
Showing top 20 nodes out of 316
----------------------------------------------------------+-------------
      flat  flat%   sum%        cum   cum%   calls calls% + context 	 	 
----------------------------------------------------------+-------------
         0     0%     0%     51.92s 53.67%                | github.com/pingcap/tidb/server.(*Server).onConn
                                            51.92s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
----------------------------------------------------------+-------------
                                            51.92s   100% |   github.com/pingcap/tidb/server.(*Server).onConn
     0.15s  0.16%  0.16%     51.92s 53.67%                | github.com/pingcap/tidb/server.(*clientConn).Run
                                            48.09s 92.62% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
                                             0.25s  0.48% |   runtime.mallocgc
----------------------------------------------------------+-------------
                                            48.09s   100% |   github.com/pingcap/tidb/server.(*clientConn).Run
     0.07s 0.072%  0.23%     48.09s 49.71%                | github.com/pingcap/tidb/server.(*clientConn).dispatch
                                            47.01s 97.75% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
                                             0.10s  0.21% |   runtime.newobject
----------------------------------------------------------+-------------
                                            47.01s   100% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
     0.12s  0.12%  0.35%     47.01s 48.59%                | github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
                                            23.79s 50.61% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                            22.54s 47.95% |   github.com/pingcap/tidb/server.(*TiDBStatement).Execute
----------------------------------------------------------+-------------
                                            23.79s   100% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
     0.01s  0.01%  0.36%     23.79s 24.59%                | github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                            14.12s 59.35% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset.func1
                                             0.67s  2.82% |   runtime.newobject
                                             0.59s  2.48% |   runtime.systemstack
                                             0.57s  2.40% |   runtime.mallocgc
----------------------------------------------------------+-------------
                                            15.77s 67.71% |   runtime.newobject
                                             1.45s  6.23% |   github.com/pingcap/tidb/planner.optimize
                                             1.27s  5.45% |   github.com/pingcap/tidb/executor.(*recordSet).Close
                                             0.60s  2.58% |   github.com/pingcap/tidb/session.(*session).CommonExec
                                             0.57s  2.45% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                             0.25s  1.07% |   github.com/pingcap/tidb/server.(*clientConn).Run
                                             0.19s  0.82% |   github.com/pingcap/tidb/planner.Optimize
     3.12s  3.23%  3.59%     23.29s 24.07%                | runtime.mallocgc
                                             7.33s 31.47% |   runtime.systemstack
----------------------------------------------------------+-------------
                                            22.54s   100% |   github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
         0     0%  3.59%     22.54s 23.30%                | github.com/pingcap/tidb/server.(*TiDBStatement).Execute
                                            22.49s 99.78% |   github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
----------------------------------------------------------+-------------
                                            22.49s   100% |   github.com/pingcap/tidb/server.(*TiDBStatement).Execute
     0.01s  0.01%  3.60%     22.49s 23.25%                | github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
                                            22.16s 98.53% |   github.com/pingcap/tidb/session.(*session).CommonExec
----------------------------------------------------------+-------------
                                            22.16s   100% |   github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
     0.03s 0.031%  3.63%     22.16s 22.91%                | github.com/pingcap/tidb/session.(*session).CommonExec
                                            15.96s 72.02% |   github.com/pingcap/tidb/executor.CompileExecutePreparedStmt
                                             1.76s  7.94% |   runtime.newobject
                                             0.60s  2.71% |   runtime.mallocgc
                                             0.10s  0.45% |   runtime.systemstack
----------------------------------------------------------+-------------
                                             2.51s 14.29% |   github.com/pingcap/tidb/planner.optimize
                                             1.88s 10.70% |   github.com/pingcap/tidb/executor.(*recordSet).Close
                                             1.76s 10.02% |   github.com/pingcap/tidb/session.(*session).CommonExec
                                             0.67s  3.81% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                             0.33s  1.88% |   github.com/pingcap/tidb/planner.Optimize
                                             0.26s  1.48% |   github.com/pingcap/tidb/executor.CompileExecutePreparedStmt
                                             0.10s  0.57% |   github.com/pingcap/tidb/server.(*clientConn).dispatch
     1.80s  1.86%  5.49%     17.57s 18.16%                | runtime.newobject
                                            15.77s 89.76% |   runtime.mallocgc
----------------------------------------------------------+-------------
                                             7.33s 42.32% |   runtime.mallocgc
                                             0.59s  3.41% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
                                             0.40s  2.31% |   github.com/pingcap/tidb/planner.optimize
                                             0.28s  1.62% |   github.com/pingcap/tidb/executor.(*recordSet).Close
                                             0.10s  0.58% |   github.com/pingcap/tidb/session.(*session).CommonExec
     0.10s   0.1%  5.59%     17.32s 17.90%                | runtime.systemstack
----------------------------------------------------------+-------------
                                            15.96s   100% |   github.com/pingcap/tidb/session.(*session).CommonExec
         0     0%  5.59%     15.96s 16.50%                | github.com/pingcap/tidb/executor.CompileExecutePreparedStmt
                                            15.41s 96.55% |   github.com/pingcap/tidb/planner.Optimize
                                             0.26s  1.63% |   runtime.newobject
----------------------------------------------------------+-------------
                                            15.41s 99.55% |   github.com/pingcap/tidb/executor.CompileExecutePreparedStmt
                                            14.54s 93.93% |   github.com/pingcap/tidb/planner/core.(*Execute).getPhysicalPlan
     0.01s  0.01%  5.60%     15.48s 16.00%                | github.com/pingcap/tidb/planner.Optimize
                                            15.14s 97.80% |   github.com/pingcap/tidb/planner.optimize
                                             0.33s  2.13% |   runtime.newobject
                                             0.19s  1.23% |   runtime.mallocgc
----------------------------------------------------------+-------------
                                            15.14s   100% |   github.com/pingcap/tidb/planner.Optimize
     0.01s  0.01%  5.61%     15.14s 15.65%                | github.com/pingcap/tidb/planner.optimize
                                            14.77s 97.56% |   github.com/pingcap/tidb/planner/core.(*Execute).OptimizePreparedPlan
                                             2.51s 16.58% |   runtime.newobject
                                             1.45s  9.58% |   runtime.mallocgc
                                             0.40s  2.64% |   runtime.systemstack
----------------------------------------------------------+-------------
                                            14.77s   100% |   github.com/pingcap/tidb/planner.optimize
     0.06s 0.062%  5.68%     14.77s 15.27%                | github.com/pingcap/tidb/planner/core.(*Execute).OptimizePreparedPlan
                                            14.70s 99.53% |   github.com/pingcap/tidb/planner/core.(*Execute).getPhysicalPlan
----------------------------------------------------------+-------------
                                            14.70s   100% |   github.com/pingcap/tidb/planner/core.(*Execute).OptimizePreparedPlan
     0.03s 0.031%  5.71%     14.70s 15.20%                | github.com/pingcap/tidb/planner/core.(*Execute).getPhysicalPlan
                                            14.54s 98.91% |   github.com/pingcap/tidb/planner.Optimize
----------------------------------------------------------+-------------
                                            14.06s 96.24% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset.func1
     0.13s  0.13%  5.84%     14.61s 15.10%                | github.com/pingcap/parser/terror.Call
                                            13.98s 95.69% |   github.com/pingcap/tidb/server.(*tidbResultSet).Close
----------------------------------------------------------+-------------
                                            14.12s   100% |   github.com/pingcap/tidb/server.(*clientConn).writeResultset
     0.05s 0.052%  5.89%     14.12s 14.60%                | github.com/pingcap/tidb/server.(*clientConn).writeResultset.func1
                                            14.06s 99.58% |   github.com/pingcap/parser/terror.Call
----------------------------------------------------------+-------------
                                            13.98s   100% |   github.com/pingcap/parser/terror.Call
     0.06s 0.062%  5.95%     13.98s 14.45%                | github.com/pingcap/tidb/server.(*tidbResultSet).Close
                                            13.92s 99.57% |   github.com/pingcap/tidb/executor.(*recordSet).Close
----------------------------------------------------------+-------------
                                            13.92s   100% |   github.com/pingcap/tidb/server.(*tidbResultSet).Close
     0.04s 0.041%  6.00%     13.92s 14.39%                | github.com/pingcap/tidb/executor.(*recordSet).Close
                                             1.88s 13.51% |   runtime.newobject
                                             1.27s  9.12% |   runtime.mallocgc
                                             0.28s  2.01% |   runtime.systemstack
----------------------------------------------------------+-------------
```

```bash
go tool pprof -nodecount 20 -flat -text profile

Duration: 2.01mins, Total samples = 96.74s (80.39%)
Showing nodes accounting for 41.44s, 42.84% of 96.74s total
Dropped 1485 nodes (cum <= 0.48s)
Showing top 20 nodes out of 316
      flat  flat%   sum%        cum   cum%
     8.03s  8.30%  8.30%      8.44s  8.72%  syscall.Syscall
     6.30s  6.51% 14.81%     11.64s 12.03%  runtime.scanobject
     3.73s  3.86% 18.67%      4.14s  4.28%  runtime.findObject
     3.12s  3.23% 21.89%     23.29s 24.07%  runtime.mallocgc
     2.30s  2.38% 24.27%      5.70s  5.89%  runtime.heapBitsSetType
     2.19s  2.26% 26.54%      2.19s  2.26%  crypto/sha256.block
     2.18s  2.25% 28.79%      2.18s  2.25%  runtime.memclrNoHeapPointers
     1.97s  2.04% 30.82%      1.97s  2.04%  runtime.memmove
     1.80s  1.86% 32.69%     17.57s 18.16%  runtime.newobject
     1.73s  1.79% 34.47%      2.04s  2.11%  runtime.step
     1.57s  1.62% 36.10%      1.57s  1.62%  runtime.nextFreeFast
     1.03s  1.06% 37.16%      1.03s  1.06%  runtime.lock
     0.99s  1.02% 38.18%      3.13s  3.24%  runtime.pcvalue
     0.87s   0.9% 39.08%      0.87s   0.9%  runtime.markBits.isMarked
     0.73s  0.75% 39.84%      2.80s  2.89%  runtime.selectgo
     0.69s  0.71% 40.55%      0.77s   0.8%  runtime.findfunc
     0.61s  0.63% 41.18%      5.87s  6.07%  runtime.gentraceback
     0.59s  0.61% 41.79%      0.62s  0.64%  time.now
     0.51s  0.53% 42.32%      0.52s  0.54%  runtime.(*itabTableType).find
     0.50s  0.52% 42.84%      0.88s  0.91%  runtime.mapaccess2_faststr
```

从profile中可以得到以下信息：

1. TiDB的CPU使用率较高
2. syscall.Syscall开销较高，通常代表IO
3. runtime.scanobject、runtime.findObject等GC相关函数开销较高
4. planner.Optimize和server.(*clientConn).writeResultset消耗了大量的CPU时间

##### 3.3.3.4 内存

```bash
go tool pprof -nodecount 20 -flat -text -alloc_space heap

Showing nodes accounting for 70.64GB, 51.78% of 136.44GB total
Dropped 1444 nodes (cum <= 0.68GB)
Showing top 20 nodes out of 260
      flat  flat%   sum%        cum   cum%
   19.86GB 14.56% 14.56%    19.86GB 14.56%  github.com/pingcap/tidb/util/chunk.newVarLenColumn
   10.95GB  8.02% 22.58%    11.25GB  8.24%  github.com/pingcap/tidb/executor.(*indexWorker).extractTaskHandles
    7.55GB  5.53% 28.12%     7.55GB  5.53%  github.com/pingcap/tidb/util/chunk.newFixedLenColumn
    3.71GB  2.72% 30.83%     5.82GB  4.26%  github.com/pingcap/tidb/planner/core.(*PlanBuilder).buildDataSource
    3.61GB  2.65% 33.48%     3.61GB  2.65%  github.com/pingcap/tidb/statistics/handle.statsCache.copy
    3.59GB  2.63% 36.11%     3.59GB  2.63%  github.com/pingcap/tidb/expression.(*Column).Clone
    2.32GB  1.70% 37.81%     2.33GB  1.71%  google.golang.org/grpc.(*parser).recvMsg
    1.92GB  1.41% 39.22%     1.92GB  1.41%  github.com/prometheus/client_golang/prometheus.(*histogram).Write
    1.89GB  1.38% 40.60%     1.89GB  1.39%  fmt.Sprintf
    1.73GB  1.27% 41.87%     2.40GB  1.76%  github.com/pingcap/tidb/executor.ResetContextOfStmt
    1.69GB  1.24% 43.11%     8.82GB  6.47%  github.com/pingcap/tidb/statistics/handle.(*Handle).HandleAutoAnalyze
    1.42GB  1.04% 44.15%     1.43GB  1.05%  github.com/pingcap/tidb/util/chunk.(*Column).AppendBytes
    1.35GB  0.99% 45.15%     1.35GB  0.99%  github.com/pingcap/tidb/planner/core.NewPlanBuilder
    1.35GB  0.99% 46.13%     1.53GB  1.12%  google.golang.org/grpc/internal/transport.(*http2Client).Write
    1.34GB  0.98% 47.11%     1.34GB  0.98%  github.com/pingcap/kvproto/pkg/kvrpcpb.(*GetResponse).Unmarshal
    1.33GB  0.98% 48.09%     1.33GB  0.98%  github.com/pingcap/tidb/statistics.(*HistColl).GenerateHistCollFromColumnInfo
    1.32GB  0.96% 49.05%     1.32GB  0.96%  github.com/pingcap/tidb/util/memory.NewTracker
    1.29GB  0.95% 50.00%     2.40GB  1.76%  github.com/pingcap/tidb/planner/core.buildSchemaFromFields
    1.21GB  0.89% 50.89%     1.21GB  0.89%  github.com/pingcap/tidb/infoschema.(*infoSchema).SchemaTables
    1.21GB  0.89% 51.78%     1.21GB  0.89%  github.com/pingcap/tidb/planner/property.(*StatsInfo).Scale
```

```bash
go tool pprof -nodecount 20 -cum -text -alloc_space heap

Showing nodes accounting for 21.88GB, 16.03% of 136.44GB total
Dropped 1444 nodes (cum <= 0.68GB)
Showing top 20 nodes out of 260
      flat  flat%   sum%        cum   cum%
         0     0%     0%    63.54GB 46.57%  github.com/pingcap/tidb/server.(*Server).onConn
         0     0%     0%    63.51GB 46.55%  github.com/pingcap/tidb/server.(*clientConn).Run
         0     0%     0%    63.38GB 46.45%  github.com/pingcap/tidb/server.(*clientConn).dispatch
    0.22GB  0.16%  0.16%    62.92GB 46.12%  github.com/pingcap/tidb/server.(*clientConn).handleStmtExecute
    0.14GB   0.1%  0.26%    47.12GB 34.53%  github.com/pingcap/tidb/server.(*TiDBStatement).Execute
         0     0%  0.26%    46.98GB 34.43%  github.com/pingcap/tidb/session.(*session).ExecutePreparedStmt
         0     0%  0.26%    42.79GB 31.36%  github.com/pingcap/tidb/session.(*session).CommonExec
         0     0%  0.26%    32.26GB 23.65%  github.com/pingcap/tidb/planner.Optimize
    0.13GB 0.097%  0.36%    31.67GB 23.21%  github.com/pingcap/tidb/planner.optimize
    0.39GB  0.28%  0.64%    29.82GB 21.86%  github.com/pingcap/tidb/executor.CompileExecutePreparedStmt
    1.10GB  0.81%  1.45%    28.19GB 20.66%  github.com/pingcap/tidb/util/chunk.New
         0     0%  1.45%    27.41GB 20.09%  github.com/pingcap/tidb/util/chunk.newColumn
         0     0%  1.45%    27.09GB 19.86%  github.com/pingcap/tidb/util/chunk.NewColumn
         0     0%  1.45%    26.16GB 19.18%  github.com/pingcap/tidb/planner/core.(*Execute).OptimizePreparedPlan
         0     0%  1.45%    26.16GB 19.18%  github.com/pingcap/tidb/planner/core.(*Execute).getPhysicalPlan
         0     0%  1.45%    21.92GB 16.07%  github.com/pingcap/tidb/executor.newFirstChunk
   19.86GB 14.56% 16.01%    19.86GB 14.56%  github.com/pingcap/tidb/util/chunk.newVarLenColumn
    0.03GB 0.022% 16.03%    17.83GB 13.06%  github.com/pingcap/tidb/session.runStmt
         0     0% 16.03%    17.31GB 12.69%  github.com/pingcap/tidb/executor.(*IndexLookUpExecutor).startIndexWorker.func1
         0     0% 16.03%    17.25GB 12.65%  github.com/pingcap/tidb/executor.(*indexWorker).fetchHandles
```

```bash
go tool pprof -nodecount 20 -flat -text -inuse_space heap

Showing nodes accounting for 40860.97kB, 79.15% of 51623.02kB total
Showing top 20 nodes out of 310
      flat  flat%   sum%        cum   cum%
 6866.17kB 13.30% 13.30%  6866.17kB 13.30%  github.com/pingcap/tidb/util/arena.NewAllocator
 5720.46kB 11.08% 24.38%  5720.46kB 11.08%  bufio.NewWriterSize
 5720.46kB 11.08% 35.46%  5720.46kB 11.08%  github.com/pingcap/parser.New
 4680.37kB  9.07% 44.53%  4680.37kB  9.07%  bufio.NewReaderSize
 2048.36kB  3.97% 48.50%  2048.36kB  3.97%  github.com/pingcap/tidb/util/chunk.newVarLenColumn
 2048.25kB  3.97% 52.47%  3072.47kB  5.95%  github.com/pingcap/parser.yyParse
 1540.12kB  2.98% 55.45%  1540.12kB  2.98%  github.com/pingcap/tidb/util/chunk.newFixedLenColumn
 1536.42kB  2.98% 58.42%  1536.42kB  2.98%  github.com/pingcap/tidb/statistics.(*HistColl).GenerateHistCollFromColumnInfo
 1536.30kB  2.98% 61.40%  1536.30kB  2.98%  github.com/pingcap/tidb/planner/core.deriveLimitStats
 1025.12kB  1.99% 63.39%  1025.12kB  1.99%  github.com/pingcap/tidb/server.(*packetIO).readOnePacket
 1024.22kB  1.98% 65.37%  1024.22kB  1.98%  github.com/pingcap/tidb/planner/core.PhysicalIndexLookUpReader.Init
 1024.22kB  1.98% 67.35%  1024.22kB  1.98%  github.com/pingcap/tidb/types/parser_driver.newParamMarkerExpr
 1024.16kB  1.98% 69.34%  1024.16kB  1.98%  github.com/pingcap/tidb/expression.ColumnInfos2ColumnsAndNames
 1024.15kB  1.98% 71.32%  2048.26kB  3.97%  github.com/pingcap/tidb/statistics.PseudoTable
 1024.05kB  1.98% 73.31%  1024.05kB  1.98%  container/list.(*List).insertValue
  902.59kB  1.75% 75.05%   902.59kB  1.75%  compress/flate.NewWriter
  544.67kB  1.06% 76.11%   544.67kB  1.06%  google.golang.org/grpc/internal/transport.newBufWriter
  528.17kB  1.02% 77.13%   528.17kB  1.02%  github.com/pingcap/tidb/types.BinaryLiteral.ToString
  522.70kB  1.01% 78.15%   522.70kB  1.01%  golang.org/x/net/http2.(*Framer).WriteDataPadded
  520.04kB  1.01% 79.15%   520.04kB  1.01%  golang.org/x/net/http2.NewFramer.func1
```

从profile中可以得到以下信息：

1. chunk.newVarLenColumn、executor.(*indexWorker).extractTaskHandles、chunk.newFixedLenColumn集中申请了大部分的内存
2. 大部分申请内存的需求都是在planner.Optimize、executor.(*indexWorker).fetchHandles中产生的

##### 3.3.3.5 IO

![Workload e disk performance](./profiles/ycsb/tidb/workloade/e_disk_perf.png)

![Workload e tikv summart](./profiles/ycsb/tidb/workloade/e_tikv_summary.png)

![Workload e tikv network](./profiles/ycsb/tidb/workloade/e_tikv_net.png)

根据以上监控数据可以得出：

1. 网络吞吐没有引起程序瓶颈
2. 磁盘利用率较高，但仍然有提升空间
3. 磁盘利用率抖动比较明显

##### 3.3.4.6 结论

- Workload e是95%的`Scan`操作和5%的`Insert`操作组成的负载
- `Read`的QPS由于加入了5%的`Insert`操作发生了显著下降
- 在现有硬件资源和拓扑结构下，等待磁盘IO带来的高开销是无法避免的
- GC的开销占比较大，而内存的申请主要发生在执行器获取数据和计划器优化执行计划时

## 4. 总结

本次测试报告中，TiKV节点的磁盘IO引起了瓶颈，这是预期中的结果。Workload a|c|e 覆盖了YCSB中所有SQL操作，在源码不透明的情况下，测试明显反馈出了以下问题：

- workload a&workload c中都出现了handle.(*Handle).HandleAutoAnalyze引发的大量内存申请，这有可能引发GC较高的CPU开销
- planner.Optimize和executor.(*indexWorker).fetchHandles在workload e中申请使用了大量内存，直接引起了GC较高的CPU开销

因此共提出以下2条优化建议：

1. 降低根据主键的标量表达式点查较重负载下`handle.(*Handle).HandleAutoAnalyze`的内存申请
2. 降低根据主键的标量表达式范围查询较重的负载下`planner.Optimize`和`executor.(*indexWorker).fetchHandles`内存申请
