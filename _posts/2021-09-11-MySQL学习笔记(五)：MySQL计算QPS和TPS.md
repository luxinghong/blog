---
layout: post
title: "MySQL学习笔记(五)：MySQL计算QPS和TPS"
subtitle: ""
author: ""
header-style: text
tags:
  - mysql
---



### QPS

基于 Com_select

> 基于questions的会统计show命令，mysql设置环境变量的时候也会增加，不太准。
>
> Com_select 用于统计 select 语句的执行次数，类似的还有 Com_delete、Com_update 等。
>
> 参考：[Com_xxx](https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html#statvar_Com_xxx)

```bash
#!/usr/bin/env bash
OLD_QPS=`echo "show global status where Variable_name='Com_select';"|mysql -uroot -pxxx -N|awk '{print $2}'`
sleep $1
NEW_QPS=`echo "show global status where Variable_name='Com_select';"|mysql -uroot -pxxx -N|awk '{print $2}'`
echo "($NEW_QPS-$OLD_QPS) / $1" | bc
```

`$1` 代表添加到shell的第一个参数值，`$2` 代表第二个，以此类推。`$0` 为shell文件名。

获取当前时刻的 qps：

```bash
./qps.sh 1
```





### TPS

```bash
#/usr/bin/env bash
OLD_COM_INSERT=`echo "show global status where Variable_name='Com_insert';"|mysql -uroot -pxxx -N|awk '{print $2}'`
OLD_COM_UPDATE=`echo "show global status where Variable_name='Com_update';"|mysql -uroot -pxxx -N|awk '{print $2}'`
OLD_COM_DELETE=`echo "show global status where Variable_name='Com_delete';"|mysql -uroot -pxxx -N|awk '{print $2}'`
sleep $1
NEW_COM_INSERT=`echo "show global status where Variable_name='Com_insert';"|mysql -uroot -pxxx -N|awk '{print $2}'`
NEW_COM_UPDATE=`echo "show global status where Variable_name='Com_update';"|mysql -uroot -pxxx -N|awk '{print $2}'`
NEW_COM_DELETE=`echo "show global status where Variable_name='Com_delete';"|mysql -uroot -pxxx -N|awk '{print $2}'`

echo "(($NEW_COM_INSERT - $OLD_COM_INSERT) + ($NEW_COM_UPDATE - $OLD_COM_UPDATE) + ($NEW_COM_DELETE - $OLD_COM_DELETE)) / $1" | bc
```

