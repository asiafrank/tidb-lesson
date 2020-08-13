# lesson01 TiDB开发环境搭建
要求：编译部署
- 1 TiDB
- 1 PD
- 3 TiKV

修改源码后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的日志。<br />

> 本文开发环境基于 macOS

## 基础环境

### 1. 下载源码

```shell
git clone https://github.com/pingcap/tidb.git
git clone https://github.com/tikv/tikv.git
git clone https://github.com/pingcap/pd.git
```


### 2. 安装 Rust 环境

因国内网络环境原因，切换下载源，然后再安装：

```shell
# 设置环境变量 RUSTUP_DIST_SERVER （用于更新 toolchain）：
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
# RUSTUP_UPDATE_ROOT （用于更新 rustup）：
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```

安装脚本下载及安装：

```shell
wget https://cdn.jsdelivr.net/gh/rust-lang-nursery/rustup.rs/rustup-init.sh
sh rustup-init.sh
```


### 3. 配置cargo环境变量

将环境变量配置在 `~/.bash_profile` 中即可：

```shell
export PATH="$HOME/.cargo/bin:$PATH"
```

### 4. 安装 go 环境

从 [官网](https://golang.org/dl/) 下载 macOS，安装包，执行即可。<br />

### 5. 编译

> 基于 master 分支编译

编译 pd：执行 `make`

编译 tidb：执行 `make`

编译 TiKV：执行 `make build` 。

> 编译TiKV，如果使用 `make dev` ，会跑一遍单元测试，有的单元测试并不能正常通过。

### 6. 启动

按要求启动 1 PD，1 TiDB，3 TiKV：

```shell
# 启动 pd
./bin/pd-server --name=pd1 \
                --data-dir=pd1 \
                --client-urls="http://127.0.0.1:2379" \
                --peer-urls="http://127.0.0.1:2380" \
                --initial-cluster="pd1=http://127.0.0.1:2380" \
                --log-file=pd1.log

# 启动 tikv 3 个节点
./tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20160" \
                --data-dir=tikv1 \
                --log-file=tikv1.log

./tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20161" \
                --data-dir=tikv2 \
                --log-file=tikv2.log

./tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20162" \
                --data-dir=tikv3 \
                --log-file=tikv3.log

# 验证 pd 和 tikv 启动正确
./bin/pd-ctl store -u http://127.0.0.1:2379

# 启动 tidb
./bin/tidb-server --store=tikv \
    --path="127.0.0.1:2379" \
    --log-file=tidb.log \
    # Remove the next line if you want verbose logging.
    -L=error
```

上述命令中，pd 端口 2379，tikv 和 tidb 启动时需要声明注册到 pd 的地址。

其中 tidb 默认客户端访问端口是 4000，状态通知端口是 10080。


更详细的命令行参数见：

- [tidb-server](https://docs.pingcap.com/tidb/stable/command-line-flags-for-tidb-configuration) 
- [tikv-server](https://docs.pingcap.com/tidb/stable/command-line-flags-for-tikv-configuration) 
- [pd-server](https://docs.pingcap.com/tidb/stable/command-line-flags-for-pd-configuration) 


### 7. mysql客户端访问试试

```shell
$ mysql -h 127.0.0.1 -P 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.25-TiDB-v4.0.0-beta.2-948-g148f2d456-dirty TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```


## 改源码试试

按要求：修改源码后，使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的日志。

搜索代码 transaction，定位到 kv\txn.go，store\tikv\txn.go，session\txn.go。

这三个文件看起来是与 transaction 相关度最高的了。

直觉告诉我们：kv\txn.go 是 store\tikv\txn.go 的抽象。

对于我们小白来说，自顶向下，才是正常的学习路径。因此从 kv\txn.go 来查找修改线索。


其中 `RunInNewTxn` 方法值得注意：

```go
// RunInNewTxn will run the f in a new transaction environment.
func RunInNewTxn(store Storage, retryable bool, f func(txn Transaction) error) error {
    var (
        err           error
        originalTxnTS uint64
        txn           Transaction
    )
    for i := uint(0); i < maxRetryCnt; i++ {
        txn, err = store.Begin()
        if err != nil {
            logutil.BgLogger().Error("RunInNewTxn", zap.Error(err))
            return err
        }

        // originalTxnTS is used to trace the original transaction when the function is retryable.
        if i == 0 {
            originalTxnTS = txn.StartTS()
        }

        err = f(txn)
        if err != nil {
            err1 := txn.Rollback()
            terror.Log(err1)
            if retryable && IsTxnRetryableError(err) {
                logutil.BgLogger().Warn("RunInNewTxn",
                    zap.Uint64("retry txn", txn.StartTS()),
                    zap.Uint64("original txn", originalTxnTS),
                    zap.Error(err))
                continue
            }
            return err
        }

        err = txn.Commit(context.Background())
        if err == nil {
            break
        }
        if retryable && IsTxnRetryableError(err) {
            logutil.BgLogger().Warn("RunInNewTxn",
                zap.Uint64("retry txn", txn.StartTS()),
                zap.Uint64("original txn", originalTxnTS),
                zap.Error(err))
            BackOff(i)
            continue
        }
        return err
    }
    return err
}
```

注释上写着“f 方法会在新的事务环境中运行”。那么就是这里没错了。

仔细观察发现， `store.Begin()` 语句会返回一个 `txn` 对象。那么我们只需在这个 Begin 方法里打印 `hello transaction` 即可。


故修改 `store/tikv/kv.go:281 Begin`  方法，使事务开始时打印 hello transaction

```go
func (s *tikvStore) Begin() (kv.Transaction, error) {
    logutil.BgLogger().Info("hello transaction")
    txn, err := newTiKVTxn(s)
    if err != nil {
        return nil, errors.Trace(err)
    }
    return txn, nil
}
```

重新编译 tidb，运行后结果如下：

```
[2020/08/13 13:45:48.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:49.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:49.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:50.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:50.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:51.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:51.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:52.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:52.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:53.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:53.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:54.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:54.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:55.206 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:55.206 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:56.206 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:56.206 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:57.208 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:57.209 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:58.207 +08:00] [INFO] [kv.go:282] ["hello transaction"]
[2020/08/13 13:45:58.207 +08:00] [INFO] [kv.go:282] ["hello transaction"]
```

## 疑问

日志中显示，即使没有任何语句执行，每隔 1秒，仍会打印一次 hello transaction。

定位发现，都是 `ddl_worker.go handleDDLJobQueue`  在调用。

**handleDDLJobQueue 该方法的作用是什么？为什么需要 Transaction？** 

> TiDB有个特性，DDL的修改并不锁表。

初步推断，DDL 是异步任务去批量处理的。Job 每间隔 1 秒左右时间，开启一个新事务，获取 DDL Job 来执行。<br />基于分布式系统考虑，这样的做法是逐步更新所有 tikv 集群中的 DDL，而非一次执行完毕。<br />所以会打印这么多 `hello transaction` 。<br />


# 参考资料

- [Community Contributors README](https://github.com/pingcap/community/blob/master/contributors/README.md) 


- [pd](https://github.com/pingcap/pd) 
- [pd contributing](https://github.com/pingcap/pd/blob/master/CONTRIBUTING.md) 
- [pd development](https://github.com/pingcap/pd/blob/master/docs/development.md) 
- [tikv deploy using binary](https://github.com/tikv/tikv/blob/master/docs/how-to/deploy/using-binary.md) 
- [building running and benchmarking tikv and tidb](https://pingcap.com/blog/building-running-and-benchmarking-tikv-and-tidb/) 



