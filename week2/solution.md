# 高性能TiDB课程作业（第二周）

分值：300题目描述：

使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。测试报告需要包括以下内容：

1.  部署环境的机器配置(CPU、内存、磁盘规格型号)，拓扑结构(TiDB、TiKV 各部署于哪些节点)
2. 调整过后的 TiDB 和 TiKV 配置
3. 测试输出结果
    - 关键指标的监控截图
    - TiDB Query Summary 中的 qps 与 duration
    - TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
    - TiKV Details 面板中 grpc 的 qps 以及 duration输出：

写出你对该配置与拓扑环境和 workload 下 TiDB 集群负载的分析，提出你认为的 TiDB 的性能的瓶颈所在(能提出大致在哪个模块即 可)截止时间：下周二（8.25）24:00:00(逾期提交不给分)


## 1. 机器配置

CPU 3.1 GHz 四核Intel Core i5
内存 16 GB 1867 MHz DDR3
磁盘规格型号 APPLE HDD HTS541010A9E662 1 TB 5400 RPM

## 2. 拓扑结构

本次测试一共使用了三台虚拟机，每台虚拟机分配了2048MB内存和1个虚拟CPU的上限，以下是服务器的网络拓扑结构

* TiDB Server 192.168.99.101 (VM)
* TiKV Server 192.168.99.102 (VM)
* PD Server 192.168.99.103 (VM)
* Prometheus Server 192.168.99.103 (VM)
* Grafana Server 92.168.99.103 (VM)
* Alertmanager Server 92.168.99.103 (VM)

```bash
tiup cluster deploy tidb-cluster-1 v4.0.0  topo.yaml -u root -p
```
# Sysbench

[sysbench test](https://docs.pingcap.com/zh/tidb/stable/benchmark-tidb-using-sysbench#%E6%95%B0%E6%8D%AE%E5%AF%BC%E5%85%A5)

```bash
sysbench --config-file=sysbench.conf oltp_point_select --tables=32 --table-size=10000000 prepare
sysbench --config-file=config oltp_point_select --tables=32 --table-size=10000000 run
sysbench --config-file=config oltp_update_index --tables=32 --table-size=10000000 run
sysbench --config-file=config oltp_read_only --tables=32 --table-size=10000000 run
```

# TPC

## TPC-C

[tpc-c test](https://docs.pingcap.com/zh/tidb/stable/benchmark-tidb-using-tpcc#%E5%A6%82%E4%BD%95%E5%AF%B9-tidb-%E8%BF%9B%E8%A1%8C-tpc-c-%E6%B5%8B%E8%AF%95)

```bash
# Prepare
# Create 4 warehouses and use 4 partitions by HASH 
./bin/go-tpc tpcc --warehouses 4 --parts 4 prepare

# Run
# Run TPCC workloads, you can just run or add --wait option to including wait times
./bin/go-tpc tpcc --warehouses 4 run
# Run TPCC including wait times(keying & thinking time) on every transactions
./bin/go-tpc tpcc --warehouses 4 run --wait

# Check
# Check consistency. you can check after prepare or after run
./bin/go-tpc tpcc --warehouses 4 check

# Clean Up
# Cleanup
./bin/go-tpc tpcc --warehouses 4 cleanup

# Others
# Generate csv files (split to 100 files each table)
./bin/go-tpc tpcc --warehouses 4 prepare -T 100 --output-type csv --output-dir data
# Specified tables when generating csv files
./bin/go-tpc tpcc --warehouses 4 prepare -T 100 --output-type csv --output-dir data --tables history,orders
# Start pprof
./bin/go-tpc tpcc --warehouses 4 prepare --output-type csv --output-dir data --pprof :10111
```

## TPC-H

```bash
# Prepare data with scale factor 1
./bin/go-tpc tpch --sf=1 prepare
# Prepare data with scale factor 1, create tiflash replica, and analyze table after data loaded
./bin/go-tpc tpch --sf 1 --analyze --tiflash prepare

# Run TPCH workloads with result checking
./bin/go-tpc tpch --sf=1 --check=true run
# Run TPCH workloads without result checking
./bin/go-tpc tpch --sf=1 run

# Cleanup
./bin/go-tpc tpch cleanup

# CH-benCHmark
# Prepare data
./bin/go-tpc ch prepare
# Prepare data, create tiflash replica, and analyze table after data loaded
./bin/go-tpc ch --analyze --tiflash prepare

# Run
./bin/go-tpc ch --warehouses $warehouses -T $tpWorkers -t $apWorkers --time $measurement-time run
```

# TiDB Dashboard

[TiDB dashboard](https://docs.pingcap.com/zh/tidb/stable/dashboard-diagnostics-usage#%E7%94%A8%E5%AF%B9%E6%AF%94%E6%8A%A5%E5%91%8A%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98)

# YCSB

```bash
# Load
./bin/go-ycsb load basic -P workloads/workloada
# Run
./bin/go-ycsb run basic -P workloads/workloada
```