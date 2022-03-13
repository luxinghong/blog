---
layout: post
title: "MySQL学习笔记(十二)：join"
subtitle: ""
author: ""
header-style: text
tags:
  - mysql
---



**先说个结论，join的优化思路都是尽量让被驱动表上join的字段用上索引。**



所以，分析join也从被驱动表上join字段有没有索引两种情况来分析。



# 可以使用被驱动表的索引

### NLJ-Nested Loop Join

这是一种最自然的算法，假设由我们来实现join首先也会想到这种方法。

以 `select * from t1 join t2 on t1.a=t2.a` 为例，`t2.a` 上有索引。

步骤如下：

1. 从t1表中取出一行R；
2. 从R中取出a字段，到t2表中查找；
3. 从t2中取出匹配的行，和R组成一行，作为结果集的一部分；
4. 重复1-3步骤，直至t1表遍历结束。

所以它就是2个for循环，外层循环嵌套内存循环，这也是  `Nested Loop Join` 名称的由来。

由于 t2.a 上有索引，在内存循环查找的时候，走的是索引树，快速定位，无需全表扫描，所以这种方式是很快的。

如果一条join语句执行计划的 extra 字段什么都没写，就表示使用了 NLJ 算法。





# 被驱动表上没有索引

如果被驱动表上没有索引，按照 NLJ 算法，会导致内存循环被驱动表上做全表扫描，效率会很低。

MySQL作了一种改进，称为 BNL 算法。

### BNL-Block Nested Loop

既然问题出在内存循环被驱动表上做全表扫描，那么就从这个问题入手，避免在被驱动表上做全表扫描。

如何做？

MySQL会为每个连接分配一个内存区域：`join_buffer`，大小由 `join_buffer_size` 控制，默认值为256k。

步骤如下：

1. 将t1表读入join_buffer ；

   > 注意：只会读入结果集中需要的字段，比如  `select *`  会读入t1表所有字段；而 `select t1.a,t1.b,t2.* from t1 join t2 on t1.a=t2.a`  只会读入 t1 表的 a、b 字段

2. 从 t2 表中取出每一行R，到 join_buffer 中去查找，满足条件的和R组成一行，作为结果集的一部分返回。

**这个算法的重点就在于对于被驱动表只需要扫描一次，然后所有的查找、匹配工作都在内存(join_buffer)中完成，速度很快。**

其实有点反转的意思，被驱动表转为了驱动表。

那这个 Block 是什么意思？

理想情况下 join_buffer 能够全部读入 t1 表的数据自然是最好，如果不能完全装入，则需要分块读入，这就是 Block 的含义。

显而易见，这和 `join_buffer_size` 息息相关。

如果需要分块读入，假设需要分3次读入，执行步骤就变成了执行3次上面的1、2步骤，这时需要扫描3次被驱动表。

所以增大 `join_buffer_size` 也是优化 join 的一个常用手段。





# 如何选择驱动表

- 假设驱动表行数N，被驱动表行数是M。

  对于 NLJ ，时间复杂度近似 $N + N*2*log_2M$

  > 驱动表需要扫描N行
  >
  > 执行N次被驱动表查找、匹配
  >
  > 在被驱动表上定位的时间复杂度为 $log_2M$
  >
  > 需要回表，$*2$

  可看到N相对M的影响更大。

- 对于 BNL，很明显，能一次性把驱动表的数据都装入 join_buffer，效率是最高的。

  > 需要注意一点，BNL中小表的选择应该是以实际使用数据量大小为准。
  >
  > 比如这2个sql：
  >
  > ```sql
  > select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
  > select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
  > ```
  >
  > 应以 t2 作为主表，因为 t2 有过滤条件，只会将50行放入 join_buffer。选择 t1 的话，需要将 t1 表所有数据放入 join_buffer。
  >
  >  
  >
  > 又比如
  >
  > ```sql
  > select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;
  > select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
  > ```
  >
  > 因为 t1 只需要查询 b 字段，以 t1 为主表只需将字段b放入 join_buffer。
  >
  > 如果以 t2 作为主表，需要将 t2.id<=100 的数据读入 join_buffer。
  >
  > 这时就需要比较两者数据量大小，选择较小一方作为主表。



**综上所述，总是应该选择小表作为驱动表。**







# 进一步优化

### 对 NLJ 的优化

先介绍下 MRR ：MySQL 针对回表所做的一种优化算法。

**假设回表查到的记录都不在 buffer_pool 中 (如果数据页都在内存中，直接快速定位，那就没什么影响)，需要读盘，但是回表的 id 对应的记录所在的数据页并不一定是连续的，这就意味着需要随机读盘。**

众所周知，随机读盘是最耗时的操作。对此，MySQL提出了一种回表优化算法：

#### MRR-Multi Range Read

1. 针对该情况，会将需要回表的id放到 read_rnd_buffer 中

2. 然后在 read_rnd_buffer (大小由参数 read_rnd_buffer_size 控制) 中对 id 进行递增排序，再到主键索引树上查找。

   此时，id 是连续的，即便需要读盘，也是顺序读盘，这就大大提高了效率。

   而且，还能利用到CPU的空间局部性原理：CPU 在读取数据时，会将附近区域的一段数据一并读入。这是基于一个理论：如果使用了某个位置的数据，在不久的将来该位置附近的数据大概率也会被使用到。因此，一并读入以提高效率。

> 默认情况下，该优化没开启。
>
> 开启方法：`set optimizer_switch='mrr=on,mrr_cost_based=off'` 。
>
> 可看到这是在优化器中的设置。在 explain 中的 extra 字段显示 `Using MRR` 即表示使用了 MRR 优化算法。



#### BKA-Batched Key Access	

BKA 其实就是 MRR 在 NLJ 算法上的应用。

还是以上文 NLJ 中示例说明，它的逻辑是用 t1.a 去匹配 t2.a，匹配到以后，**需要回表**，得到 t2 完整的一行记录再与 t1 的记录组成最终结果集中的一行记录。

BKA 利用 MRR ，将回表中的随机读盘优化为顺序读盘。

具体步骤和 MRR 是一致的，区别是这里用的不是 read_rnd_buffer，而是 join_buffer (没错，NLJ 也可以用上 join_buffer)。

由于默认 MRR 没有开启，所以 BKA 也需要手动开启，开启方法：

```sql
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```







### 对 BNL 的优化

最直接的方法就是给被驱动表 join 字段加上索引，让 BNL 转 BKA 算法即可。



那如果出于种种原因，不能加索引，咋办？

1. 基于上文 BNL 那一节的描述，没办法用索引，只能减少被驱动表的全表扫描次数。即增大 join_buffer_size，可以一次性装入驱动表，尽量只全表扫描1次被驱动表；

2. 还有一种曲线救国的方法，既然不能在原表上加索引，那就创建一张临时表，插入被驱动表的数据，在临时表上加索引，然后再和这张临时表join，让它使用 BKA 算法；

3. MySQL8 支持了 hash join。何为 hash join？

   原先存在 join_buffer 中的结构是无序数组，所以 BNL 算法需要遍历比较。假设被驱动表有100万行记录，就需要进行 100万次对主驱动表的遍历比较。hash join 将 join_buffer 中的数据结构改为了哈希表，这样就无需进行100万次对主驱动表的遍历比较，而是100万次哈希查找，哈希查找的时间复杂度是 O(1)，效率大大提高了。

   > MySQL5 并不支持 hash join，但是可以在应用端使用代码模拟。
   >
   > 用一个哈希结构存储主驱动表，遍历被驱动表，然后在哈希表中根据join字段进行哈希查找，最后组装结果返回即可。