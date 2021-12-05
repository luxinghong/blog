---
layout: post
title: "MySQL学习笔记(十一)：order by"
subtitle: ""
author: ""
header-style: text
tags:
  - mysql
---





# 全字段排序

从 `explain` 中可看到 `Extra` 内容为 `Using filesort`，表示需要排序(**统一表示需要排序，不区分内部还是外部**)，MySQL 会为每个线程分配一块内存用于排序，称为 **`sort_buffer`**。

为了演示效果，先将 `sort_buffer_size` 设为 32768(32k)，顾名思义即为 `sort_buffer` 的大小。默认值为256k，最小值为32k。

排序根据数据量和 `sort_buffer` 大小，可能是内存排序，也可能是外部排序(需要借助磁盘临时文件辅助排序)。

可通过查看 `optimizer_trace` 来确定是否使用了外部排序：

```sql
# 只对当前session生效
1.set optimizer_trace = 'enabled=on';
# 这两句必须一起执行才能看到 `OPTIMIZER_TRACE` 输出，而且只能在命令行中使用，datagrip中看不到输出
2.执行sql;SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```

截取 `optimizer_trace` 部分输出(在末尾部分)如下：

```json
"filesort_summary": {
              "memory_available": 32768,
              "key_size": 8,
              "row_size": 39,
              "max_rows_per_buffer": 840,
              "num_rows_estimate": 18446744073709551615,
              "num_rows_found": 20211,
              "num_initial_chunks_spilled_to_disk": 21,
              "peak_memory_used": 35384,
              "sort_algorithm": "std::sort",
              "sort_mode": "<fixed_sort_key, packed_additional_fields>"
            }
```

`num_initial_chunks_spilled_to_disk` 即为排序过程中使用的临时文件数，可见此次排序使用了21个临时文件，一般使用**归并算法**。

`memory_available` 即为排序可使用的内存大小，和上面设置的 `sort_buffer_size`一致。

`peak_memory_used` 表示排序过程中的峰值内存使用大小，可见大于 `sort_buffer_size`，内存装不下了，只能使用外部排序。

`<fixed_sort_key, packed_additional_fields>` 表示 `sort_buffer` 中包含了排序的字段和查询引用到的字段，结果集会直接从 `sort_buffer` 中返回。 `packed` 表示排序过程对字符串做了“紧凑”处理，在排序过程中是按照字段实际长度来分配空间的。

`num_rows_found` 表示有 20211 行数据参与了排序。



将 `sort_buffer_size` 恢复为默认值：262144(256k)，再看看有什么变化：

```json
"filesort_summary": {
              "memory_available": 262144,
              "key_size": 8,
              "row_size": 35,
              "max_rows_per_buffer": 1001,
              "num_rows_estimate": 18446744073709551615,
              "num_rows_found": 20211,
              "num_initial_chunks_spilled_to_disk": 0,
              "peak_memory_used": 43043,
              "sort_algorithm": "std::stable_sort",
              "unpacked_addon_fields": "using_priority_queue",
              "sort_mode": "<fixed_sort_key, additional_fields>"
            }
```

可看到 `peak_memory_used` < `memory_available`，说明 `sort_buffer` 完全够用了，直接在内存排序即可，`num_initial_chunks_spilled_to_disk` 为0，表示不需要借助磁盘临时文件来辅助排序。

而且 `sort_mode` 中也没有了 `packed`，因为内存空间已经完全够用了，不再需要做“紧凑”处理了。

`using_priority_queue` 表示使用了优先级队列，即堆排序。

------

<br/><br/><br/>





# `rowid` 排序

再来看另一个参数：**`max_length_for_sort_data`**，默认值为4096，用于控制排序的行数据长度。如果单行数据长度超过这个值，MySQL就会使用 `rowid排序`。

在 `optimizer_trace` 中的 `sort_mode` 表现为 `<fixed_sort_key, rowid>`，意思是排序时只使用排序字段和rowid，目的是为了装入更多数据，尽量使用内存排序。自然，为了获取其他字段内容，还需要根据rowid回表查询。

并且，如果转成 `rowid` 排序内存还放不下的话，还是会转成外部排序。

MySQL的一个设计思想是能用内存就用内存，对于Innodb表来说，会优先选择全字段排序，因为不需要回表(当数据页不在内存中时，回表还是需要读磁盘)，结果集直接就从 `sort_buffer` 返回了。外部排序是最后的选择。

**注：从8.0.20开始，`max_length_for_sort_data` 已经不推荐使用了，调整这个参数将不会有任何作用。**

------

<br/><br/><br/>







# 如何优化

从上面分析可以看出，尽量使用内存排序是最优解，对应可调整的参数是 `sort_buffer_size`。那还有没有更进一步的优化？

**很明显，借助索引，可以完全避免排序**。

比如这样一句sql：

```sql
select * from t where city = 'xxx' order by name limit 1000;
```

便可以建立一个 `(city,name)`  的联合索引，那么在 `(city,name)` 这棵索引树上便是先按 `city` 排序，再按`name` 排序(即 city 相同时 name 局部有序)。

这样的话只需先快速定位到 city，然后依次往后遍历取出 id，再回表取出整行数据，作为结果集的一部分直接就返回了，完全避免了排序。<u>而且，由于是有序的，对于 limit 来说更友好，不再需要遍历所有 city=xxx 的数据了，遍历到第1000条便可直接结束了</u>(`explain` 中的 `rows` 不准)。



更进一步，如果查询没必要获取全部字段，则可利用到**覆盖索引**，直接从索引树获取到全部字段，连回表查询都不用了。

如上面的 sql 可改为：

```sql
select city,name,age from t where city = 'xxx' order by name limit 1000;
```

此时建一个 `(city,name,age)` 的联合索引，即可满足排序需求，又不需要回表查询，此时是最优解。

当然，还是那句老话，维护索引有代价，需综合考虑。

------

<br/><br/><br/>





# 示例

假设维护一个人员信息表，现已有索引 `(city,name)` ，问以下查询是否需要排序？

```sql
select * from t where city in (A,B) order by name limit 100;
```

虽然已有 `(city,name)` 索引，但只能保证同一 city 内 name 有序，不同 city 之间 name 仍然是无序的。所以该查询在数据库内部仍然需要排序。



如果出于种种原因想要避免在数据库内部排序，有什么办法？

那就在应用端排序，把一条sql拆为两句：

```sql
select * from t where city = A order by name limit 100;
select * from t where city = B order by name limit 100;
```

**拆开的查询可以利用到索引，不需要排序**。

在应用端便可得到两个有序数组，使用归并排序合并为一个有序数组后，取前100即为结果集。





如果是分页查询呢，比如：

```sql
select * from t where city in (A,B) order by name limit 10000,100;
```

还是和上面的思路一样，拆分为2个查询：

```sql
select * from t where city = A order by name limit 10100;
select * from t where city = B order by name limit 10100;
```

同样使用归并算法得到一个有序数组，取第 10000 ~ 10100 的值即为结果集。

这时的问题是数据量太大，如果应用端内存紧张的话，可考虑只查询 id 和 name：

```sql
select id,name from t where city = A order by name limit 10100;
select id,name from t where city = B order by name limit 10100;
```

该查询不需排序，且会用到覆盖索引，也不用回表。合并得到有序数组后，取出第 10000 ~ 10100 的 id，再拿这100个 id 去数据库查出所有记录即可。

------

 <br/> <br/> <br/>





# 官方优化指南

[ORDER BY Optimization](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html)