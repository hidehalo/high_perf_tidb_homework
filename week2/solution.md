# 高性能TiDB课程作业（第二周）

分值：300题目描述：

使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。测试报告需要包括以下内容：

1. 部署环境的机器配置(CPU、内存、磁盘规格型号)，拓扑结构(TiDB、TiKV 各部署于哪些节点)
2. 调整过后的 TiDB 和 TiKV 配置
3. 测试输出结果
    - 关键指标的监控截图
    - TiDB Query Summary 中的 qps 与 duration
    - TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
    - TiKV Details 面板中 grpc 的 qps 以及 duration输出：

写出你对该配置与拓扑环境和 workload 下 TiDB 集群负载的分析，提出你认为的 TiDB 的性能的瓶颈所在(能提出大致在哪个模块即 可)截止时间：下周二（8.25）24:00:00(逾期提交不给分)

## 1. 机器配置

- CPU 3.1 GHz 四核Intel Core i5
- 内存 16 GB 1867 MHz DDR3
- 磁盘规格型号 APPLE HDD HTS541010A9E662 1 TB 5400 RPM

## 2. 拓扑结构

| IP地址             | 角色                  | 操作系统        | 核数          | 内存|
| ---               | ---                   | ---           | ---           | ---|
| 192.168.99.101    | TiDB Server           | CentOS 8.3.1  | 1vCPU         | 2GB|
| 192.168.99.102    | TiKV Server           | CentOS 8.3.1  | 4vCPU         | 4GB|
| 192.168.99.103    | PD Server             | CentOS 8.3.1  | 1vCPU         | 1GB|
| 192.168.99.103    | Prometheus Server     | -             | -             | -  |
| 192.168.99.103    | Grafana Server        | -             | -             | -  |
| 192.168.99.103    | Alertmanager Server   | -             | -             | -  |

```bash
# 初始化集群部署环境
tiup cluster deploy tidb-cluster-1 v4.0.0  topo.yaml -u root -p
# 开始部署
tiup cluster start tidb-cluster-1
# 检查集群的结点状态
tiup cluster display tidb-cluster-1
```

### 2.1 TiDB&TiKV服务器配置

```yaml
server_configs:
  tidb:
    log.slow-threshold: 300
    binlog.enable: false
    binlog.ignore-error: false
  tikv:
    server.grpc-concurrency: 2
    rocksdb.max-background-jobs: 4
    raftdb.max-background-jobs: 4
    readpool.unified.max-thread-count: 3
    readpool.storage.use-unified-pool: false
    readpool.coprocessor.use-unified-pool: true
```

### 2.2 集群状态

> tidb Cluster: tidb-cluster-1
> tidb Version: v4.0.0

| ID                    |Role           |Host             |Ports         | OS/Arch       | Status      | Data Dir                      | Deploy Dir|
| --                    | ----          | ----            | -----        | -------       | ------      | --------                      | ----------|
| 192.168.99.103:9093   | alertmanager  | 192.168.99.103  | 9093/9094    | linux/x86_64  | Up          | /tidb-data/alertmanager-9093  | /tidb-deploy/alertmanager-9093|
| 192.168.99.103:3000   | grafana       | 192.168.99.103  | 3000         | linux/x86_64  | activating  | -                             | /tidb-deploy/grafana-3000|
| 192.168.99.103:2379   | pd            | 192.168.99.103  | 2379/2380    | linux/x86_64  | Up\|L\|UI   | /tidb-data/pd-2379            | /tidb-deploy/pd-2379|
| 192.168.99.103:9090   | prometheus    | 192.168.99.103  | 9090         | linux/x86_64  | activating  | /tidb-data/prometheus-9090    | /tidb-deploy/prometheus-9090|
| 192.168.99.101:4000   | tidb          | 192.168.99.101  | 4000/10080   | linux/x86_64  | Up          | -                             | /tidb-deploy/tidb-4000|
| 192.168.99.102:20160  | tikv          | 192.168.99.102  | 20160/20181  | linux/x86_64  | Up          | /tidb-data/tikv-20160         | /tidb-deploy/tikv-20160|

## 3. Sysbench压力测试

### 3.1 产生数据集

```bash
sysbench --config-file=sysbench.conf oltp_point_select --tables=32 --table-size=1000000 prepare
```

### 3.2 输出结果

```bash
sysbench --config-file=sysbench.conf oltp_point_select --tables=32 --table-size=1000000 run
```

```bash
sysbench --config-file=sysbench.conf oltp_update_index --tables=32 --table-size=1000000 run
```


```bash
sysbench --config-file=sysbench.conf oltp_read_only --tables=32 --table-size=1000000 run
```

### 3.3 TiDB query summary QPS&duration

### 3.4 TiKV details server's CPU&QPS

### 3.5 TiKV details GRPC's QPS&duration

## 3.6 结论

## 4. TPC

### 4.1 TPC-C

#### 4.1.1 产生数据集

```bash
# Create 4 warehouses and use 4 partitions by HASH 
./bin/go-tpc tpcc --warehouses 4 --parts 4 prepare
```

#### 4.1.2 输出结果

```bash
# Run TPCC workloads, you can just run or add --wait option to including wait times
./bin/go-tpc tpcc --warehouses 4 run
# Run TPCC including wait times(keying & thinking time) on every transactions
./bin/go-tpc tpcc --warehouses 4 run --wait
```

#### 4.1.3 TiDB query summary QPS&duration

TODO

#### 4.1.4 TiKV details server's CPU&QPS

TODO

#### 4.1.5 TiKV details GRPC's QPS&duration

TODO

#### 4.1.6 结论

TODO

### 4.2 TPC-H

TODO

#### 4.2.1 产生数据集

```bash
# Prepare data with scale factor 1
./bin/go-tpc tpch --sf=1 prepare
# Prepare data with scale factor 1, create tiflash replica, and analyze table after data loaded
./bin/go-tpc tpch --sf 1 --analyze --tiflash prepare
```

TODO

#### 4.2.2 输出结果

```bash
# Run TPCH workloads with result checking
./bin/go-tpc tpch --sf=1 --check=true run
# Run TPCH workloads without result checking
./bin/go-tpc tpch --sf=1 run
```

TODO

#### 4.2.3 TiDB query summary QPS&duration

TODO

#### 4.2.4 TiKV details server's CPU&QPS

TODO

#### 4.2.5 TiKV details GRPC's QPS&duration

TODO

#### 4.2.6 结论

TODO

## 5. YCSB

TODO

### 5.1 产生数据集

```bash
# Load
./bin/go-ycsb load basic -P workloads/workloada
```

TODO

### 5.2 输出结果

```bash
# Run
./bin/go-ycsb run basic -P workloads/workloada
```

TODO

### 5.3 TiDB query summary QPS&duration

TODO

### 5.4 TiKV details server's CPU&QPS

TODO

### 5.5 TiKV details GRPC's QPS&duration

TODO

### 5.6 结论

TODO
