---
layout: post
title: "MySQL学习笔记(四)：MySQL定时备份和恢复"
subtitle: ""
author: ""
header-style: text
tags:
  - mysql
---



定时备份：使用 `mysqldump` + `crontab`



### mysqldump 用法

- 备份全部数据库的数据和结构

  ```sql
  mysqldump -uroot -p123456 -A > /data/mysqlDump/mydb.sql
  ```

- 备份全部数据库的结构（加 -d 参数）

  ```sql
  mysqldump -uroot -p123456 -A -d > /data/mysqlDump/mydb.sql
  ```

- 备份全部数据库的数据(加 -t 参数)

  ```sql
  mysqldump -uroot -p123456 -A -t > /data/mysqlDump/mydb.sql
  ```

- 备份单个数据库的数据和结构(数据库名mydb)

  ```sql
  mysqldump -uroot-p123456 mydb > /data/mysqlDump/mydb.sql
  ```

- 备份单个数据库的结构

  ```sql
  mysqldump -uroot -p123456 mydb -d > /data/mysqlDump/mydb.sql
  ```

- 备份单个数据库的数据

  ```sql
  mysqldump -uroot -p123456 mydb -t > /data/mysqlDump/mydb.sql
  ```

- 备份多个表的数据和结构（数据，结构的单独备份方法与上同）

  ```sql
  mysqldump -uroot -p123456 mydb t1 t2 > /data/mysqlDump/mydb.sql
  ```

- 一次备份多个数据库

  ```sql
  mysqldump -uroot -p123456 --databases db1 db2 > /data/mysqlDump/mydb.sql
  ```

  

### crontab 用法

crontab不同操作系统实现不同，语法是通用的

crontab 表达式含义：

```bash
* * * * *
```

1. 表示分钟
2. 表示小时
3. 表示一个月中的第几天
4. 表示月份
5. 表示一个星期中的第几天



- `*` 表示分钟都要执行（以此类推）
- `a-b` 表示从 a 到 b 这段时间内要执行（以此类推）
- `*/n` 表示每 n 分钟执行一次（以此类推）
- `a,b,c` 表示在第 a、b、c 分钟执行（以此类推）





### 定时任务脚本

```bash
#!/bin/bash

number=31
backup_dir=/var/lib/mysql
dd=`date +%Y-%m-%d_%H:%M:%S`
tool=mysqldump
username=root
password=xxxx
database_name=test

if [ ! -d $backup_dir ];
then
    mkdir -p $backup_dir;
fi

$tool -u$username -p$password $database_name > $backup_dir/$database_name-$dd.sql

delfiles=`ls -l -crt $backup_dir/*.sql | awk '{print $9 }' | head -1`

count=`ls -l -crt $backup_dir/*.sql | awk '{print $9 }' | wc -l`

if [ $count -gt $number ];
then
    rm $delfiles;
fi
```







### 利用全量备份和binlog恢复数据

1.  使用全量备份恢复临时库

   ```bash
   mysql -uroot -p database < dump.sql
   ```

2.  `flush logs` 重开一个binlog，一是避免操作当前binlog文件防止发生意外情况，二是缩小范围排除干扰，在之前的binlog中定位操作范围

   ```bash
   flush logs
   ```

3. 使用 `mysqlbinlog` 导出 sql，主要是设置 `--start-position` 和 `--stop-position`，不需要设置 `--base64-output=decode-rows`

   ```bash
   mysqlbinlog --start-position "6276" --stop-position "6481" --database test binlog.000011 > binlog.sql
   ```

4. 使用导出的sql进行增量恢复

   ```bash
   mysql -uroot -p database < binlog.sql
   ```

   



### 参考

[MySQL 定时备份数据库（全库备份）](https://www.cnblogs.com/letcafe/p/mysqlautodump.html)