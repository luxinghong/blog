---
layout: post
title: "MySQL学习笔记(二)：MySQL权限控制"
subtitle: ""
author: ""
header-style: text
tags:
  - mysql
---



MySQL权限可分为全局级、数据库级、表级、列级、子程序级（函数、存储过程）



### 全局级

```sql
select * from mysql.user;
```



### 数据库级

```sql
select * from mysql.db;
```



### 表级

```sql
select * from mysql.tables_priv;
```



### 列级

```sql
select * from mysql.columns_priv;
```



### 子程序级

```sql
select * from mysql.procs_priv;
```



### 查询用户拥有权限

```sql
show grants for root@localhost;
```





### 示例

```sql
#创建用户
create user 'test'@'%' identified by  'test';

#授予全局级查询权限
grant select on *.* to 'test'@'%';

#授予数据库级查询权限
grant select on test.* to 'test'@'%';

#授予表级查询权限
grant select on mysql.user to 'test'@'%';

#授予列级权限（只能查询person表的name字段）
grant select(name) on test.person to 'test'@'%';

#撤销权限
revoke select on *.* from 'test'@'%';

#刷新权限
flush privileges;
```

