---
title: 从大数据量分库分表 MySQL 合并迁移数据到 TiDB
summary: 使用 Dumpling 和 TiDB Lightning 合并导入分表数据到 TiDB，以及如何使用 DM 持续增量复制数据。本文介绍的方法适用于导入数据总量大于 1 TiB 的场景。
---

# 从大数据量分库分表 MySQL 合并迁移数据到 TiDB

如果分表数据总规模特别大（例如大于 1 TiB），并且允许 TiDB 集群在迁移期间无其他业务写入，那么你可以使用 TiDB Lightning 对分表数据进行快速合并导入，然后根据业务需要选择是否使用 TiDB DM 进行增量数据的分表同步。本文所称“大数据量”通常指 TiB 级别以上。本文档举例介绍了导入数据的操作步骤。

如果分库分表合并迁移在 1 TiB 以内，请参考[TiB 级以下分库分表 MySQL 迁移数据到 TiDB](https://docs.pingcap.com/zh/tidb-data-migration/stable/usage-scenario-shard-merge)，支持全量和增量且更为简单。

使用 TiDB Lightning 快速合并导入的原理如下图所示。

![使用 Dumpling 和 TiDB Lightning 合并导入分表数据](/media/lightning/shard-merge-using-lightning.png)

在这个示例中，假设有两个数据库 my_db1 和 my_db2 ，使用 Dumpling 分别从 my_db1 中导出 table1 和 table2 两个表，从 my_db2 中导出 table3 和 table4 两个表，然后再用 TiDB Lightning 把导出的 4 个表合并导入到下游 TiDB 中的同一个库 my_db 的同一个表格 table5 中。

本文将以三个步骤演示导入流程：

1. 使用 Dumpling 导出全量数据备份。在本文档示例中，分别从 2 个源数据库中各导出 2 个表：
    - 从 实例 1 MySQL 的 my_db1 导出 table1、table2
    - 从 实例 2 MySQL 的 my_db2 导出 table3、table4
2. 启动 Lightning 执行导入 TiDB 中的 mydb.table5
3. 使用 DM 进行增量数据迁移（可选）

## 前提条件

- [使用 TiUP 安装 DM 集群](https://docs.pingcap.com/zh/tidb-data-migration/stable/deploy-a-dm-cluster-using-tiup)
- [使用 TiUP 安装 Dumpling 和 Lightning](/migration-tools.md)
- [DM 所需上下游数据库权限](https://docs.pingcap.com/zh/tidb-data-migration/stable/dm-worker-intro)
- [Lightning 所需下游数据库权限](/tidb-lightning/tidb-lightning-faq.md#tidb-lightning-对下游数据库的账号权限要求是怎样的)
- [Dumpling 所需上游数据库权限](/dumpling-overview.md#从-tidbmysql-导出数据)

### 资源要求

**操作系统**：本文档示例使用的是若干新的、纯净版 CentOS 7 实例，您可以在本地虚拟化或在供应商提供的平台上部署一台小型的云虚拟主机。TiDB Lightning 运行过程默认会占满 CPU，建议单独部署。如果条件不允许，可以和其他组件（比如 `tikv-server`）部署在同一台机器上，然后通过配置 `region-concurrency` 限制 TiDB Lightning 使用 CPU 资源。混合部署的情况下可以将其大小配置为逻辑 CPU 数的 75%，以限制 CPU 的使用。

**内存和 CPU**：因为 TiDB Lightning 对计算机资源消耗较高，建议分配 64 GB 以上的内存以及 32 核以上的 CPU，而且确保 CPU 核数和内存（GB）比为 1:2 以上，以获取最佳性能。

**磁盘空间**：

- Dumpling 需要足够储存整个数据源的存储空间。
- Lightning 导入期间需要排序键值对的临时存放空间，至少需要数据源最大单表的空间。
- 若全量数据量较大，可适当加长上游 binlog 保存时间，以避免增量同步时缺必要 binlog 导致重做。

**说明**：目前无法精确计算 Dumpling 从 MySQL 导出的数据大小，但你可以用下面 SQL 语句统计信息表的 data_length 字段估算数据量：

{{< copyable "sql" >}}

```sql
# 统计所有 schema 大小，单位 MiB，注意修改 ${schema_name}
select table_schema,sum(data_length)/1024/1024 as data_length,sum(index_length)/1024/1024 as index_length,sum(data_length+index_length)/1024/1024 as sum from information_schema.tables where table_schema = "${schema_name}" group by table_schema;

# 统计最大单表，单位 MiB，注意修改 ${schema_name}
select table_name,table_schema,sum(data_length)/1024/1024 as data_length,sum(index_length)/1024/1024 as index_length,sum(data_length+index_length)/1024/1024 as sum from information_schema.tables where table_schema = "${schema_name}" group by table_name,table_schema order by sum  desc limit 5;
```

### 目标 TiKV 集群的磁盘空间要求

**磁盘空间**：目标 TiKV 集群必须有足够空间接收新导入的数据。除了[标准硬件配置](/hardware-and-software-requirements.md)以外，目标 TiKV 集群的总存储空间必须大于 **数据源大小 × [副本数量](/faq/deploy-and-maintain-faq.md#每个-region-的-replica-数量可配置吗调整的方法是) × 2**。例如集群默认使用 3 副本，那么总存储空间需为数据源大小的 6 倍以上。公式中的 2 倍可能难以理解，其依据是以下因素的估算空间占用：

* 索引会占据额外的空间
* RocksDB 的空间放大效应

### 分表数据冲突检查

迁移中如果涉及合库合表，来自多张分表的数据可能引发主键或唯一索引的数据冲突。因此在迁移之前，需要检查各分表数据的业务特点。详情请参考[跨分表数据在主键或唯一索引冲突处理](https://docs.pingcap.com/zh/tidb-data-migration/stable/shard-merge-best-practices#跨分表数据在主键或唯一索引冲突处理),这里做简要描述：

假设 table1~4 具有相同的表结构如下：

```sql
CREATE TABLE `table1` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `sid` bigint(20) NOT NULL,
  `pid` bigint(20) NOT NULL,
  `comment` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `sid` (`sid`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

其中`id`列为主键，具有自增属性，多个分表范围重复会引发数据冲突。`sid`列为分片键，可以保证全局满足唯一索引。因此可以移除下游`table5`表`id`列的唯一键属性

```sql
CREATE TABLE `table5` (
  `id` bigint(20) NOT NULL,
  `sid` bigint(20) NOT NULL,
  `pid` bigint(20) NOT NULL,
  `comment` varchar(255) DEFAULT NULL,
  INDEX (`id`),
  UNIQUE KEY `sid` (`sid`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

## 第 1 步：用 Dumpling 导出全量数据备份

如果需要导出的多个分表属于同一个上游 MySQL 实例，建议直接使用 Dumpling 的 `-f` 参数一次导出多个分表的结果。如果多个分表分布在不同的 MySQL 实例，可以使用 Dumpling 分两次导出，并将两次导出的结果放置在相同的父目录下即可。下面的例子中同时用到了上述两种方式，然后将导出的数据存放在同一父目录下。

首先使用 Dumpling 从 my_db1 中导出表 table1 和 table2，如下：

```
tiup dumpling -h ${ip} -P 3306 -u root -t 16 -r 200000 -F 256MB -B my_db1 -f 'my_db1.table[12]' -o ${data-path}/my_db1
```

以上命令行中用到的参数描述如下。要了解更多 Dumpling 参数，请参考 [Dumpling 使用文档](/dumpling-overview.md)。

| 参数              |  描述 |
|-                  |-|
|-u 或 --user       |MySQL 数据库的用户|
|-p 或 --password   |MySQL 数据库的用户密码|
|-P 或 --port       |MySQL 数据库的端口|
|-h 或 --host       |MySQL 数据库的 IP 地址|
|-t 或 --thread     |导出的线程数。增加线程数会增加 Dumpling 并发度提高导出速度，但也会加大数据库内存消耗，因此不宜设置过大，一般不超过 64。|
|-o 或 --output     |存储导出文件的目录，支持本地文件路径或[外部存储 URL 格式](/br/backup-and-restore-storages.md)|
|-r 或 --row        |用于指定单个文件的最大行数，指定该参数后 Dumpling 会开启表内并发加速导出，同时减少内存使用。|
|-F                 |指定单个文件的最大大小，单位为 MiB。强烈建议使用`-F`参数以避免单表过大导致备份过程中断|
|-B 或 --database  | 导出指定数据库 |
|-f 或 --filter | 导出能匹配模式的表，语法可参考 [table-filter](/table-filter.md)。|

然后使用同样的方式从 my_db2 中导出表 table3 和 table4。注意路径在相同`${data-path}`下的不同子目录`my_db2`。

{{< copyable "shell-regular" >}}

```shell
tiup dumpling -h ${ip} -P 3306 -u root -t 16 -r 200000 -F 256MB -B my_db2 -f 'my_db2.table[34]' -o ${data-path}/my_db2
```

这样所需的全量备份数据就全部导出到了 `${data-path}` 目录中。将所有源数据表格存储在一个目录中，是为了后续方便用 TiDB Lightning 导入。

第 3 步增量同步的时候所需的起始位点信息，在`${data-path}`目录下,`my_db1`和`my_db2`的`metadata`文件中，这是 Dumpling 自动生成的元信息文件，请记录其中的 binlog 位置信息。

## 第 2 步：启动 TiDB Lightning 进行导入

在启动 TiDB Lightning 进行迁移之前，建议先了解如何处理检查点，然后根据需要选择合适的方式进行迁移。

### 断点续传

大量数据导入一般耗时数小时甚至数天，长时间运行的进程会有一定机率发生非正常中断。如果每次重启都从头开始，之前已成功导入的数据就会前功尽弃。为此，TiDB Lightning 提供了断点续传的功能，即使 TiDB Lightning 崩溃，在重启时仍然从断点开始继续工作。

若 TiDB Lightning 因不可恢复的错误而退出，例如数据出错，在重启时不会使用断点，而是直接报错离开。为保证已导入的数据安全，必须先解决掉这些错误才能继续。你可以使用`tidb-lightning-ctl` 命令控制导入出错后的行为。该命令的选项有：

* --checkpoint-error-destroy：出现错误后，让失败的表从头开始整个导入过程。
* --checkpoint-error-ignore：如果导入表曾经出错，该命令会清除出错状态，如同错误没有发生过一样。
* --checkpoint-remove：无论是否有出错，把表的断点清除。

关于断点续传的更多信息，请参考 [TiDB Lightning 断点续传](/tidb-lightning/tidb-lightning-checkpoints.md)。

### 创建目标表结构

根据前述“分表数据冲突检查”修改后，手动在下游 TiDB 建 my_db 库和 table5 表。之后需要在导入过程中将`tidb-lightning.toml`中设置。

```
[mydumper]
no-schema = true # 若已经在下游创建好库和表，此项设为 true 表示不进行 schema 创建
```

### 执行导入操作

启动 tidb-lightning 的步骤如下：

1. 编写配置文件`tidb-lightning.toml`。

    {{< copyable "" >}}

    ```

    [lightning]
    # 日志
    level = "info"
    file = "tidb-lightning.log"

    [tikv-importer]
    # "local"：默认使用该模式，适用于 TB 级以上大数据量，但导入期间下游 TiDB 无法对外提供服务。
    # "tidb"：TB 级以下数据量也可以采用`tidb`后端模式，下游 TiDB 可正常提供服务。 关于后端模式更多信息请参阅：https://docs.pingcap.com/tidb/stable/tidb-lightning-backends
    backend = "local"
    # 设置排序的键值对的临时存放地址，目标路径需要是一个空目录，至少需要数据源最大单表的空间，建议与 `data-source-dir` 不同磁盘目录，独占 IO 会获得更好的导入性能
    sorted-kv-dir = "${sorted-kv-dir}"

    # 设置分库分表合并规则，将 my_db1 中的 table1、table2 两个表,以及 my_db2 中的 table3、table4 两个表，共计 2 个数据库中的 4 个表都导入到目的数据库 my_db 中的 table5 表中。
    [[routes]]
    schema-pattern = "my_db1"
    table-pattern = "table[1-2]"
    target-schema = "my_db"
    target-table = "table5"

    [[routes]]
    schema-pattern = "my_db2"
    table-pattern = "table[3-4]"
    target-schema = "my_db"
    target-table = "table5"

    [mydumper]
    # 源数据目录。设置为 Dumpling 导出数据的路径，如果 Dumpling 执行了多次并分属不同的目录，请将多次导出的数据置放在相同的父目录下并指定此父目录即可。
    data-source-dir = "${data-path}" # 本地或 S3 路径，例如：'s3://my-bucket/sql-backup?region=us-west-2'
    # 由于 table1~4 合并为 table5。所以这里不创建表库，避免根据 Dumpling 的导出文件在下游创建出 table1~4。
    no-schema = true

    # 目标集群的信息，示例仅供参考。请把 IP 地址等信息替换成真实的信息。
    [tidb]
    # 目标集群的信息
    host = ${host}              # 例如：172.16.32.1
    port = ${port}              # 例如：4000
    user = "${user_name}"       # 例如："root"
    password = "${password}"    # 例如："rootroot"
    status-port = ${status-port} # 导入过程 Lightning 需要在从 TiDB 的“状态端口”获取表结构信息，例如：10080
    # PD 集群的地址，Lightning 通过 PD 获取部分信息。
    pd-addr = "${ip}:${port}"   # 例如 172.16.31.3:2379。当 backend = "local" 时 status-port 和 pd-addr 必须正确填写，否则导入将出现异常。

    ```

2. 运行 `tidb-lightning`。如果直接在命令行中启动程序，可能会因为 `SIGHUP` 信号而退出，建议配合`nohup`或`screen`等工具，如：

    若从 S3 导入，则需将有权限访问该 Amazon S3 后端存储的账号的 SecretKey 和 AccessKey 作为环境变量传入 Lightning 节点。同时还支持从 `~/.aws/credentials` 读取凭证文件。

    {{< copyable "shell-regular" >}}

    ```shell
    export AWS_ACCESS_KEY_ID=${access_key}
    export AWS_SECRET_ACCESS_KEY=${secret_key}
    nohup tiup tidb-lightning -config tidb-lightning.toml -no-schema=true > nohup.out 2>&1 &
    ```

3. 导入开始后，可以采用以下任意方式查看进度：

   - 通过 `grep` 日志关键字 `progress` 查看进度，默认 5 分钟更新一次。
   - 通过监控面板查看进度，请参考 [TiDB Lightning 监控](/tidb-lightning/monitor-tidb-lightning.md)。
   - 通过 Web 页面查看进度，请参考 [Web 界面](/tidb-lightning/tidb-lightning-web-interface.md)。

4. 导入完毕后，TiDB Lightning 会自动退出。查看日志的最后 5 行中会有 `the whole procedure completed`，则表示导入成功。

> **注意：**
>
> 无论导入成功与否，最后一行都会显示 `tidb lightning exit`。它只是表示 TiDB Lightning  正常退出，不代表任务完成。

如果导入过程中遇到问题，请参见 [TiDB Lightning 常见问题](/tidb-lightning/tidb-lightning-faq.md)。

## 第 3 步： 使用 DM 持续复制增量数据到 TiDB (可选)

基于 binlog 从指定位置同步数据库到 TiDB，可以使用 DM 来执行增量复制

### 添加数据源

新建`source1.yaml`文件, 写入以下内容：

{{< copyable "" >}}

```yaml

# 唯一命名，不可重复。
source-id: "mysql-01"

# DM-worker 是否使用全局事务标识符 (GTID) 拉取 binlog。使用前提是上游 MySQL 已开启 GTID 模式。若上游存在主从自动切换，则必须使用 GTID 模式。
enable-gtid: true

from:
  host: "${host}"           # 例如：172.16.10.81
  user: "root"
  password: "${password}"   # 支持但不推荐使用明文密码，建议使用 dmctl encrypt 对明文密码进行加密后使用
  port: 3306

```

在终端中执行下面的命令，使用`tiup dmctl`将数据源配置加载到 DM 集群中:

{{< copyable "shell-regular" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} operate-source create source1.yaml
```

该命令中的参数描述如下：

|参数           |描述|
|-              |-|
|--master-addr  |dmctl 要连接的集群的任意 DM-master 节点的 {advertise-addr}|
|operate-source create  |  向 DM 集群加载数据源  |

重复以上步骤直至所有 MySQL 实例被加入 DM。

### 添加同步任务

编辑`task.yaml`，配置增量同步模式，以及每个数据源的同步起点：

{{< copyable "" >}}

```yaml

name: task-test               # 任务名称，需要全局唯一。
task-mode: incremental        # 任务模式，设为 "incremental" 即只进行增量数据迁移。
# 分库分表合并任务则需要配置 shard-mode。默认使用悲观协调模式 "pessimistic"，在深入了解乐观协调模式的原理和使用限制后，也可以设置为乐观协调模式 "optimistic"
# 详细信息可参考：https://docs.pingcap.com/zh/tidb-data-migration/stable/feature-shard-merge
shard-mode: "pessimistic"

## 配置下游 TiDB 数据库实例访问信息
target-database:              # 下游数据库实例配置。
  host: "${host}"             # 例如：127.0.0.1
  port: 4000
  user: "root"
  password: "${password}"     # 推荐使用经过 dmctl 加密的密文。

##  使用黑白名单配置需要同步的表
block-allow-list:             # 数据源数据库实例匹配的表的 block-allow-list 过滤规则集，如果 DM 版本早于 v2.0.0-beta.2 则使用 black-white-list。
  bw-rule-1:                  # 黑白名单配置项 ID。
    do-dbs: ["my_db1"]        # 迁移哪些库。这里将实例1的 my_db1 和 实例2的 my_db2 分别配置为两条 rule。以示例如何避免实例1的 my_db2 被同步。
  bw-rule-2:
    do-dbs: ["my_db2"]

routes:                               # 上游和下游表之间的路由 table routing 规则集
  route-rule-1:                       # 配置名称。将 my_db1 中的 table1 和 table2 合并导入下游 my_db.table5
    schema-pattern: "my_db1"          # 库名匹配规则，支持通配符 "*" 和 "?"
    table-pattern: "table[1-2]"       # 表名匹配规则，支持通配符 "*" 和 "?"
    target-schema: "my_db"            # 目标库名称
    target-table: "table5"            # 目标表名称
  route-rule-2:                       # 配置名称。将 my_db2 中的 table3 和 table4 合并导入下游 my_db.table5
    schema-pattern: "my_db2"
    table-pattern: "table[3-4]"
    target-schema: "my_db"
    target-table: "table5"

## 配置数据源，以两个数据源为例
mysql-instances:
  - source-id: "mysql-01"             # 数据源 ID，即 source1.yaml 中的 source-id
    block-allow-list: "bw-rule-1"     # 引入上面黑白名单配置。同步实例1的 my_db1
    route-rules: ["route-rule-1"]     # 引入上面表合并配置。
#       syncer-config-name: "global"  # 引用后面的 syncers 增量数据配置。
    meta:                             # task-mode 为 incremental 且下游数据库的 checkpoint 不存在时 binlog 迁移开始的位置; 如果 checkpoint 存在，则以 checkpoint 为准。
      binlog-name: "${binlog-name}"   # 第 1 步中 ${data-path}/my_db1/metadata 记录的日志位置，当上游存在主从切换时，必须使用 gtid。
      binlog-pos: ${binlog-position}
      # binlog-gtid:                  " 例如：09bec856-ba95-11ea-850a-58f2b4af5188:1-9"
  - source-id: "mysql-02"             # 数据源 ID，即 source1.yaml 中的 source-id
    block-allow-list: "bw-rule-2"     # 引入上面黑白名单配置。实例2的 my_db2
    route-rules: ["route-rule-2"]     # 引入上面表合并配置。

#       syncer-config-name: "global"  # 引用后面的 syncers 增量数据配置。
    meta:                             # task-mode 为 incremental 且下游数据库的 checkpoint 不存在时 binlog 迁移开始的位置; 如果 checkpoint 存在，则以 checkpoint 为准。
      # binlog-name: "${binlog-name}"   # 第 1 步中 ${data-path}/my_db2/metadata 记录的日志位置，当上游存在主从切换时，必须使用 gtid。
      # binlog-pos: ${binlog-position}
      binlog-gtid: "09bec856-ba95-11ea-850a-58f2b4af5188:1-9"

## 【可选配置】 如果增量数据迁移需要重复迁移已经在全量数据迁移中完成迁移的数据，则需要开启 safe mode 避免增量数据迁移报错。
##  该场景多见于以下情况：全量迁移的数据不属于数据源的一个一致性快照，随后从一个早于全量迁移数据之前的位置开始同步增量数据。
# syncers:            # sync 处理单元的运行配置参数。
#  global:           # 配置名称。
#    safe-mode: true # 设置为 true，会将来自数据源的 INSERT 改写为 REPLACE，将 UPDATE 改写为 DELETE 与 REPLACE，从而保证在表结构中存在主键或唯一索引的条件下迁移数据时可以重复导入 DML。在启动或恢复增量复制任务的前 1 分钟内 TiDB DM 会自动启动 safe mode。

```

关于任务的更多配置项，可以参考[DM 任务完整配置文件介绍](https://docs.pingcap.com/zh/tidb-data-migration/stable/task-configuration-file-full)

在你启动数据迁移任务之前，建议使用`check-task`命令检查配置是否符合 DM 的配置要求，以降低后期报错的概率。

{{< copyable "shell-regular" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} check-task task.yaml
```

使用 tiup dmctl 执行以下命令启动数据迁移任务。

{{< copyable "shell-regular" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} start-task task.yaml
```

该命令中的参数描述如下：

|参数|描述|
|-|-|
| --master-addr | dmctl 要连接的集群的任意 DM-master 节点的 {advertise-addr}，例如：172.16.10.71:8261 |
|start-task|命令用于创建数据迁移任务|

如果任务启动失败，可根据返回结果的提示进行配置变更后执行 start-task task.yaml 命令重新启动任务。遇到问题请参考 [故障及处理方法](https://docs.pingcap.com/zh/tidb-data-migration/stable/error-handling) 以及 [常见问题](https://docs.pingcap.com/zh/tidb-data-migration/stable/faq)

### 查看任务状态

如需了解 DM 集群中是否存在正在运行的迁移任务及任务状态等信息，可使用`tiup dmctl`执行`query-status`命令进行查询：

{{< copyable "shell-regular" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} query-status ${task-name}
```

关于查询结果的详细解读，请参考[查询状态](https://docs.pingcap.com/zh/tidb-data-migration/stable/query-status)

### 监控任务与查看日志

你可以通过 Grafana 或者日志查看迁移任务的历史状态以及各种内部运行指标。

- 通过 Grafana 查看

    如果使用 TiUP 部署 DM 集群时，正确部署了 Prometheus、Alertmanager 与 Grafana，则使用部署时填写的 IP 及 端口进入 Grafana，选择 DM 的 dashboard 查看 DM 相关监控项。

- 通过日志查看

    DM 在运行过程中，DM-worker, DM-master 及 dmctl 都会通过日志输出相关信息，其中包含迁移任务的相关信息。各组件的日志目录如下：

    - DM-master 日志目录：通过 DM-master 进程参数`--log-file`设置。如果使用 TiUP 部署 DM，则日志目录默认位于`/dm-deploy/dm-master-8261/log/`。
    - DM-worker 日志目录：通过 DM-worker 进程参数`--log-file`设置。如果使用 TiUP 部署 DM，则日志目录默认位于`/dm-deploy/dm-worker-8262/log/`。

## 探索更多

- [关于 Dumpling](/dumpling-overview.md)
- [关于 Lightning](/tidb-lightning/tidb-lightning-overview.md)
- [分库分表合并中的悲观/乐观模式](https://docs.pingcap.com/zh/tidb-data-migration/stable/feature-shard-merge)
- [暂停数据迁移任务](https://docs.pingcap.com/zh/tidb-data-migration/stable/pause-task)
- [恢复数据迁移任务](https://docs.pingcap.com/zh/tidb-data-migration/stable/resume-task)
- [停止数据迁移任务](https://docs.pingcap.com/zh/tidb-data-migration/stable/stop-task)
- [导出和导入集群的数据源和任务配置](https://docs.pingcap.com/zh/tidb-data-migration/stable/export-import-config)
- [处理出错的 DDL 语句](https://docs.pingcap.com/zh/tidb-data-migration/stable/handle-failed-ddl-statements)