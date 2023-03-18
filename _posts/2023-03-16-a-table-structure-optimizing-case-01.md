---
layout: post
title: 一则表结构优化案例（一）
date: 2023-03-16
categories: blog
tags: [MySQL, Oracle, 表结构优化]
description: 
---

据我了解，大部分公司DBA都只负责数据库的运维架构工作，尤其是MySQL DBA, 通常一个合格的DBA不止要会运维工作，而且要关心和数据库的方方面面，在工作中要站在数据库的角度来看待整体的性能问题，想法设法减少数据库的负载，实际情况下数据库很容易成为性能的瓶颈点，通常情况下研发人员只站在业务设计的角度来考虑问题，专注功能实现，往往会忽视了一些潜在的表性能问题，面对研发提过来的DDL工单，一个合格的DBA要关注如下：
   1. 表的使用场景: 尤其是高并发场景,比如用户主动点击按钮才会查询/更新/插入还是系统主动推送给用户才会使用，是登录APP时查询还是退出时使用，不同的场景表的DML频率完全不同。
   2. 表的类别：日志表？是否只是用来纯INSERT，不会查询和更新，表如果要查询，查询条件是什么？
   3. 表记录数未来是否会无线膨胀？尤其是MySQL数据库。
   4. 表是否有主键？查询列或者更新列是否都建上了索引。
   5. 等等

### 优化原则一：避免在OLTP数据库中跑OLAP语句

案例：
    研发给了一签到模型图还有一张表结构
<center ><img src="https://i.imgur.com/PAusIWh.png"  style="zoom: 40%;" /> </center>

```bash
CREATE TABLE user_signin_records
(
`id` int NOT NULL AUTO_INCREMENT COMMENT '自增id' ,
`user_id` bigint NOT NULL DEFAULT 0 COMMENT '账号id' ,
`sign_time` datetime NOT NULL DEFAULT Now() COMMENT '签到日期' ,
PRIMARY KEY (`id`)
)
DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT '签到记录数据';
```
希望能实现如下功能点：
1. 用户通过点击"上月"“下月”点击翻页可以实现查看历史签到的日期。
2. 最上方可以看到历史总签到天数，不需要判断连续签到。

研发想法：第二点只需要一天一条记录，每天INSERT一天记录即可。第二个功能点：用如下SQL就能解决。

```sql
select count(*) from user_signin_records where user_id=1000;
```

存在的问题：
1. 表上USER_ID列上没有索引,如果根据USER_ID来查询，将会是全表扫描。
2. 单表记录数可能会膨胀,如果每天有几百万人签到，那单表记录数会膨胀得非常快。
3. 单个玩家签到记录数很多的时候会导致每次COUNT都很慢，严重影响性能，避免在OLTP数据库跑OLAP语句。

对应解决办法：
1. 给（USER_ID,SIGN_TIME）加上联合索引，因为“翻月”查询的时候的查询条件是（USER_ID，SIGN_TIME)这时候能尽可能用上联合索引。
2. 将表转为分区表,按月做range分区，通过自动化工具来控制表自动增删分区，并且从业务侧上限制用户只能查询最近半年的的签到记录，实际情况下用户大部分情况下不可能查询三个月之前的签到记录。
```bash
select user_id,
	     sign_time 
       from user_signin_records
	where user_id = 1000 
	and sign_time >= DATE_ADD(DATE_ADD(LAST_DAY(curdate()), INTERVAL 1 DAY), INTERVAL -1 MONTH)
        and sign_time < DATE_ADD(LAST_DAY(curdate()), INTERVAL 1 DAY);
```
3. 在别的用户主表（比如USER_INFO）增加字段来额外记录用户总签到天数，并且"签到（INSERT）"和“增加签到记录数（UPDATE）"应该看做一个事务，保证签到操作的原子性。

```SQL
alter table user_info add sign_days int not null default 0 comment '用户总签到天数';
```

- 业务逻辑实现SQL
```SQL
BEGIN;

INSERT INTO user_signin_records(USER_ID, sign_time)
VALUES(1000, now());

UPDATE user_info
SET sign_days = sign_days +1
WHERE user_id=1000;

COMMIT;
```
因此，最终的表结构如下：
> 由于使用了分区表，需要严格限制每一个SQL都必须带上分区建（sign_time）
```bash
CREATE TABLE `user_signin_records` (
  `id` int NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `user_id` bigint NOT NULL DEFAULT '0' COMMENT '账号id',
  `sign_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '签到日期',
  PRIMARY KEY (`id`,`sign_time`),
  KEY `IDX_USER_ID` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='签到记录数据'
/*!50500 PARTITION BY RANGE  COLUMNS(sign_time)
(PARTITION P20230301 VALUES LESS THAN ('2023-03-01') ENGINE = InnoDB,
 PARTITION P20230401 VALUES LESS THAN ('2023-04-01') ENGINE = InnoDB) */
```


改进：某些情况分区表维护起来略显麻烦，因此还是要想法设法建设单表记录。

1. 签到表每个月一个字段来记录，一个月最多31天
2. log_date每天签到时刷新，每月轮换时重新插入新一个月的记录
3. buffxxx里面0表示未签到，1表示签到。

```SQL
CREATE TABLE `user_signin_records` (
  `id` int NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `user_id` bigint NOT NULL DEFAULT '0' COMMENT '账号id',
  `log_date` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '签到日期',
  `buff01` tinyint NOT NULL DEFAULT 0  COMMENT '1号是否签到',
  `buff02` tinyint NOT NULL DEFAULT 0  COMMENT '2号是否签到',
   。。。依次类推。。。
  `buff29` tinyint NOT NULL DEFAULT 0  COMMENT '29号是否签到',
  `buff30` tinyint NOT NULL DEFAULT 0  COMMENT '30号是否签到',
  `buff31` tinyint NOT NULL DEFAULT 0  COMMENT '31号是否签到',
  PRIMARY KEY (`id`),
  KEY `IDX_USER_ID` (`user_id`, `log_date`)
)
```
> 改进版本表结构通过类似行转列的方式将能大大减少单表记录数。

在应用开发中，数据库很容易成为系统的瓶颈，站在整体的角度上来看，数据库不方便扩展，但是应用却容易扩展，因此多多关注容易成为性能瓶颈的点是有必要的，通过合理设计表结构能提前避免掉很多麻烦，使得系统在面对高并发时更从容一些。

上面的仅仅是通过一个简单的例子来举例说明日常开发的可能涉及到的问题，实际情况可能复杂得多, 通过该例子说明一个好好表结构设计需要DBA多多站在业务的角度来思考问题，而不是一遇到数据库性能问题只会更换更好的硬件，加更大的内存、换更好的SSD或者调整my.cnf参数来优化数据库。