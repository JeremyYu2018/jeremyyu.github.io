---
layout: post
title: PostgreSQL 11.6编译安装
date: 2019-12-31
categories: blog
tags: [PostgreSQL]
description: 
---

[TOC]
## Centos 7.4 上 PostgreSQL 11.6编译安装
@(PostgreSQL)

下面简单记录下在Centos7上面编译安装PostgreSQL 11.6。

###  第一步：安装依赖
```bash
[root@paas-1 ~]# yum groupinstall "Development tools"
[root@paas-1 ~]# yum install –y bison flex readline-devel zlib-devel
```
###  第二步：下载对应源代码

这里我选择11版本的最后一个小版本v11.6，下载之后并解压
```bash
[root@paas-1 ~]# pwd
/root
[root@paas-1 ~]# mkdir /opt/pg && cd /opt/pg
[root@paas-1 pg]# wget https://ftp.postgresql.org/pub/source/v11.6/postgresql-11.6.tar.gz
--2019-12-30 10:14:53--  https://ftp.postgresql.org/pub/source/v11.6/postgresql-11.6.tar.gz
Resolving ftp.postgresql.org (ftp.postgresql.org)... 72.32.157.246, 217.196.149.55, 87.238.57.227, ...
Connecting to ftp.postgresql.org (ftp.postgresql.org)|72.32.157.246|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 26001678 (25M) [application/x-tgz]
Saving to: ‘postgresql-11.6.tar.gz’

22% [=============================>                                                                                                           ] 5,873,664   71.5KB/s  eta 3m 45s
100%[========================================================================================================================================>] 26,001,678   124KB/s   in 4m 33s

2019-12-30 10:19:28 (92.9 KB/s) - ‘postgresql-11.6.tar.gz’ saved [26001678/26001678]

[root@paas-1 pg]#
[root@paas-1 pg]# tar xzf postgresql-11.6.tar.gz
```
解压完成。

### 第三步：创建postgres用户和组
如果用户已经存在，请略过。
```bash
# groupadd  postgres
# useradd   postgres
```

### 第四步：编译安装

1. 配置, 这里`--prefix`参数指定的地方是编译好的postgres的二进制存放目录，该目录不需要提前创建。
```bash
[root@paas-1 pg]# pwd
/opt/pg
[root@paas-1 pg]# cd postgresql-11.6
[root@paas-1 postgresql-11.6]#
[root@paas-1 postgresql-11.6]# ./configure --prefix=/db/pgsql
checking build system type... x86_64-pc-linux-gnu
############省略很多内容########
config.status: linking src/include/port/linux.h to src/include/pg_config_os.h
config.status: linking src/makefiles/Makefile.linux to src/Makefile.port
```
2. 编译
```bash
[root@paas-1 postgresql-11.6]# pwd
/opt/pg/postgresql-11.6
[root@paas-1 postgresql-11.6]# make
make -C ./src/backend generated-headers
make[1]: Entering directory `/opt/pg/postgresql-11.6/src/backend'
############省略很多内容########
make[1]: Leaving directory `/opt/pg/postgresql-11.6/config'
All of PostgreSQL successfully made. Ready to install.
```
编译完成

3. 安装
```bash
[root@paas-1 postgresql-11.6]# make install
############省略很多内容########
PostgreSQL installation complete.
```

4. 查看安装情况， 可以发现/db/pgsql/目录生成了4个目录，同时将/db整个目录owner和group都修改为postgres
```bash
[root@paas-1 postgresql-11.6]# ls /db/pgsql/ -l
total 16
drwxr-xr-x 2 root root 4096 Dec 30 10:35 bin
drwxr-xr-x 6 root root 4096 Dec 30 10:35 include
drwxr-xr-x 4 root root 4096 Dec 30 10:35 lib
drwxr-xr-x 6 root root 4096 Dec 30 10:35 share
[root@paas-1 postgresql-11.6]# chown -R postgres:postgres /db
```

### 第五步：配置用户环境变量

切换到postgres用户，根据自己的情况修改一下环境变量, 并使用source重新加载新环境变量
```bash
[root@paas-1 ~]# su  - postgres
-bash-4.2$ cat .bash_profile
[ -f /etc/profile ] && source /etc/profile
# If you want to customize your settings,
# Use the file below. This is not overridden
# by the RPMS.
[ -f /var/lib/pgsql/.pgsql_profile ] && source /var/lib/pgsql/.pgsql_profile
export PGHOME=/db/pgsql
export PGDATA=/db/dbc1
export LD_LIBRARY_PATH=$PGHOME/lib
export PATH=$PGHOME/bin:$PATH
-bash-4.2$ source .bash_profile
```

### 第六步：初始化数据库簇(database cluster)

1. 初始化，`/db/dbc1`目录不需要自己创建, 初始化时会自动创建，如果PGDATA目录已经存在，且目录不为空(包括存在隐藏文件)，initdb会报错`initdb: directory "/db/dbc1" exists but is not empty`

```bash
-bash-4.2$ initdb -D /db/dbc1 -E utf8
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory /db/dbc1 ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default timezone ... PRC
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    pg_ctl -D /db/dbc1 -l logfile start
```
初始化成功。

