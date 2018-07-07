---
layout: post
title: xtrabackup备份总结
date: 2018-06-29
categories: blog
tags: [percona,xtrabackup,备份与恢复]
description: 
---

 这几周一直在研究备份与恢复的工具，重点关注在percona公司的xtrabackup热备工具，不得不说percona公司真是一家伟大的公司，提供众多非常好用的工具。pmm,percona-server,percona-xtrabackup,percona-toolkits等，每一样都是得到了业界的认可，被广泛应用于生产环境中，而今天说的percona-xtrabackup是众多优秀的之一，备份还原利器。
 ![](https://i.imgur.com/zMRNsyt.png)   
    
 xtrabackup具有如下特点： 

 1. Create hot InnoDB backups without pausing your database
 2. Make incremental backups of MySQL
 3. Stream compressed MySQL backups to another server
 4. Move tables between MySQL servers on-line
 5. Create new MySQL replication slaves easily
 6. Backup MySQL without adding load to the server
 
尽管拥有众多的卓越的特性。但是我们日常备份的场景大多是全备。如果是增量备份通常备份的是binlog，xtrabackup备份工具在早期的时候主要是两个工具：innobackupex和xtrabackup,但是在如今最新的2.4中已经innobackupex已经变成一个指向xtrabackup的软链接。另外官方推荐使用xtrabackup命令行，如今保留innobackupex的目的只是为了兼容性
![](https://i.imgur.com/pnV8yDl.png)

## 备份 ##
    
1. innobackupex方式

	`innobackupex --no-timestamps /data/backup/192.168.1.1_3306_full_20180627`

指定--no-times目的是使用自己的创建目录作为备份的数据存储的目录，默认是创建一个当前时间的目录。个人习惯备份目录的全名命名为`ip`+`port`+`full`+`20180629`

2. xtrabackup方式

	`xtrabackup --backup --target-dir=/data/backup/192.168.1.1_3306_full_20180627`

## 还原 ##
还原包括两个步骤：Prepare和copy-back,Prepare过程非常类似于mysqld crash之后做一次mysql recover,原理就是重新应用变化期间生成redo 日志。

1. 应用日志（Prepare阶段）

innobackupex方式

	`innobackupex --apply-log /data/backup/192.168.1.1_3306_full_20180627`

xtrabackup方式

  	`xtrabackup --prepare --target-dir=/data/backup/192.168.1.1_3306_full_20180627`

2. copy-back

innobackupex方式

	`innobackup --copy-backup /data/backup/192.168.1.1_3306_full_20180627`

xtrabackup方式

    `xtrabackup --copy-back --target-dir=/data/backups/192.168.1.1_3306_full_20180627`

注：更参数参考官方文档
甚至可以简单地使用OS命令cp所有数据到datadir。
 `cp -a /data/backup/192.168.1.1_3306_full_20180627 /data/mysql3306/data`
最后更改权限`mysql`:`mysql`启动即可,最后建议所有操作的时候都使用`--defaults-file`参数指定配置文件,个人曾经经历过一次不指定配置文件而备份失败的情况。原因是操作过程中无意使用`/etc/my.cnf`文件从而进行了错误的备份或还原操作，`/etc/my.cnf`文件是OS安装完成之后就默认的，建议清除。

备份生成的文件讲解：

1. backup-my.cnf
1. xtrabackup_binlog_info
2. xtrabackup_checkpoints
3. xtrabackup_info
4. xtrabackup_logfile


xtrabackup_info文件里面包括备份集的信息
>
    uuid = 0d586fde-7c73-11e8-a890-525400486aa0 
    name =  
    tool_name = xtrabackup  
    tool_command = -uroot -pqwer1234 -S /tmp/mysql3307.sock --backup --target-dir=/data/backup/192.168.1.1_3306_full_20180627 
    tool_version = 2.4.10  
    ibbackup_version = 2.4.10  
    server_version = 5.7.21-log  
    start_time = 2018-06-30 22:36:51  
    end_time = 2018-06-30 22:37:02 
    lock_time = 0  
    binlog_pos = filename 'mysql-bin.000003', position '1291', GTID of the last change '0dfc1ecd-6bf0-11e8-b64a-525400486aa0:1-6,
    fee9c8e2-6bef-11e8-b310-525400486aa0:1-2128'  
    innodb_from_lsn = 0 
    innodb_to_lsn = 3839877 
    partial = N  
    incremental = N  
    format = file 
    compact = N 
    compressed = N  
    encrypted = N 

bakup-my.cnf文保存了server的常规信息，如`uuid`,版本，备份起止时间，起始lsn和终点lsn,全备还是部分备份，有时候我们会有备份单库单表的需求。是否增量备份。格式是`file`还是tar,是否压缩或者加密等。

xtrabackup_checkpoints文件的内容如下
>
    backup_type = full-prepared  
    from_lsn = 0  
    to_lsn = 3839877  
    last_lsn = 3839886  
    compact = 0  
    recover_binlog_info = 0 
>

重点关注`from_lsn`和`to_lsn`,和`xtrabackup_info`中两个参数`innodb_from_lsn`和`innodb_to_lsn`对比，两参数一一致。那`xtrabackup_checkpoints`中`last_lsn`又代表什么意思呢？之所以会有`last_lsn`，`to_lsn`到`last_lsn`代表备份期间生成增量lsn,如果在一个没有任何dml的测试机上发现两个值是不一样，说明`xtrabackup`在备份过程中也会产生一个生成一个redo。

`xtrabackup_binlog_info`文件如下,这里带有两个GTID的原因是该库曾经作为`slave`执行了2128个事务，之后又在本库执行了6个事务，结合上图可以得到之前主库上的uuid为`fee9c8e2-6bef-11e8-b310-525400486aa0`
>
    mysql-bin.000003	1291  0dfc1ecd-6bf0-11e8-b64a-525400486aa0:1-6,
    fee9c8e2-6bef-11e8-b310-525400486aa0:1-2128
>

Prepare之后的文件，除了上面提到5个文件处，还多了`ibtmp1`,`redo log file`,`xtrabackup_binlog_pos_innodb`。
`xtrabackup_binlog_pos_innodb`文件内容如下。
>
mysql-bin.000003        1291
>


## 最后 ##
最后再以一个常问问题结尾。`xtrabackup`备份过程中会阻塞dml么？答案是会的，因为在备份过程中会产生全局读锁，用于拷贝非Innodb表，同时记录binlog位点，如果非事务表很多的话，加锁时间就会增长，相应阻塞时间就会增长，以图为例讲述。

![](https://i.imgur.com/pHYMYF2.png)

过程总共7步。

1.  启动一个redo拷贝线程，此线程一直拷贝redo变化。
2.  开始拷贝`ibd`,共享表空间`ibdata1`等事务表数据。
3.  开启一个`FTWRL`全局读锁，此后的时间用来拷贝非`Innodb`表（Non-Innodb），这段时间数据库是只读的。
4.  开始拷贝所有的非事务表。
5.  获得binlog position
6.  解锁,`unlock tables`释放全局读锁。
7.  停止redo拷贝线程，备份期间生成的变化的`redo log`保存在`xtabackup_logfile`

开源数据库领域备份与还原工具中，percona-xtrabackup是一枝独秀，经历了无数实际考验，与官方单线程逻辑备份工具mysqldump配合能有效地完成日常大部分备份还原任务，这里面还有很多细节可以挖掘。如需要了解内部更深入的实现原理，可能需要花更多时间去积极探索。xtrabackup针对不同版本有着不同的备份策略，针对MySQL在最后阶段会关闭所有表，执行一个FTWRL, 为了更一步减少锁表时间，新版本还增加了`lock tables for backup`和`lock binary log for backup`,不过这条命令对目前MySQL是不支持，目前只支持percona server。这有待我更一步探索原理。