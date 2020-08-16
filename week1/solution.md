# 高性能TiDB课程作业（第一周）

## 1. 安装&构建

### 1.1 PD

```bash
# 克隆PD源码
git clone https://github.com/pingcap/pd.git;
# 编译PD
make;
```

### 1.2 TiDB

```bash
# 克隆TiDB源码
git clone https://github.com/pingcap/tidb.git;
```

### 1.3 TiKV

```bash
# 克隆TiKV源码
git clone https://github.com/tikv/tikv.git;
# 安装nightly版本的rust编译工具链
rustup install nightly;
# 设置TiKV本地rust编译工具链为nightly
rustup default nightly;
# 编译TiKV
make;
```

## 2. 修改TiDB源码

修改store/tikv/kv.go

```go
func (s *tikvStore) Begin() (kv.Transaction, error) {
	txn, err := newTiKVTxn(s)
	if err != nil {
		return nil, errors.Trace(err)
	}
	fmt.Println("hello transaction")
	return txn, nil
}

// BeginWithStartTS begins a transaction with startTS.
func (s *tikvStore) BeginWithStartTS(startTS uint64) (kv.Transaction, error) {
	txn, err := newTikvTxnWithStartTS(s, startTS, s.nextReplicaReadSeed())
	if err != nil {
		return nil, errors.Trace(err)
	}
	fmt.Println("hello transaction")
	return txn, nil
}
```

## 3. 测试TiDB

```bash
# 确保修改不会导致软件测试失败
cd store;
go test test.v;
cd ../;
make test;
```

编译TiDB

```bash
# 编译TiDB
make;
```

## 4. 启动集群服务

### 4.1 启动 pd-server

```bash
cd pd/bin;
# 启动PD节点
./pd-server --name=pd1 \
    --data-dir=pd1 \
    --client-urls="http://127.0.0.1:2379" \
    --peer-urls="http://127.0.0.1:2380" \
    --initial-cluster="pd1=http://127.0.0.1:2380" \
    --log-file=pd1.log &;
# 检查后台状态
jobs;
```

### 4.2 启动 tikv-server

```bash
cd tikv/target/release;
# 确保用户可以打开文件数量不会小于TiKV的最小限制
sudo launchctl limit maxfiles 100000 200000;
# 启动TiKV节点1
./tikv-server --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20160" \
    --status-addr="127.0.0.1:20181" \
    --data-dir=tikv1 \
    --log-file=tikv1.log &;
# 启动TiKV节点2
./tikv-server --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20161" \
    --status-addr="127.0.0.1:20182" \
    --data-dir=tikv2 \
    --log-file=tikv2.log &;
# 启动TiKV节点3
./tikv-server --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20162" \
    --status-addr="127.0.0.1:20183" \
    --data-dir=tikv3 \
    --log-file=tikv3.log &;
# 检查后台状态
jobs;
```

### 4.3 启动 tidb-server

```bash
cd tidb/bin;
# 启动TiDB节点
./tidb-server --store=tikv \
    --path="127.0.0.1:2379" \
    --log-file=tidb.log&;
# 检查后台状态
jobs;
# 验证一下TiDB服务器工作良好
mysql -h 127.0.0.1 -u root -P 4000
use mysql;
select * from tidb;
```

| VARIABLE_NAME            | VARIABLE_VALUE                                                                              | COMMENT |
|----                      |----                                                                                         |-----    |
| bootstrapped             | True                                                                                        | Bootstrap flag. Do not delete.                                                              |
| tidb_server_version      | 49                                                                                          | Bootstrap version. Do not delete.                                                           |
| system_tz                | Asia/Shanghai                                                                               | TiDB Global System Timezone.                                                                |
| new_collation_enabled    | False                                                                                       | If the new collations are enabled. Do not edit it.                                          |
| tikv_gc_leader_uuid      | 5cfd55c1e400001                                                                             | Current GC worker leader UUID. (DO NOT EDIT)                                                |
| tikv_gc_leader_desc      | host:rmbp.local.cc, pid:82357, start at 2020-08-16 11:36:28.307964 +0800 CST m=+0.221490405 | Host name and pid of current GC leader. (DO NOT EDIT)                                       |
| tikv_gc_leader_lease     | 20200816-11:58:28 +0800                                                                     | Current GC worker leader lease. (DO NOT EDIT)                                               |
| tikv_gc_enable           | true                                                                                        | Current GC enable status                                                                    |
| tikv_gc_run_interval     | 10m0s                                                                                       | GC run interval, at least 10m, in Go format.                                                |
| tikv_gc_life_time        | 10m0s                                                                                       | All versions within life time will not be collected by GC, at least 10m, in Go format.      |
| tikv_gc_last_run_time    | 20200816-11:49:28 +0800                                                                     | The time when last GC starts. (DO NOT EDIT)                                                 |
| tikv_gc_safe_point       | 20200816-11:39:28 +0800                                                                     | All versions after safe point can be accessed. (DO NOT EDIT)                                |
| tikv_gc_auto_concurrency | true                                                                                        | Let TiDB pick the concurrency automatically. If set false, tikv_gc_concurrency will be used |
| tikv_gc_scan_lock_mode   | physical                                                                                    | Mode of scanning locks, "physical" or "legacy"                                              |
| tikv_gc_mode             | distributed                                                                                 | Mode of GC, "central" or "distributed"                                                      |
