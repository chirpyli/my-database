## PostgreSQL备份恢复工具——wal-g
目前开源的备份恢复工具有很多，比如pgbackrest、pg_probackup、barman等，这里我们讲一下另一个备份工具wal-g的使用。



### 安装部署
wal-g是采用Go语言实现的，所以，需要先安装Go环境，在进行编译安装。
1. 安装go环境，参考Golang官网文档[Download and install](https://golang.google.cn/doc/install)进行安装即可。
2. 可编译安装部署，亦可直接下载安装包。这里我们采用编译安装的方式，可参考文档[Installing](https://github.com/wal-g/wal-g?tab=readme-ov-file#installing)

安装完成后，可--help检查一下是否安装成功。
```shell
postgres@slpc:~$ wal-g --help
PostgreSQL backup tool

Usage:
wal-g [command]

Available Commands:
backup-fetch  Fetches a backup from storage
backup-list   Prints full list of backups from which recovery is available
backup-mark   Marks a backup permanent or impermanent
backup-push   Makes backup and uploads it to storage
catchup-fetch Fetches an incremental backup from storage
catchup-list  Prints available incremental backups
catchup-push  Creates incremental backup from lsn
completion    Output shell completion code for the specified shell
copy          copy specific or all backups
daemon        Runs WAL-G in daemon mode which executes commands sent from the lightweight walg-daemon-client.
delete        Clears old backups and WALs
flags         Display the list of available global flags for all wal-g commands
help          Help about any command
pgbackrest    Interact with pgbackrest backups (beta)
st            (DANGEROUS) Storage tools
wal-fetch     Fetches a WAL file from storage
wal-push      Uploads a WAL file to storage
wal-receive   Receive WAL stream with postgres Streaming Replication Protocol and push to storage
wal-restore   Restores WAL segments from storage.
wal-show      Show storage WAL segments info grouped by timelines.
wal-verify    Verify WAL storage folder. Available checks: integrity, timeline.

Flags:
      --config string   config file (default is $HOME/.walg.json)
  -h, --help            help for wal-g
      --turbo           Ignore all kinds of throttling defined in config
  -v, --version         version for wal-g

To get the complete list of all global flags, run: 'wal-g flags'

Use "wal-g [command] --help" for more information about a command.
```

### wal-g配置

wal-g有非常众多的配置，可通过环境变量进行配置或者使用配置文件，配置文件支持json、yaml以及[viper package](https://github.com/spf13/viper)支持的配置文件格式。

我们以最常用的json以及yaml为例进行说明：

#### 环境变量
通过环境变量进行配置的使用示例：
```bash
export WALG_COMPRESSION_METHOD=zstd   # 配置压缩算法
# 其他配置略...
```


#### json格式
```json
{"WALG_COMPRESSION_METHOD": "lzma"}
```
使用方式，`wal-g --config /configpath/*.json`。测试示例如下：
```shell
postgres@slpc:~$ wal-g wal-push /home/postgres/pg15.5/pgdata/pg_wal/00000001000000000000000F --config /home/postgres/pgdata/wal-g.json
INFO: 2024/02/29 10:53:36.578807 Files will be uploaded to storage: default
INFO: 2024/02/29 10:53:37.385081 FILE PATH: 00000001000000000000000F.lzma
```


#### yaml格式
例如：
```yaml
# cat /etc/wal-g/wal-g.yaml
PGDATABASE: "postgres"
WALE_S3_PREFIX: "s3://some/s3/prefix/"
WALG_NETWORK_RATE_LIMIT: 8388608
PGPASSFILE: "/home/gpadmin/.pgpass"
WALG_GP_LOGS_DIR: "/var/log/greenplum"
AWS_ACCESS_KEY_ID: "aws_access_key_id"
WALG_UPLOAD_CONCURRENCY: 5
WALG_PGP_KEY_PATH: "/path/to/PGP_KEY"
WALG_DOWNLOAD_CONCURRENCY: 5
WALG_DOWNLOAD_FILE_RETRIES: 15
WALE_GPG_KEY_ID: "gpg_key_id"
WALG_DISK_RATE_LIMIT: 167772160
PGUSER: "gpadmin"
GOMAXPROCS: 6
PGHOST: "localhost"
AWS_ENDPOINT: "https://s3-endpoint.host.name"
AWS_SECRET_ACCESS_KEY: "aws_secret_access_key"
WALG_COMPRESSION_METHOD: "brotli"
```
使用方式，`wal-g --config /configpath/*.yaml`。测试示例如下：
```shell
postgres@slpc:~$ wal-g wal-push /home/postgres/pg15.5/pgdata/pg_wal/00000001000000000000000F --config /home/postgres/pgdata/wal-g.yaml
INFO: 2024/02/29 11:08:24.272284 Files will be uploaded to storage: default
INFO: 2024/02/29 11:08:24.343777 FILE PATH: 00000001000000000000000F.br
```
### 备份


#### 备份到本地盘
通过`wal-g backup-push $PGDATA`的方式进行备份，设置环境变量，设置备份目录`WALG_FILE_PREFIX`，设置数据库实例位置`PGDATA`。
```bash
export WALG_FILE_PREFIX=/home/postgres/pgdata/backup
export WALG_COMPRESSION_METHOD=zstd
export PGDATA=/home/postgres/pg15.5/pgdata
```
示例如下：
```shell
postgres@slpc:~/pgdata$ PGHOST=192.168.109.133 PGPASSWORD=postgres wal-g backup-push $PGDATA
INFO: 2024/03/04 14:00:44.531702 Backup will be pushed to storage: default
ERROR: 2024/03/04 14:00:44.564208 unexpected message type
ERROR: 2024/03/04 14:00:44.564269 Failed to connect using provided PGHOST and PGPORT, trying localhost:5432
ERROR: 2024/03/04 14:00:44.572534 Connect: postgres connection failed: FATAL: unrecognized configuration parameter "gp_role" (SQLSTATE 42704)
postgres@slpc:~/pgdata$ PGHOST=192.168.109.133 wal-g backup-push $PGDATA
INFO: 2024/03/04 14:01:36.195250 Backup will be pushed to storage: default
INFO: 2024/03/04 14:01:36.277066 Calling pg_start_backup()
INFO: 2024/03/04 14:01:36.336652 Initializing the PG alive checker (interval=1m0s)...
INFO: 2024/03/04 14:01:36.336740 Starting a new tar bundle
INFO: 2024/03/04 14:01:36.336767 Walking ...
INFO: 2024/03/04 14:01:36.337112 Starting part 1 ...
INFO: 2024/03/04 14:01:36.566550 Packing ...
INFO: 2024/03/04 14:01:36.568290 Finished writing part 1.
INFO: 2024/03/04 14:01:36.568363 Starting part 2 ...
INFO: 2024/03/04 14:01:36.568987 /global/pg_control
INFO: 2024/03/04 14:01:36.578094 Finished writing part 2.
INFO: 2024/03/04 14:01:36.578148 Calling pg_stop_backup()
INFO: 2024/03/04 14:01:36.626079 Starting part 3 ...
INFO: 2024/03/04 14:01:36.627065 backup_label
INFO: 2024/03/04 14:01:36.627130 tablespace_map
INFO: 2024/03/04 14:01:36.629653 Finished writing part 3.
INFO: 2024/03/04 14:01:36.630428 Querying pg_database
INFO: 2024/03/04 14:01:36.730815 Wrote backup with name base_00000001000000000000000A to storage default

```
我们看一下备份目录的文件：
```shell
postgres@slpc:~/pgdata/backup$ tree .
.
├── basebackups_005
│   ├── base_00000001000000000000000A
│   │   ├── files_metadata.json
│   │   ├── metadata.json
│   │   └── tar_partitions
│   │       ├── part_001.tar.zst
│   │       ├── part_003.tar.zst
│   │       └── pg_control.tar.zst
│   └── base_00000001000000000000000A_backup_stop_sentinel.json
├── wal_005
│   ├── 000000010000000000000001.zst
│   ├── 000000010000000000000002.00000028.backup.zst
│   ├── 000000010000000000000002.zst
│   ├── 000000010000000000000003.zst

```

#### 恢复

数据库备份后可通过`wal-g backup-fetch`进行恢复。

用法示例：
```shell
wal-g backup-fetch bakmn LATEST
```
恢复日志如下：
```shell
postgres@slpc:~/pgdata$ wal-g backup-fetch bakmn LATEST
INFO: 2024/03/05 09:29:05.629353 Selecting the latest backup...
INFO: 2024/03/05 09:29:05.629786 Backup to fetch will be searched in storages: [default]
INFO: 2024/03/05 09:29:05.630010 LATEST backup is: 'base_00000001000000000000000A'
INFO: 2024/03/05 09:29:05.647958 Finished extraction of part_003.tar.zst
INFO: 2024/03/05 09:29:07.259976 Finished extraction of part_001.tar.zst
INFO: 2024/03/05 09:29:07.262729 Finished extraction of pg_control.tar.zst
INFO: 2024/03/05 09:29:07.262782 
Backup extraction complete.
```
在启动前创建`recovery.signal`文件
```shell
postgres@slpc:~/pgdata/bakmn$ touch recovery.signal
```
然后启动数据库
```shell
postgres@slpc:~/pgdata$ pg_ctl start -D bakmn
server started
```
恢复成功。我们验证一下：
```sql
postgres@slpc:~/pgdata/backup$ psql     --连接数据库
psql (15.5)
Type "help" for help.

postgres=# select * from t1;
 a 
---
 1
(1 row)

postgres=# insert into t1 values(2);   -- 插入数据
INSERT 0 1
postgres=# select pg_switch_wal();    -- 触发日志归档
 pg_switch_wal 
---------------
 0/D0001E8
(1 row)

postgres=# \q
postgres@slpc:~/pgdata/backup$ ls
basebackups_005  wal_005  wal-g.log
postgres@slpc:~/pgdata/backup$ tree wal_005/   -- 查看归档日志目录
wal_005/
├── 000000010000000000000001.zst
├── 000000010000000000000002.00000028.backup.zst
├── 000000010000000000000002.zst
├── 000000010000000000000003.zst
├── 000000010000000000000004.00000028.backup.zst
├── 000000010000000000000004.zst
├── 000000010000000000000005.zst
├── 000000010000000000000006.zst
├── 000000010000000000000007.zst
├── 000000010000000000000008.zst
├── 000000010000000000000009.zst
├── 00000001000000000000000A.00000028.backup.zst
├── 00000001000000000000000A.zst
├── 00000001000000000000000B.zst
├── 00000001000000000000000C.zst
├── 00000002000000000000000D.zst             -- 新的归档日志，时间线已切为2
└── 00000002.history.zst                     -- 时间线历史文件
```
每次恢复数据库，其时间线都会加1。 时间线用于区分原始数据库和恢复生成的数据库集簇，是PITR的核心概念。


#### 增量备份
设置环境变量
```bash
export WALG_DELTA_ORIGIN=LATEST_FULL
```
执行增量备份
```shell
postgres@slpc:~$ wal-g backup-push $PGDATA --delta-from-name base_000000020000000000000024
INFO: 2024/03/05 16:33:02.689154 Backup will be pushed to storage: default
INFO: 2024/03/05 16:33:02.698428 Selecting the backup with name base_000000020000000000000024 as the base for the current delta backup...
INFO: 2024/03/05 16:33:02.956011 Delta will be made from full backup.
INFO: 2024/03/05 16:33:02.968992 Delta backup from base_000000020000000000000024 with LSN 0/24000028.
INFO: 2024/03/05 16:33:03.075391 Calling pg_start_backup()
INFO: 2024/03/05 16:33:03.093341 Initializing the PG alive checker (interval=1m0s)...
INFO: 2024/03/05 16:33:03.093443 Delta backup enabled
INFO: 2024/03/05 16:33:03.093481 Starting a new tar bundle
INFO: 2024/03/05 16:33:03.093510 Walking ...
INFO: 2024/03/05 16:33:03.094918 Starting part 1 ...
INFO: 2024/03/05 16:33:03.128718 Packing ...
INFO: 2024/03/05 16:33:03.129758 Finished writing part 1.
INFO: 2024/03/05 16:33:03.163413 Starting part 2 ...
INFO: 2024/03/05 16:33:03.164104 /global/pg_control
INFO: 2024/03/05 16:33:03.176944 Finished writing part 2.
INFO: 2024/03/05 16:33:03.176973 Calling pg_stop_backup()
INFO: 2024/03/05 16:33:03.226442 Starting part 3 ...
INFO: 2024/03/05 16:33:03.227771 backup_label
INFO: 2024/03/05 16:33:03.227807 tablespace_map
INFO: 2024/03/05 16:33:03.233816 Finished writing part 3.
INFO: 2024/03/05 16:33:03.250012 Querying pg_database
INFO: 2024/03/05 16:33:03.553529 Wrote backup with name base_000000020000000000000029_D_000000020000000000000024 to storage default
```
查看备份文件
```shell
postgres@slpc:~$ wal-g st ls basebackups_005
type size last modified                     name
dir  0    0001-01-01 00:00:00 +0000 UTC     base_000000020000000000000024/
dir  0    0001-01-01 00:00:00 +0000 UTC     base_000000020000000000000029_D_000000020000000000000024/
obj  398  2024-03-05 06:55:53.786 +0000 UTC base_000000020000000000000024_backup_stop_sentinel.json
obj  522  2024-03-05 08:33:03.501 +0000 UTC base_000000020000000000000029_D_000000020000000000000024_backup_stop_sentinel.json
```
再次执行备份
```bash
postgres@slpc:~$ wal-g backup-push $PGDATA --delta-from-name base_000000020000000000000029_D_000000020000000000000024
INFO: 2024/03/05 16:49:52.073832 Backup will be pushed to storage: default
INFO: 2024/03/05 16:49:52.073934 Selecting the backup with name base_000000020000000000000029_D_000000020000000000000024 as the base for the current delta backup...
INFO: 2024/03/05 16:49:52.349094 Delta will be made from full backup.
INFO: 2024/03/05 16:49:52.362781 Delta backup from base_000000020000000000000024 with LSN 0/24000028.
INFO: 2024/03/05 16:49:52.443188 Calling pg_start_backup()
INFO: 2024/03/05 16:49:52.547250 Initializing the PG alive checker (interval=1m0s)...
INFO: 2024/03/05 16:49:52.547356 Delta backup enabled
INFO: 2024/03/05 16:49:52.547394 Starting a new tar bundle
INFO: 2024/03/05 16:49:52.547448 Walking ...
INFO: 2024/03/05 16:49:52.547831 Starting part 1 ...
INFO: 2024/03/05 16:49:52.585035 Packing ...
INFO: 2024/03/05 16:49:52.588192 Finished writing part 1.
INFO: 2024/03/05 16:49:52.631401 Starting part 2 ...
INFO: 2024/03/05 16:49:52.641163 /global/pg_control
INFO: 2024/03/05 16:49:52.650529 Finished writing part 2.
INFO: 2024/03/05 16:49:52.650607 Calling pg_stop_backup()
INFO: 2024/03/05 16:49:52.694601 Starting part 3 ...
INFO: 2024/03/05 16:49:52.697699 backup_label
INFO: 2024/03/05 16:49:52.697793 tablespace_map
INFO: 2024/03/05 16:49:53.183120 Finished writing part 3.
INFO: 2024/03/05 16:49:53.341786 Querying pg_database
INFO: 2024/03/05 16:49:53.730008 Wrote backup with name base_00000002000000000000002C_D_000000020000000000000024 to storage default
postgres@slpc:~$ wal-g st ls basebackups_005
type size last modified                     name
dir  0    0001-01-01 00:00:00 +0000 UTC     base_000000020000000000000024/
dir  0    0001-01-01 00:00:00 +0000 UTC     base_000000020000000000000029_D_000000020000000000000024/
dir  0    0001-01-01 00:00:00 +0000 UTC     base_00000002000000000000002C_D_000000020000000000000024/
obj  398  2024-03-05 06:55:53.786 +0000 UTC base_000000020000000000000024_backup_stop_sentinel.json
obj  522  2024-03-05 08:33:03.501 +0000 UTC base_000000020000000000000029_D_000000020000000000000024_backup_stop_sentinel.json
obj  522  2024-03-05 08:49:53.712 +0000 UTC base_00000002000000000000002C_D_000000020000000000000024_backup_stop_sentinel.json
postgres@slpc:~$ wal-g backup-list
INFO: 2024/03/05 16:59:06.308660 List backups from storages: [default]
backup_name                                              modified             wal_file_name            storage_name
base_000000020000000000000024                            2024-03-05T06:55:53Z 000000020000000000000024 default
base_000000020000000000000029_D_000000020000000000000024 2024-03-05T08:33:03Z 000000020000000000000029 default
base_00000002000000000000002C_D_000000020000000000000024 2024-03-05T08:49:53Z 00000002000000000000002C default

```


### 存储
wal-g可备份到S3、Google Cloud Storage、Azure、Swift、远端主机、本地盘

#### 本地盘
可通过`WALG_FILE_PREFIX`进行配置。例如归档日志到某个目录：
```bash
archive_command = 'export WALG_FILE_PREFIX=/home/postgres/pgdata/archive; wal-g wal-push %p'
```
一般生产情况下，不会采用这种方式。建议归档到S3。
#### 远端主机
可通过`WALG_SSH_PREFIX`配置归档或者备份到其他远端主机。
- WALG_SSH_PREFIX (e.g. ssh://localhost/walg-folder)
- SSH_PORT ssh connection port
- SSH_USERNAME connect with username
- SSH_PASSWORD connect with password
- SSH_PRIVATE_KEY_PATH or connect with a SSH KEY by specifying its full path

示例环境变量配置如下：
```bash
export WALG_SSH_PREFIX=ssh://192.168.109.136/home/postgres/pgdata/backup   # 备份到192.168.109.136机器的home/postgres/pgdata/backup 目录下
export SSH_USERNAME=postgres
export SSH_PASSWORD=postgres
export SSH_PORT=22
export WALG_COMPRESSION_METHOD=zstd
```
进行备份：
```bash
postgres@slpc:~$ wal-g backup-push $PGDATA --full
INFO: 2024/03/05 14:30:06.664883 Backup will be pushed to storage: default
INFO: 2024/03/05 14:30:06.701011 Doing full backup.
INFO: 2024/03/05 14:30:06.733931 Calling pg_start_backup()
INFO: 2024/03/05 14:30:06.810888 Initializing the PG alive checker (interval=1m0s)...
INFO: 2024/03/05 14:30:06.811014 Starting a new tar bundle
INFO: 2024/03/05 14:30:06.811053 Walking ...
INFO: 2024/03/05 14:30:06.811514 Starting part 1 ...
INFO: 2024/03/05 14:30:08.067431 Packing ...
INFO: 2024/03/05 14:30:08.069307 Finished writing part 1.
INFO: 2024/03/05 14:30:08.071450 Starting part 2 ...
INFO: 2024/03/05 14:30:08.072180 /global/pg_control
INFO: 2024/03/05 14:30:08.075420 Finished writing part 2.
INFO: 2024/03/05 14:30:08.075463 Calling pg_stop_backup()
INFO: 2024/03/05 14:30:08.265699 Starting part 3 ...
INFO: 2024/03/05 14:30:08.266629 backup_label
INFO: 2024/03/05 14:30:08.266706 tablespace_map
INFO: 2024/03/05 14:30:08.289423 Finished writing part 3.
INFO: 2024/03/05 14:30:08.296357 Querying pg_database
INFO: 2024/03/05 14:30:08.495505 Wrote backup with name base_00000002000000000000001D to storage default
postgres@slpc:~$ wal-g backup-list
INFO: 2024/03/05 14:30:27.814430 List backups from storages: [default]
backup_name                   modified                  wal_file_name            storage_name
base_00000002000000000000001D 2024-03-05T14:30:08+08:00 00000002000000000000001D default
```
登录192.168。109.136机器，查看备份文件
```bash
postgres@slpc:~/pgdata/backup$ tree .
.
├── basebackups_005
│   ├── base_00000002000000000000001D
│   │   ├── files_metadata.json
│   │   ├── metadata.json
│   │   └── tar_partitions
│   │       ├── part_001.tar.zst
│   │       ├── part_003.tar.zst
│   │       └── pg_control.tar.zst
│   └── base_00000002000000000000001D_backup_stop_sentinel.json
└── wal_005
    ├── 00000002000000000000001B.zst
    ├── 00000002000000000000001C.zst
    ├── 00000002000000000000001D.00000028.backup.zst
    └── 00000002000000000000001D.zst

4 directories, 10 files
```

#### S3
建议将备份以及日志归档到S3中，成本以及可靠性等均有优势。
需要配置`WALG_S3_PREFIX`为S3。

用法示例：
```bash
export WALG_S3_PREFIX=s3://pgbackup
export AWS_ENDPOINT=eos-shanghai-4.cmecloud.cn
export AWS_SECRET_ACCESS_KEY="ccNPsAeO0FtBIl90RqZgAG8O098GpNnel1BAWa4G"
export AWS_ACCESS_KEY_ID="IP51OA2Y0629WJH9PEB9"
```
```bash
postgres@slpc:~$ wal-g backup-push $PGDATA
INFO: 2024/03/05 14:55:51.787825 Backup will be pushed to storage: default
INFO: 2024/03/05 14:55:51.859070 Calling pg_start_backup()
INFO: 2024/03/05 14:55:51.925236 Initializing the PG alive checker (interval=1m0s)...
INFO: 2024/03/05 14:55:51.925473 Starting a new tar bundle
INFO: 2024/03/05 14:55:51.926138 Walking ...
INFO: 2024/03/05 14:55:51.926573 Starting part 1 ...
INFO: 2024/03/05 14:55:52.371235 Packing ...
INFO: 2024/03/05 14:55:52.372979 Finished writing part 1.
INFO: 2024/03/05 14:55:53.012767 Starting part 2 ...
INFO: 2024/03/05 14:55:53.020235 /global/pg_control
INFO: 2024/03/05 14:55:53.025917 Finished writing part 2.
INFO: 2024/03/05 14:55:53.025981 Calling pg_stop_backup()
INFO: 2024/03/05 14:55:53.073788 Starting part 3 ...
INFO: 2024/03/05 14:55:53.075182 backup_label
INFO: 2024/03/05 14:55:53.075239 tablespace_map
INFO: 2024/03/05 14:55:53.114629 Finished writing part 3.
INFO: 2024/03/05 14:55:53.175671 Querying pg_database
INFO: 2024/03/05 14:55:53.795401 Wrote backup with name base_000000020000000000000024 to storage default
postgres@slpc:~$ wal-g backup-list
INFO: 2024/03/05 14:57:04.216972 List backups from storages: [default]
backup_name                   modified             wal_file_name            storage_name
base_000000020000000000000024 2024-03-05T06:55:53Z 000000020000000000000024 default
```

然后就可以登录S3，可以是云厂商的对象存储，也可以自己用minio搭建对象存储。查看其中的备份以及归档文件。

### 日志归档
使用wal-g进行日志归档，并进行恢复。wal-g支持归档到S3，需要进行配置。

#### 归档
以pg15版本为例，配置如下：
```
archive_mode = on               # enables archiving; off, on, or always
archive_command = 'export WALG_FILE_PREFIX=/home/postgres/pgdata/archive; wal-g wal-push %p'
```
归档当指定目录，可以看到默认情况下是进行压缩了的，原有的16M文件被压缩为了2M。对归档日志进行压缩不但能够节约存储的成本，也能在涉及到I/O操作，网络传输等场景降低代价。
```shell
postgres@slpc:~/pgdata/archive$ ls
wal_005
postgres@slpc:~/pgdata/archive$ cd wal_005/
postgres@slpc:~/pgdata/archive/wal_005$ ls
000000010000000000000001.lz4
postgres@slpc:~/pgdata/archive/wal_005$ du -h
2.0M	.
```

#### 恢复

通过`wal-g wal-fetch`进行恢复，日志恢复有个功能，就是预取功能，可以在请求指定WAL日志前先预先fetch日志到`./.wal-g/prefetch`文件下，当缓存的日志文件比当前需要的WAL日志老时，就可以被删除了，以防止缓存日志膨胀。如果缓存的日志文件被`wal-fetch`请求，那么请求后日志从缓存中删除然后触发预取新的缓存日志。

示例：
```bash
restore_command = 'wal-g wal-fetch %f %p >> /home/postgres/pgdata/backup/wal-g.log 2>&1'
```
备机执行归档恢复日志如下：
```log
2024-03-04 11:20:17.306 CST [11149] DEBUG:  executing restore command "wal-g wal-fetch 000000010000000000000004 pg_wal/RECOVERYXLOG >> /home/postgres/pgdata/backup/wal-g.log 2>&1"
2024-03-04 11:20:17.475 CST [11149] LOG:  restored log file "000000010000000000000004" from archive
2024-03-04 11:20:17.514 CST [11149] DEBUG:  got WAL segment from archive
2024-03-04 11:20:17.514 CST [11149] DEBUG:  checkpoint record is at 0/4000060
```

### 压缩
压缩通过环境变量`WALG_COMPRESSION_METHOD`进行配置。支持如下压缩算法：
- lz4  默认压缩算法是lz4，压缩速度最快，但是压缩比较低
- lzma 压缩速度慢，但是压缩比是lz4的6倍
- zstd  压缩速度和压缩比较为均衡，速度比lz4略慢，但是压缩比比lz4高
- brotli  压缩速度和压缩比较为均衡

配置示例，以zstd压缩算法为例，配置环境变量`WALG_COMPRESSION_METHOD`
```shell
export WALG_COMPRESSION_METHOD=zstd
```
我们看一下对WAL日志归档文件的压缩，我们向表中插入一行数据，然后`select pg_switch_wal();`进行日志切换进行归档，16M数据被压缩为8k。
```shell
postgres@slpc:~/pgdata/archive$ ls
wal_005
postgres@slpc:~/pgdata/archive$ cd wal_005/
postgres@slpc:~/pgdata/archive/wal_005$ ls
000000010000000000000003.zst
postgres@slpc:~/pgdata/archive/wal_005$ du -h
8.0K	.
```

### 加密
略，详见 https://wal-g.readthedocs.io/

### 监控
略，详见https://wal-g.readthedocs.io/

### 其他命令
#### wal-show
可使用`wal-g wal-show`查看备份情况
```shell
postgres@slpc:~/pgdata/backup$ wal-g wal-show
+-----+------------+-----------------+--------------------------+--------------------------+---------------+----------------+--------+---------------+
| TLI | PARENT TLI | SWITCHPOINT LSN | START SEGMENT            | END SEGMENT              | SEGMENT RANGE | SEGMENTS COUNT | STATUS | BACKUPS COUNT |
+-----+------------+-----------------+--------------------------+--------------------------+---------------+----------------+--------+---------------+
|   1 |          0 |             0/0 | 000000010000000000000001 | 00000001000000000000000C |            12 |             12 | OK     |             1 |
|   2 |          1 |       0/D000000 | 00000002000000000000000D | 000000020000000000000011 |             5 |              5 | OK     |             1 |
+-----+------------+-----------------+--------------------------+--------------------------+---------------+----------------+--------+---------------+
```

#### wal-verify
`wal-g wal-verify integrity`用于验证WAL日志段文件是否完整，可执行PITR。
```shell
postgres@slpc:~/pgdata/backup$ wal-g wal-verify integrity
INFO: 2024/03/05 09:45:07.637578 Current WAL segment: 00000002000000000000000D
INFO: 2024/03/05 09:45:07.640460 Building check runner: integrity
INFO: 2024/03/05 09:45:07.641946 Detected earliest available backup: base_00000001000000000000000A
INFO: 2024/03/05 09:45:07.642015 Running the check: integrity
[wal-verify] integrity check status: OK
[wal-verify] integrity check details:
+-----+--------------------------+--------------------------+----------------+--------+
| TLI | START                    | END                      | SEGMENTS COUNT | STATUS |
+-----+--------------------------+--------------------------+----------------+--------+
|   1 | 00000001000000000000000A | 00000001000000000000000C |              3 |  FOUND |
+-----+--------------------------+--------------------------+----------------+--------+
```
`wal-g wal-verify timeline`用于检查timeline信息
```shell
postgres@slpc:~/pgdata/backup$ wal-g wal-verify timeline
INFO: 2024/03/05 10:01:01.236100 Current WAL segment: 00000002000000000000000E
INFO: 2024/03/05 10:01:01.239507 Building check runner: timeline
INFO: 2024/03/05 10:01:01.239586 Running the check: timeline
WARNING: 2024/03/05 10:01:01.239683 Could not parse the timeline Id from 000000010000000000000002.00000028.backup.zst. Skipping...
WARNING: 2024/03/05 10:01:01.239732 Could not parse the timeline Id from 000000010000000000000004.00000028.backup.zst. Skipping...
WARNING: 2024/03/05 10:01:01.239758 Could not parse the timeline Id from 00000001000000000000000A.00000028.backup.zst. Skipping...
[wal-verify] timeline check status: OK
[wal-verify] timeline check details:
Highest timeline found in storage: 2
Current cluster timeline: 2
```

#### backup-list
列出当前可用的备份
```bash
postgres@slpc:~/pgdata$ wal-g backup-push $PGDATA    # 新增备份
INFO: 2024/03/05 10:42:36.461264 Backup will be pushed to storage: default
INFO: 2024/03/05 10:42:36.526088 Calling pg_start_backup()
INFO: 2024/03/05 10:42:36.579452 Initializing the PG alive checker (interval=1m0s)...
INFO: 2024/03/05 10:42:36.579508 Starting a new tar bundle
INFO: 2024/03/05 10:42:36.579570 Walking ...
INFO: 2024/03/05 10:42:36.579922 Starting part 1 ...
INFO: 2024/03/05 10:42:36.804335 Packing ...
INFO: 2024/03/05 10:42:36.805626 Finished writing part 1.
INFO: 2024/03/05 10:42:36.805702 Starting part 2 ...
INFO: 2024/03/05 10:42:36.806202 /global/pg_control
INFO: 2024/03/05 10:42:36.818013 Finished writing part 2.
INFO: 2024/03/05 10:42:36.818063 Calling pg_stop_backup()
INFO: 2024/03/05 10:42:36.864683 Starting part 3 ...
INFO: 2024/03/05 10:42:36.865665 backup_label
INFO: 2024/03/05 10:42:36.865721 tablespace_map
INFO: 2024/03/05 10:42:36.868750 Finished writing part 3.
INFO: 2024/03/05 10:42:36.869898 Querying pg_database
INFO: 2024/03/05 10:42:36.963805 Wrote backup with name base_000000020000000000000011 to storage default
postgres@slpc:~/pgdata$ wal-g backup-list   # 查看当前可用的备份
INFO: 2024/03/05 10:42:49.146926 List backups from storages: [default]
backup_name                   modified                  wal_file_name            storage_name
base_00000001000000000000000A 2024-03-04T14:01:36+08:00 00000001000000000000000A default
base_000000020000000000000011 2024-03-05T10:42:36+08:00 000000020000000000000011 default    # 新增的备份
postgres@slpc:~/pgdata$ wal-g backup-list --detail
INFO: 2024/03/05 10:45:20.010794 List backups from storages: [default]
backup_name                   modified                  wal_file_name            storage_name start_time           finish_time          hostname data_dir                    pg_version start_lsn  finish_lsn is_permanent
base_00000001000000000000000A 2024-03-04T14:01:36+08:00 00000001000000000000000A default      2024-03-04T06:01:36Z 2024-03-04T06:01:36Z slpc     /home/postgres/pgdata/mn    150005     0/A000028  0/A000100  false
base_000000020000000000000011 2024-03-05T10:42:36+08:00 000000020000000000000011 default      2024-03-05T02:42:36Z 2024-03-05T02:42:36Z slpc     /home/postgres/pgdata/bakmn 150005     0/11000028 0/11000138 false
```

#### delete
用于删除归档日志，删除备份文件

比如我们仅保留一份备份，删除其他的备份，可执行`wal-g delete retain full 1`，结果如下
```shell
postgres@slpc:~/pgdata/backup$ wal-g delete retain full 1
INFO: 2024/03/05 10:51:42.833707 Backup to delete will be searched in storages: [default]
INFO: 2024/03/05 10:51:42.833892 retrieving permanent objects
INFO: 2024/03/05 10:51:42.834811 Start delete
INFO: 2024/03/05 10:51:42.835484 Objects in folder:
INFO: 2024/03/05 10:51:42.835596 	will be deleted: basebackups_005/base_00000001000000000000000A_backup_stop_sentinel.json, from storage: default
INFO: 2024/03/05 10:51:42.835608 	will be deleted: wal_005/000000010000000000000001.zst, from storage: default
INFO: 2024/03/05 10:51:42.835614 	will be deleted: wal_005/000000010000000000000002.00000028.backup.zst, from storage: default
INFO: 2024/03/05 10:51:42.835622 	will be deleted: wal_005/000000010000000000000002.zst, from storage: default
INFO: 2024/03/05 10:51:42.835633 	will be deleted: wal_005/000000010000000000000003.zst, from storage: default
INFO: 2024/03/05 10:51:42.835642 	will be deleted: wal_005/000000010000000000000004.00000028.backup.zst, from storage: default
INFO: 2024/03/05 10:51:42.835647 	will be deleted: wal_005/000000010000000000000004.zst, from storage: default
INFO: 2024/03/05 10:51:42.835651 	will be deleted: wal_005/000000010000000000000005.zst, from storage: default
INFO: 2024/03/05 10:51:42.835656 	will be deleted: wal_005/000000010000000000000006.zst, from storage: default
INFO: 2024/03/05 10:51:42.835661 	will be deleted: wal_005/000000010000000000000007.zst, from storage: default
INFO: 2024/03/05 10:51:42.835667 	will be deleted: wal_005/000000010000000000000008.zst, from storage: default
INFO: 2024/03/05 10:51:42.835671 	will be deleted: wal_005/000000010000000000000009.zst, from storage: default
INFO: 2024/03/05 10:51:42.835676 	will be deleted: wal_005/00000001000000000000000A.00000028.backup.zst, from storage: default
INFO: 2024/03/05 10:51:42.835681 	will be deleted: wal_005/00000001000000000000000A.zst, from storage: default
INFO: 2024/03/05 10:51:42.835702 	will be deleted: wal_005/00000001000000000000000B.zst, from storage: default
INFO: 2024/03/05 10:51:42.835709 	will be deleted: wal_005/00000001000000000000000C.zst, from storage: default
INFO: 2024/03/05 10:51:42.835715 	will be deleted: wal_005/00000002000000000000000D.zst, from storage: default
INFO: 2024/03/05 10:51:42.835727 	will be deleted: wal_005/00000002000000000000000E.zst, from storage: default
INFO: 2024/03/05 10:51:42.835736 	will be deleted: wal_005/00000002000000000000000F.zst, from storage: default
INFO: 2024/03/05 10:51:42.835741 	will be deleted: wal_005/000000020000000000000010.zst, from storage: default
INFO: 2024/03/05 10:51:42.835751 	will be deleted: basebackups_005/base_00000001000000000000000A/files_metadata.json, from storage: default
INFO: 2024/03/05 10:51:42.835798 	will be deleted: basebackups_005/base_00000001000000000000000A/metadata.json, from storage: default
INFO: 2024/03/05 10:51:42.835826 	will be deleted: basebackups_005/base_00000001000000000000000A/tar_partitions/part_001.tar.zst, from storage: default
INFO: 2024/03/05 10:51:42.835860 	will be deleted: basebackups_005/base_00000001000000000000000A/tar_partitions/part_003.tar.zst, from storage: default
INFO: 2024/03/05 10:51:42.835877 	will be deleted: basebackups_005/base_00000001000000000000000A/tar_partitions/pg_control.tar.zst, from storage: default
INFO: 2024/03/05 10:51:42.835907 Dry run, nothing were deleted
```
可以看到，并没有实际删除备份，这是因为删除备份，日志归档等文件是十分谨慎的，为了防止误操作以及用户确认删除的文件信息。
```shell
postgres@slpc:~/pgdata/backup$ wal-g backup-list
INFO: 2024/03/05 10:52:11.011245 List backups from storages: [default]
backup_name                   modified                  wal_file_name            storage_name
base_00000001000000000000000A 2024-03-04T14:01:36+08:00 00000001000000000000000A default
base_000000020000000000000011 2024-03-05T10:42:36+08:00 000000020000000000000011 default
```
我们需要执行`wal-g delete retain full 1 --confirm`才实际执行删除动作。
```shell
postgres@slpc:~/pgdata/backup$ wal-g delete retain full 1 --confirm
INFO: 2024/03/05 10:57:15.929301 Backup to delete will be searched in storages: [default]
INFO: 2024/03/05 10:57:15.929458 retrieving permanent objects
INFO: 2024/03/05 10:57:15.930468 Start delete
INFO: 2024/03/05 10:57:15.931124 Objects in folder:
INFO: 2024/03/05 10:57:15.931287 	will be deleted: basebackups_005/base_00000001000000000000000A_backup_stop_sentinel.json, from storage: default
INFO: 2024/03/05 10:57:15.931315 	will be deleted: wal_005/000000010000000000000001.zst, from storage: default
INFO: 2024/03/05 10:57:15.931336 	will be deleted: wal_005/000000010000000000000002.00000028.backup.zst, from storage: default
INFO: 2024/03/05 10:57:15.931354 	will be deleted: wal_005/000000010000000000000002.zst, from storage: default
INFO: 2024/03/05 10:57:15.931394 	will be deleted: wal_005/000000010000000000000003.zst, from storage: default
INFO: 2024/03/05 10:57:15.931438 	will be deleted: wal_005/000000010000000000000004.00000028.backup.zst, from storage: default
INFO: 2024/03/05 10:57:15.931452 	will be deleted: wal_005/000000010000000000000004.zst, from storage: default
INFO: 2024/03/05 10:57:15.931467 	will be deleted: wal_005/000000010000000000000005.zst, from storage: default
INFO: 2024/03/05 10:57:15.931476 	will be deleted: wal_005/000000010000000000000006.zst, from storage: default
INFO: 2024/03/05 10:57:15.931490 	will be deleted: wal_005/000000010000000000000007.zst, from storage: default
INFO: 2024/03/05 10:57:15.931527 	will be deleted: wal_005/000000010000000000000008.zst, from storage: default
INFO: 2024/03/05 10:57:15.931539 	will be deleted: wal_005/000000010000000000000009.zst, from storage: default
INFO: 2024/03/05 10:57:15.931550 	will be deleted: wal_005/00000001000000000000000A.00000028.backup.zst, from storage: default
INFO: 2024/03/05 10:57:15.931559 	will be deleted: wal_005/00000001000000000000000A.zst, from storage: default
INFO: 2024/03/05 10:57:15.931568 	will be deleted: wal_005/00000001000000000000000B.zst, from storage: default
INFO: 2024/03/05 10:57:15.931578 	will be deleted: wal_005/00000001000000000000000C.zst, from storage: default
INFO: 2024/03/05 10:57:15.931590 	will be deleted: wal_005/00000002000000000000000D.zst, from storage: default
INFO: 2024/03/05 10:57:15.931602 	will be deleted: wal_005/00000002000000000000000E.zst, from storage: default
INFO: 2024/03/05 10:57:15.931611 	will be deleted: wal_005/00000002000000000000000F.zst, from storage: default
INFO: 2024/03/05 10:57:15.931620 	will be deleted: wal_005/000000020000000000000010.zst, from storage: default
INFO: 2024/03/05 10:57:15.931642 	will be deleted: basebackups_005/base_00000001000000000000000A/files_metadata.json, from storage: default
INFO: 2024/03/05 10:57:15.931653 	will be deleted: basebackups_005/base_00000001000000000000000A/metadata.json, from storage: default
INFO: 2024/03/05 10:57:15.931677 	will be deleted: basebackups_005/base_00000001000000000000000A/tar_partitions/part_001.tar.zst, from storage: default
INFO: 2024/03/05 10:57:15.931690 	will be deleted: basebackups_005/base_00000001000000000000000A/tar_partitions/part_003.tar.zst, from storage: default
INFO: 2024/03/05 10:57:15.931703 	will be deleted: basebackups_005/base_00000001000000000000000A/tar_partitions/pg_control.tar.zst, from storage: default
postgres@slpc:~/pgdata/backup$ wal-g backup-list
INFO: 2024/03/05 10:57:19.467593 List backups from storages: [default]
backup_name                   modified                  wal_file_name            storage_name
base_000000020000000000000011 2024-03-05T10:42:36+08:00 000000020000000000000011 default
```
可以看到备份文件被删除掉。

参考资料：
- [wal-g github](https://github.com/wal-g/wal-g)
- [wal-g文档](https://wal-g.readthedocs.io/)
- [Continuous PostgreSQL Backups using WAL-G](https://supabase.com/blog/continuous-postgresql-backup-walg)
- [Pg 13 wal-g 本地目录备份与恢复](https://blog.51cto.com/heyiyi/2579126)