---
layout: post
title: PostgreSQL基础知识总结
date: 2019-12-22
categories: blog
tags: [PostgreSQL]
description: 
---

 最近学习了炼数成金黄晓涛老师的pg课[《PostgreSQL初识与提高（第四期)》](http://www.dataguru.cn/mycourse.php?lessonid=2130)，现在总结一下相关知识，以下大部分内容整理来自该课的课件，少量内容经过自己修改加工过。
 
 ### PostgreSQL的体系结构
 
![PostgreSQL体系结构](![PostgreSQL体系结构](https://s2.ax1x.com/2019/12/23/lpTcXF.png)

### postmaster进程
postmaster是守护进程，实际上是第一个postgres进程,在pg10中，postmaster实际上是一个指向postgres的软连接
```bash
-bash-4.2$ file /usr/pgsql-10/bin/postmaster
/usr/pgsql-10/bin/postmaster: symbolic link to `postgres'
```

主要职责有：
1. 数据库的启停
2. 监听客户端的连接
3. 为每一个客户端衍生（fork）新的postgres服务进程
4. 当postgres进程出错时尝试修复
5. 管理数据文件
6. 管理数据库的辅助进程

### postgres进程
postgres进程实际上是postgres子进程，注意区分，无数服务进程都是由一个postgres父进程fork出来的。
```bash
-bash-4.2$ ps -ef | grep postgres
postgres  8071     1  0 Dec20 ?        00:00:04 /usr/pgsql-10/bin/postgres -D .
postgres  8072  8071  0 Dec20 ?        00:00:00 postgres: logger process
postgres  8074  8071  0 Dec20 ?        00:00:00 postgres: checkpointer process
postgres  8075  8071  0 Dec20 ?        00:00:04 postgres: writer process
postgres  8076  8071  0 Dec20 ?        00:00:04 postgres: wal writer process
postgres  8077  8071  0 Dec20 ?        00:00:03 postgres: autovacuum launcher process
postgres  8078  8071  0 Dec20 ?        00:00:00 postgres: archiver process
postgres  8079  8071  0 Dec20 ?        00:00:05 postgres: stats collector process
postgres  8080  8071  0 Dec20 ?        00:00:00 postgres: bgworker: logical replication launcher
root     13816 13784  0 14:34 pts/1    00:00:00 su - postgres
postgres 13817 13816  0 14:34 pts/1    00:00:00 -bash
postgres 13850 13817  0 14:34 pts/1    00:00:00 psql
postgres 13851  8071  0 14:34 ?        00:00:00 postgres: postgres postgres [local] idle
postgres 14410 13817  0 14:40 pts/1    00:00:00 psql
postgres 14411  8071  0 14:40 ?        00:00:00 postgres: postgres postgres [local] idle
postgres 14419 13817  0 14:40 pts/1    00:00:00 ps -ef
postgres 14420 13817  0 14:40 pts/1    00:00:00 grep --color=auto postgres
```
使用ps查看可以发现所有pg实例的进程pid继承情况，`13851`和`14411`的的父进程都是`8071`，这两个进程都是`8071`的进程postgres fork出来的子进程，同时可以观察到所有pg相关进程的父进程的PID都是`8071`，主要有如下职责：
1. 直接与客户端进程通讯
2. 负责接收客户端所有的请求
3. 包含数据库引擎，负责解析SQL和生成执行计划等。
4. 负责返回命令执行结果给客户端
5. 在客户端断开连接是释放进程

![PostgreSQL client postgres](https://s2.ax1x.com/2019/12/23/lpTR0J.png)

### 本地内存
本地内存是服务器进程独占的内存结构，每个postgres子进程都会分配一小块相应内存空间，随着连接会话的增加而增加，这部分内存单独属于每一个连接，它不属于实例的一部分。
主要的内存区域有：
1. work_mem: 用于排序
2. maintenance_work_mem：用于内部运维工作的内存，如VACUUM垃圾回收、创建和重建索引等
3. temp_buffers: 用于存储临时表的数据

### 数据库实例
- 实例是管理和访问数据库的一套方法，即所有客户端的请求最终都会转换为实例对数据库磁盘文件的各种操作，从而达到读写数据的目的。
- 实例包含若干个内存结构和进程。

### 共享内存
或叫全局内存，主要包括如下最大的三个内存区域。
1. Shared Buffer:
    - 用户缓存表和索引的数据块
    - 数据的读写都是直接对Buffer操作的，若所需的块不在Buffer中，则需要从磁盘中读取到Buffer中
    - 在Buffer中被修改过的，但又没有写到磁盘文件中的块被称为脏块。
    - 由shared_buffers参数控制大小。

2. WAL（Write Ahead Log）Buffer:
    - 预写日志缓存用于缓存增删该写操作产生的事务日志
    - 由wal_buffers参数控制大小

3. Clog Buffer:
    - Commit Log Buffer是记录事务状态的日志缓存

### 辅助进程
1. writer process：
    - 将shared buffer中的脏数据也写到磁盘文件中
    - 使用LRU算法清理脏页
    - 平时多在休眠，被激活时工作
2. Autovacuum launcher/workers:
    - 自动清理垃圾回收进程
    - 当参数autovacuum为on的时候启用自动清理功能
    - Launcher为清理的守护进程，每次启动的时候回调用一个或多个worker
    - worker是负责真正清理工作的进程，由autovacuum_max_workers参数设定其数量
3. wal writer process
    - 将预写日志写入磁盘文件
    - 触发时机：
        - Wal Buffer满了
        - 事务commit时
        - WAL writer进程达到简写时间时
        - checkpoint发生时
4.  checkpointer process：
    - 用于保证数据库的一致性
    - 触发后台`writer process`和`wal writer process`动作
    - 拥有多个参数控制其启动的间隔
5. logger process
    - 采集PostgreSQL的运行状态，并将运行日志写入日志文件
    - loging_collector参数为on时启动，不建议关闭
    - log_directory设定日志目录
    - log_destination设定日志输出方式，甚至格式
    - log_filename 设定日志文件名
    - log_truncate_on_rotation设定是否重复循环使用并删除日志
    - log_rotation_age设定循环时间
    - log_rotation_size设定循环的日志尺寸上限
6. archiver process
    - 用于将写满的wal日志文件转移到归档目录，该进程只有在归档模式下才会启用
7. stats collector process
    - 统计信息的收集进程。收集表和索引的空间信息和元祖信息等，甚至表的访问信息，收集到的信息可以个优化器、autovacuum使用。

 ### 数据库
 - PostgreSQL在磁盘上的一整套文件集合叫做database cluster
 - 数据库包含了数据文件、日志文件等多种文件，用于存储用户数据和保证数据一致性。

 ![PostgreSQL database cluster](https://s2.ax1x.com/2019/12/23/lpT2m4.png)


### PostgreSQL目录结构

| Item	 | Description |
| --------| ----------| 
| PG_VERSION | 	A file containing the major version number of PostgreSQL |
base	 |   Subdirectory containing per-database subdirectories
current_logfiles    |	 File recording the log file(s) currently written to by the logging collector
global  |	Subdirectory containing cluster-wide tables, such as pg_database
pg_commit_ts	| Subdirectory containing transaction commit timestamp data
pg_dynshmem	|  Subdirectory containing files used by the dynamic shared memory subsystem
pg_logical  |	Subdirectory containing status data for logical decoding
pg_multixact   |	Subdirectory containing multitransaction status data (used for shared row locks)
pg_notify	|    Subdirectory containing LISTEN/NOTIFY status data
pg_replslot |	Subdirectory containing replication slot data
pg_serial	 |  Subdirectory containing information about committed serializable transactions
pg_snapshots | 	Subdirectory containing exported snapshots
pg_stat	   |  Subdirectory containing permanent files for the statistics subsystem
pg_stat_tmp   |	Subdirectory containing temporary files for the statistics subsystem
pg_subtrans	 |  Subdirectory containing subtransaction status data
pg_tblspc	|   Subdirectory containing symbolic links to tablespaces
pg_twophase	|  Subdirectory containing state files for prepared transactions
pg_wal	  |    Subdirectory containing WAL (Write Ahead Log) files
pg_xact	|   Subdirectory containing transaction commit status data
postgresql.auto.conf	| A file used for storing configuration parameters that are set by ALTER SYSTEM
postmaster.opts	|   A file recording the command-line options the server was last started with
postmaster.pid  | 	A lock file recording the current postmaster process ID (PID), cluster data directory path, postmaster start timestamp, port number, Unix-domain socket directory path (empty on Windows), first valid listen_address (IP address or *, or empty if not listening on TCP), and shared memory segment ID (this file is not present after server shutdown)     

摘录自[PostgreSQL10 storage-file-layout](https://www.postgresql.org/docs/10/storage-file-layout.html)

1. `$PGDATA/global/pg_control` : 存储全局控制信息。
2. `$PGDATA/global/pg_filenode.map`: 用于将当前目录下系统表的OID和具体文件名进行硬编码映射（每个用户创建的数据库目录下也有同名文件）。
3. `$PGDATA/global/pg_internal.init`:用于缓存系统表，加快系统表读取速度（每个用户创建的数据库目录下也有同名文件）。


### base目录
```bash
- $PGDATA/base
   - 1
   - 13805        #数据库OID,
   - 13806
        - 112     #每个数据库下面的数据文件，以对象的OID命名。
        - 113   
        ......
        - 3764       
        - 3764_fsm  #数据文件对应的FSM（free space map）文件，用位图的方式来标识哪些block是空闲的。 
        - 3764_vm   #数据文件对应的VM（visibility map）文件，在做多版本并发控制时时通过在元组头上标识“已无效”来实现删除或更新的，最后通过VACUUM功能来清理无效数据进而回收空闲空间。
        - pg_filenode.map
        - pg_internal.init
        - PG_VERSION
```

### 表空间目录
表空间目录可以自由制定，结构与base目录类似
```bash
- $tablespace_location   #表空间跟目录
    -  PG_10_201707211   
        - 16386          # 数据库OID
            - 1247       # 数据对象OID
            - 1247_fsm
            - 1247_vm
            ....
            - pg_filenode.map
            - PG_VERSION
```

后面继续。