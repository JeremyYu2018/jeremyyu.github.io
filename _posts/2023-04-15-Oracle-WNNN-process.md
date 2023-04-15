---
layout: post
title: Oracle 19C（19.3） 游标泄漏？
date: 2023-04-15
categories: blog
tags: [Oracle, bug]
description: 
---
### 背景
告警系统显示Oracle其中一条SQL打开游标过多，登录进数据库后发现如下异常SQL。
```SQL
SQL> select sql_id,count(*) from v$open_cursor 
group by sql_id
order by count(*) desc

SQL_ID	COUNT(*)
frccccnh76gtx	2366
9zg9qd9bm4spu	341
gkttwbrtwuw90	206
f0h5rpzmhju11	200
2rrzm2a21d093	200
9s5cdq3h4nfbj	200
0przsb2sf5q8c	200
865qwpcdyggkk	198
5dqz0hqtp9fru	198
0k8522rmdzg4k	198
agxgdkx5zz9by	42

```
发现排在最前面的SQL frccccnh76gtx打开了超过2千多个游标，查看该SQL的原始内容
```
select rfile# from v$datafile where file# = :1
```

查看版本：
```
"Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0"
```

经过查阅资料发现是Oracle bug，是由Oracle SMCO进程导致的，临时解决方案是重启SMCO, 同时我还观察到当每个表空间只有一个数据文件时不会触发此bug，只有大于1个数据文件时才会触发该Bug。

当前受影响版本：
oracle Database - Enterprise Edition - 版本 18.1.0.0.0 到 19.3.0.0.0 

临时处理步骤:
可以通过设置隐藏参数来重启SMCO

```
#禁用
alter system set "_enable_spacebg"=false;
alter system set "_ENABLE_SPACE_PREALLOCATION" = 0;

#启用
alter system set "_enable_spacebg"=true;
alter system set "_ENABLE_SPACE_PREALLOCATION" = 3;

```

### 长期解决方案
 打补丁
打19C的最新补丁包即可，只不要最终版本不是19.3即可。

参考链接：
Bug 30098251 : WNNN PROCCESSES CREATE AN EXCESSIVE NUMBER OF OPEN CURSORS	
[https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=556728326036697&parent=BUG_MATRIX&sourceId=30098251&id=2639882.1&_afrWindowMode=0&_adf.ctrl-state=qgyko2vr_618](Wnnn 后台进程过量的 open cursors (Doc ID 2639882.1)	To BottomTo Bottom)