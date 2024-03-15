## PostgreSQL外部数据包装器
PostgreSQL实现了部分的SQL/MED规定，允许我们使用普通SQL查询来访问位于PostgreSQL之外的数据。这种数据被称为外部数据。

![fdw](https://www.interdb.jp/pg/pgsql04/fig-4-fdw-1.png)


SQL/MED是SQL语言中管理外部数据的一个扩展标准。MED:management of external data。它通过定义一个外部数据包装器和数据连接类型来管理外部数据。PostgreSQL提供对SQL/MED的支持，通过SQL/MED可以连接到各种异构数据库或其他PostgreSQL数据库。其相当于一套连接其他数据源的框架和标准。在SQL/MED标准中，实现了以下下四类数据库对象来访问外部数据源：
- foreign data wrapper：外部数据包装器，FDW。相当于定义外部数据驱动
- server：外部数据服务器，相当于定义一个外部数据源，需要制定外部数据源的FDW
- user mapping：用户映射，主要把外部数据源的用户映射到本地用户，用于控制权限

在SQL/MED中，远程服务器上的表被称为外部表。其中官方自带两个插件`file_fdw`、`postgres_fdw`。我们看一下他们的具体使用。

### 外部表的使用
#### file_fdw
使用外部表，首先是要安装插件。`postgresql.conf`配置文件中配置：
```shell
shared_preload_libraries = 'file_fdw,postgres_fdw'  # (change requires restart)
```
然后执行`create extension`：
```sql
postgres=# create extension file_fdw ;
CREATE EXTENSION
postgres=# create extension postgres_fdw ;
CREATE EXTENSION
```
安装插件后，查`pg_foreign_data_wrapper`系统表，存储外部数据包装器定义。外部数据包装器是一种访问位于外部服务器上数据的机制。安装了两个外部数据包装器，所以系统表中有两项内容，一条`file_fdw`，一条`postgres_fdw`。
```sql
postgres=# select * from pg_foreign_data_wrapper ;
-[ RECORD 1 ]+-------------
oid          | 16811         -- oid标识
fdwname      | file_fdw      -- 外部数据包装器的名字
fdwowner     | 10            -- 外部数据包装器的拥有者
fdwhandler   | 16809         -- 指一个负责为外部数据包装器提供执行例程的处理函数。如果没有提供处理函数则为0
fdwvalidator | 16810         -- 指一个负责检查传给外部数据包装器的选项的有效性的验证函数，包括外部服务器选项以及使用外部数据包装器的用户映射。如果没有提供验证函数则为0
fdwacl       |               -- 访问权限
fdwoptions   |               -- 外部数据包装器特定选项，以“keyword=value”字符串形式
-[ RECORD 2 ]+-------------
oid          | 16815
fdwname      | postgres_fdw
fdwowner     | 10
fdwhandler   | 16813
fdwvalidator | 16814
fdwacl       | 
fdwoptions   | 
```

创建一个外部服务器server。
```sql
postgres=# create server s1 foreign data wrapper file_fdw;
CREATE SERVER
```
创建server，实际是向`pg_foreign_server`插入一条数据。系统表`pg_foreign_server`存储外部服务器定义。外部服务器定义了外部数据的来源，例如一个远程服务器。外部服务器通过外部数据包装器来访问。
```sql
postgres=# select * from pg_foreign_server ;
-[ RECORD 1 ]-----
oid        | 16820      -- oid
srvname    | s1         -- 外部服务器的名字
srvowner   | 10         -- 外部服务器的拥有者 references pg_authid.oid)
srvfdw     | 16811      -- 此外部服务器的外部数据包装器的OID references pg_foreign_data_wrapper.oid
srvtype    |            -- 服务器的类型（可选）
srvversion |            -- 服务器的版本（可选）
srvacl     |            -- 访问权限
srvoptions |            -- 外部服务器特定选项，以“keyword=value”字符串形式
```

创建外部表，需要指定外部服务器的名字以及文件存储的位置和格式等信息。
```sql
postgres=# create foreign table foreign_t1(a int, b int) server s1 options(filename '/home/postgres/pgdata/t1.csv', format 'csv');
CREATE FOREIGN TABLE
```
建立外部表后查看`pg_foreign_table`系统表：
```sql
postgres=# select * from pg_foreign_table ;
-[ RECORD 1 ]-------------------------------------------------
ftrelid   | 16821               -- 外部表的pg_class项的OID     references pg_class.oid
ftserver  | 16820               -- 外部表所在的外部服务器的OID  references pg_foreign_server.oid
ftoptions | {filename=/home/postgres/pgdata/t1.csv,format=csv} -- 外部表选项，以“keyword=value”字符串形式
```
外部表数据示例：
```csv
2,2
1,1
```
查询外部表数据：
```sql
postgres=# select * from foreign_t1;
 a | b 
---+---
 2 | 2
 1 | 1
(2 rows)
```
查看外部表全表扫描执行计划：
```sql
postgres=# explain select * from foreign_t1;
                          QUERY PLAN                          
--------------------------------------------------------------
 Foreign Scan on foreign_t1  (cost=0.00..1.10 rows=1 width=8)
   Foreign File: /home/postgres/pgdata/t1.csv    -- 文件位置
   Foreign File Size: 8 b
(3 rows)
```
外部表只能读，不能写，执行写操作会报错。
```sql
postgres=# insert into foreign_t1 values(3,3);
ERROR:  cannot insert into foreign table "foreign_t1"
postgres=# update foreign_t1 set b = 0;
ERROR:  cannot update foreign table "foreign_t1"
```

参考文档：[file_fdw](http://www.postgres.cn/docs/14/file-fdw.html)

#### postgres_fdw
创建外部服务器，输入目标端数据库ip,端口和数据库名等信息
```sql
postgres=# create server foreign_pgserver foreign data wrapper postgres_fdw options(host '192.168.109.133',port '6432',dbname 'postgres');
CREATE SERVER
-- HINT:  Valid options in this context are: service, passfile, channel_binding, connect_timeout, dbname, host, hostaddr, port, options, application_name, keepalives, keepalives_idle, keepalives_interval, keepalives_count, tcp_user_timeout, sslmode, sslcompression, sslcert, sslkey, sslrootcert, sslcrl, sslcrldir, sslsni, requirepeer, ssl_min_protocol_version, ssl_max_protocol_version, gssencmode, krbsrvname, gsslib, target_session_attrs, use_remote_estimate, fdw_startup_cost, fdw_tuple_cost, extensions, updatable, truncatable, fetch_size, batch_size, async_capable, parallel_commit, keep_connections
```
查看系统表：
```sql
postgres=# select * from pg_foreign_server where srvname = 'foreign_pgserver';
-[ RECORD 1 ]------------------------------------------------
oid        | 16825
srvname    | foreign_pgserver
srvowner   | 10
srvfdw     | 16815
srvtype    | 
srvversion | 
srvacl     | 
srvoptions | {host=192.168.109.133,port=6432,dbname=postgres}
```
创建用户映射本地用户postgres和目标端用户名，密码。`CREATE USER MAPPING`定义一个用户到一个外部服务器的新映射。一个用户映射通常会包含连接信息，外部数据包装器会使用连接信息和外部服务器中包含的信息一起来访问一个外部数据源。
```sql
postgres=# create user mapping for postgres server foreign_pgserver options(user 'postgres',password '123456');
CREATE USER MAPPING
```
查看系统表`pg_user_mapping`，存储从本地用户到远程的映射。对这个系统表的访问对普通用户有限制，可使用视图`pg_user_mappings`替代。
```sql
postgres=# select * from pg_user_mapping;
-[ RECORD 1 ]------------------------------
oid       | 16832              
umuser    | 10                 -- 将要被映射的本地角色的OID，如果用户映射是公共的则为零, pg_authid.oid
umserver  | 16825              -- 包含此映射的外部服务器的OID
umoptions | {user=postgres,password=123456}   -- 用户映射特定的选项，以“keyword=value”字符串形式
```

创建外部表：
```sql
postgres=# create foreign table foreign_t2(a int, b int) server foreign_pgserver options(schema_name 'public', table_name 'ft2');
CREATE FOREIGN TABLE
```
需要注意的是，创建外部表，并不是在远端数据库服务中创建一个表，表必须在远端数据库中已存在，否则查询的时候会报错：
```sql
postgres=# select * from foreign_t2;
ERROR:  relation "public.ft2" does not exist
CONTEXT:  remote SQL command: SELECT a, b FROM public.ft2
```
在外部表的数据库中创建表ft2
```sql
postgres=# create table ft2(a int, b int);
CREATE TABLE
postgres=# insert into ft2 values(1,1);
INSERT 0 1
```
之后可正常查询外部表：
```sql
postgres=# select * from foreign_t2;
 a | b 
---+---
 1 | 1
(1 row)
```
与`file_fdw`不同，`postgres_fdw`是可以执行写操作的：
```sql
-- 向外部表中插入数据
postgres=# insert into foreign_t2 values(2,2);
INSERT 0 1
-- 查询
postgres=# select * from foreign_t2;
 a | b 
---+---
 1 | 1
 2 | 2
(2 rows)
-- 查看执行计划
postgres=# explain select * from foreign_t2;
                             QUERY PLAN                              
---------------------------------------------------------------------
 Foreign Scan on foreign_t2  (cost=100.00..186.80 rows=2560 width=8)
(1 row)
```


参考文档：
[SQL/MED](https://wiki.postgresql.org/wiki/SQL/MED)
[FOREIGN DATA WRAPPERS (FDW)](https://www.interdb.jp/pg/pgsql04.html)
[CREATE FOREIGN DATA WRAPPER](http://www.postgres.cn/docs/14/sql-createforeigndatawrapper.html)
[Foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)
[SqlMedConnectionManager](https://wiki.postgresql.org/wiki/SqlMedConnectionManager)