2. 查看安装效果
```bash
-bash-4.2$ ll /db/dbc1/
total 48
drwx------ 5 postgres postgres    41 Dec 30 10:44 base
drwx------ 2 postgres postgres  4096 Dec 30 10:44 global
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_commit_ts
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_dynshmem
-rw------- 1 postgres postgres  4513 Dec 30 10:44 pg_hba.conf
-rw------- 1 postgres postgres  1636 Dec 30 10:44 pg_ident.conf
drwx------ 4 postgres postgres    68 Dec 30 10:44 pg_logical
drwx------ 4 postgres postgres    36 Dec 30 10:44 pg_multixact
drwx------ 2 postgres postgres    18 Dec 30 10:44 pg_notify
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_replslot
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_serial
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_snapshots
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_stat
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_stat_tmp
drwx------ 2 postgres postgres    18 Dec 30 10:44 pg_subtrans
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_tblspc
drwx------ 2 postgres postgres     6 Dec 30 10:44 pg_twophase
-rw------- 1 postgres postgres     3 Dec 30 10:44 PG_VERSION
drwx------ 3 postgres postgres    60 Dec 30 10:44 pg_wal
drwx------ 2 postgres postgres    18 Dec 30 10:44 pg_xact
-rw------- 1 postgres postgres    88 Dec 30 10:44 postgresql.auto.conf
-rw------- 1 postgres postgres 23990 Dec 30 10:44 postgresql.conf
```

### 第七步：配置监听和端口，修改postgresql.conf

这里我将port修改为5433，不使用默认的5432.
```bash
-bash-4.2$ grep -E "listen_addresses|port" postgresql.conf
listen_addresses = '*'          # what IP address(es) to listen on;
port = 5433                             # (change requires restart)
```

### 第八步： 启停pg、查看状态、reload。
```bash
-bash-4.2$ pg_ctl start #启动postgres
waiting for server to start....2019-12-30 10:58:51.942 CST [7819] LOG:  listening on IPv4 address "0.0.0.0", port 5433
2019-12-30 10:58:51.942 CST [7819] LOG:  listening on IPv6 address "::", port 5433
2019-12-30 10:58:51.944 CST [7819] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5433"
2019-12-30 10:58:51.958 CST [7820] LOG:  database system was shut down at 2019-12-30 10:58:34 CST
2019-12-30 10:58:51.960 CST [7819] LOG:  database system is ready to accept connections
 done
server started
-bash-4.2$
-bash-4.2$ pg_ctl reload   #重载配置
server signaled
2019-12-30 10:58:56.170 CST [7819] LOG:  received SIGHUP, reloading configuration files
-bash-4.2$
-bash-4.2$ pg_ctl status
pg_ctl: server is running (PID: 7819)
/db/pgsql/bin/postgres
-bash-4.2$
-bash-4.2$ pg_ctl stop     #停止postgres进程
waiting for server to shut down....2019-12-30 10:59:03.957 CST [7819] LOG:  received fast shutdown request
2019-12-30 10:59:03.958 CST [7819] LOG:  aborting any active transactions
2019-12-30 10:59:03.959 CST [7819] LOG:  background worker "logical replication launcher" (PID 7826) exited with exit code 1
2019-12-30 10:59:03.959 CST [7821] LOG:  shutting down
2019-12-30 10:59:03.967 CST [7819] LOG:  database system is shut down
 done
server stopped
```

使用psql登录试试，由于上面配置不是使用默认的5432端口，这里需要使用-p指定5433端口。
```bash
-bash-4.2$ psql -p5433 -l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

-bash-4.2$
```
启动成功。


### 其他： 自定义configure参数
```bash
./configure --help
重要选项
– --prefix=PREFIX  -- 安装路径
– --with-blocksize=BLOCKSIZE  -- 数据库blocksize，缺省8KB
– --with-segsize=SEGSIZE   -- 表文件的段尺寸，缺省1GB
– --with-llvm   -- 使用基于JIT的llvm编译
```

有如下几点说明：
1. `segment_size`和`block_size`的`context`为`internal`，表明此参数只能在编译时定制，之后不可修改,使用`initdb --help`也没法看到可做自定义配置的选项。
2. `wal_segment_size`只能在初始化数据库时指定， `--wal-segsize=SIZE`。
3. 可以配置较大的数据库在提升I/O性能，在该版本(11.6)中`--with-wal-blocksize`和` --with-blocksize=`的可选参数为`1,2,4,8,16,32`。

```bash
./configure --prefix=/db/pgsql  --with-wal-blocksize=32   --with-blocksize=32
```


```bash
postgres=# select name,context,setting,unit from pg_settings where name in ('block_size', 'segment_size');
     name     | context  | setting | unit
--------------+----------+---------+------
 block_size   | internal | 8192    |
 segment_size | internal | 131072  | 8kB
(2 rows)
```

如下为对应表：
| ./configure参数 |  pg内部参数  |  备注|
| -----| -----| ------| 
| --with-segsize | segment_size |  表文件的段尺寸，缺省1GB |
| --with-blocksize | block_size | 数据库blocksize，缺省8KB |


### 小结
    
  pg的编译安装真的是超级简单的，默认配置选项基本能满足大部分需求，实际情况有特殊需求也可以很方便的自定义。