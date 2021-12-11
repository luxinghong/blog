---
layout: post
title: "MySQL学习笔记(七)：MySQL索引"
subtitle: ""
author: ""
header-style: text
tags:
  - mysql
  - 索引
---



# 索引的常见实现方式有哪些？

- 哈希表：O(1) 的时间复杂度，速度最快，但缺点是只适用于等值查询。因为key是无序的，所以区间查询时只能全部遍历一遍。
- 有序数组：O(logn)的时间复杂度，利用二分法。可用于等值查询和区间查询，但插入删除时间复杂度较高，因为需要移动插入点后面的所有元素。所以有序数组比较适合静态存储引擎，即基本不会变的数据。
- 搜索树：常用实现是B+树。

------

<br/><br/><br/>



# 为什么采用B+树而不是常见的二叉树?

二叉树即每个节点只有左右2个子节点，所以显而易见的问题就是当节点变多时树的高度会很高。比如需要存储100万条数据，就需要20层（n层二叉树的节点数为 $2^n-1$，20层二叉树的节点总数为1048576）。因为一个节点就是一页，那么一次查询很可能就需要进行20次随机IO（大概率会触发随机IO），在传统机械硬盘时代，一次随机IO大约10ms，那么单一次查询可能就需要200ms，这个查询是很慢的。

所以解决办法就是增加树的子节点数，由二叉变为N叉。

B+树就是一颗N叉树。一个节点就是一页，页是Innodb磁盘IO的基本单位，一页在Innodb中默认是16k，假如索引字段为整数类型占4个字节，每个key还有一个指向下一层节点的指针固定占6字节，再加上一些辅助字节总共差不多占13字节左右（非叶子节点）。16k/13=1260，那么一个节点就可以差不多有1200个分叉，一颗树高为4的B+树，就可以存 $1200^3≈17亿$ 个值。因为根节点总是在内存中，第二层大概率也在内存中，这时17亿数据量的单次查询页只需要进行2-3次磁盘IO，速度大大提高了。

顺便说一下 Innodb 中B+树的特点：

- N叉树，即每个节点可以有多个key
- 内部节点不存储数据，只有指针，只在叶子节点存储数据
- 每一层页与页之间构成一个双向链表
- 页内 records 之间构成一个单向链表
- 叶子节点为第0层，从下往上递增，root为最大层数
- 一个节点就是一个page





### Innodb中的页是什么？



#### Innodb表空间

> 详见：[The basics of InnoDB space file layout](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/)
>

Innodb的数据存储模型被称为 `space`，即“表空间”。表空间是一个逻辑概念，有一个32位的space ID，实际上可能由多个物理文件组成（如ibdata1、ibdata2）。表空间分为系统表空间（system space：ibdata1、ibdata2，space ID 为0）和表对应的表空间(per-table space：ibd文件)。ibd文件实际上是一个功能齐全的space，可以包含多张表，但在MySQL实现中一个ibd只对应一张表。

每个space会被划分为多个page，一个page默认16k。page也有一个32位的page number（页号），表示在space内的偏移量（offset），比如page 0 对应 offset 为0，page 1 对应 offset 为16384。注意一个space可能包含多个文件，所以这个offset不一定是文件内的，而是整个space中的。Innodb单表空间最大为64TB，是因为 $2^{32} * 16k$。

#### 页的基本结构

![image-20211112105252435](/blog/img/image-20211112105252435.png)

页包含一个38字节的头部（FIL为File的缩写）和一个8字节的尾部，中间的内容取决于不同的page type，可用大小为 16k-38-8=16338。



FIL Header 和 Trailer 结构如下：

![image-20211112105757234](/blog/img/image-20211112105757234.png)

可以看到，头部包含了

- Offset(Page Number)
- Page Type
- Space ID
- 指向前一页和后一页的指针，构成一个双向链表（树的同一层中）
- 最后一次改动页的LSN
- 当前系统中（所有space）最大的LSN



#### 表空间(space file)文件结构

![image-20211112125701387](/blog/img/image-20211112125701387.png)



#### 系统表空间(system space)文件结构

系统表空间(system space)的 space ID 为 0 。它采用了一些固定页号的页来存储一些关键信息。结构如下：

![image-20211112130023998](/blog/img/image-20211112130023998.png)



#### 单表空间(per-table space file)文件结构

![image-20211112130153515](/blog/img/image-20211112130153515.png)

Page3为聚簇索引(主键索引)的root，Page4为第一个二级索引的root，如果有多级索引的话以此类推。





### Innodb索引

> 详见 [The physical structure of InnoDB index pages](https://blog.jcole.us/2013/01/07/the-physical-structure-of-innodb-index-pages/)

#### 一切皆索引	

在Innodb中一切皆索引，意思是：

- 每张表都有一个主键。如果没有手动指定，会使用第一个 not null 的 unique key。如果仍然没有，会自动分配一个6字节的隐藏 `Row ID`作为主键。
- 主键索引树(聚簇索引)叶子节点key是主键值，value是是整行数据。
- 二级索引key是索引列的值，value是对应的主键值。
- 一张表有几个索引，就有几棵B+树。且至少有一棵主键B+树，数据存储在主键索引树上。查询不走索引其实是遍历主键索引树。
- B+树中一个节点为一页。



#### 索引结构

因为一个索引就是一棵B+树，B+树中一个节点对应一页，所有索引页具有和上面讲到的页的相同基本结构，都包含一个FIL Header和FIL Trailer，主体部分会有所不同，如下图所示：

![image-20211112140640173](/blog/img/image-20211112140640173.png)

重点关注其中的 `User Records` 和 `Page Directory`。

User Records 是实际存储数据的地方：

- 非叶子节点：存储指向下一层子节点的指针
- 叶子节点：假设为主键索引树，存储的就是行数据

一个page中的所有 User Records 组成了一个单链表，头是一个叫 `infimum` 的 system record(存储了当前页中最小的key)，尾是一个叫 `supremum` 的 system record(存储了当前页中最大的key)。



Index Header 结构如下：

![image-20211112142023840](/blog/img/image-20211112142023840.png)

可以看到有 `Number of Records` 、 `Page Level(叶子节点所在层为第0层，从下往上递增，root节点所在层为最大层)` 等等。





#### 二级索引叶子节点value的排序问题

假设存在以下表记录

|  a   |  b   |  c   |  d   |
| :--: | :--: | :--: | :--: |
|  1   |  2   |  3   |  d   |
|  1   |  3   |  2   |  d   |
|  1   |  4   |  3   |  d   |
|  2   |  1   |  3   |  d   |
|  2   |  2   |  2   |  d   |
|  2   |  3   |  4   |      |

存在一个联合主键 `(a,b)`，三个索引 `c(c)`、`ca(c,a)`、`cb(c,b)`，分析这三个索引是否有冗余？

普通索引叶子节点存的值为主键，这里的主键为联合主键。

索引 `ca` 即先对 c 排序，再对 a 排序，因为key已经包含了a，所以value只需要存储 b，ca相同时，b升序。记录如下：

| c(key的部分) | a(key的部分) | b(value存的值) |
| :----------: | :----------: | :------------: |
|      2       |      1       |       3        |
|      2       |      2       |       2        |
|      3       |      1       |       2        |
|      3       |      1       |       4        |
|      3       |      2       |       1        |
|      4       |      2       |       3        |

再看索引 `c`。key先对 c 排序，再对 a 排序，再对 b 排序，结果和 `ca` 是一样的。所以索引`ca` 是多余的。

索引 `cb` 先对 c 排序，再对 b 排序，再对 a 排序，可用于基于c、b 的查询，需要保留。

 











#### 实践一下

可以通过 `innodb_space` 命令直接分析文件，获取文件中存储的page、records等信息。(目前还不支持MySQL8.0)

> 详见：
>
> [innodb_ruby](https://github.com/jeremycole/innodb_ruby)
>
> [B+Tree index structures in InnoDB](https://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/)

------

<br/><br/><br/>





# 基于主键索引和普通索引的查询有什么区别

主键索引树叶子节点直接存储行数据，所以主键索引查询只需要扫描主键索引树即可。

而普通索引树叶子节点存储的是主键值，所以需要先扫描普通索引树拿到主键值，再回到主键索引树获取行数据，相较于主键索引查询多扫描了一棵索引树，这个过程称为 `回表`。

------

<br/><br/><br/>





# 选择自增主键还是业务主键

可从存储空间大小和性能两个方面来考虑：

- 占用空间：业务主键相较于自增主键都较长，由于二级索引树叶子节点存储的是主键值，所以采用业务主键的二级索引相较于自增主键会占用更多的空间。
- 性能：由于自增主键是有序的，所以在维护索引树时直接追加即可(**叶子节点所在层即第0层的最后一个节点中的最后一个record后**)，当一页写满会自动开辟一个新的页。而业务主键很难保证有序性，维护索引时很可能会在**中间插入**，就很有可能引起节点分裂(甚至是父索引节点的分裂)，自然性能会受到影响。

所以，在大多数情况下，都应优先使用自增主键。

当然事无绝对，如果只有一个索引，且该业务字段是唯一的，可以将该字段设为主键。因为不存在其他索引，就不用考虑其他索引的叶子节点大小问题。当然，性能上相较于自增主键还是会有一点影响。





### 一些索引设计原则

假设存在表 `u(id,id_card,name,age,gender)`，`id` 是主键，另有一个 `id_card` 索引





#### 覆盖索引

即索引key中包含了要查找的字段。

假如现需要根据身份证查询数据，可直接走 `id_card` 索引。

现又有另一个**高频**需求，根据身份证查询姓名。目前走的仍然是 `id_card` 索引，需要先在 `id_card` 索引树上找到对应的 `id_card`，然后再回到主索引树上根据 `id`获取到姓名，也就是 `回表`。

那有没有什么办法可以优化呢？

方法就是建立一个 `(id_card,name)` 的联合索引。这棵索引树节点的key包含了两个字段：id_card 和 name，这样的话直接在 `(id_card,name)` 索引上搜索便可直接得到 name，而不需要回表查整行记录，减少了语句的执行时间。与此同时，根据最左匹配原则，原先根据身份证查询数据的请求也可以用到这个索引，所以现在可以删除`id_card` 这个索引了。

这就是覆盖索引，即索引key中包含了要查找的字段。可使用 `explain` 查看是否使用了覆盖索引，**如果在 `explain` 中的 `extra` 列中出现了 `Using index`，说明当前查询使用了覆盖索引**，即不需要回表查询。

当然，索引是有代价的。因为每新建一个索引就相当于新建一棵索引树，虽然可以提高查询速度，但增删改就需要多维护一棵索引树。所以需要权衡使用，任何索引都是这样，数据量小的话就没有什么必要，没有太大区别。





#### 最左匹配原则

假设现在又有一个**低频**需求：根据身份证查询地址，那么有必要再建立一个 `(id_card,address)`的联合索引吗？

答案是不需要。因为这是一个低频请求，意味着请求的次数不会太多，上一节的 `(id_card,name)`索引就够用了。可先通过`(id_card,name)` 这个索引定位到相应的 id_card，获取到主键后再回表查询。原理很简单，因为联合索引 `(a,b)`是先根据 a 排序再根据  b 排序，所以对于 a 的检索可以用到这个B+树。

所以最左匹配原则的定义就是只要索引满足最左前缀，便可利用该索引来加速检索。**这个最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左N个字符。**



如果既有a,b的联合查询，又有基于a、b各自的查询呢？

这时考虑的原则就是**空间**了。如果b字段比a字段大，那么就应该建立一个 `(b,a)`的联合索引和一个 `a` 的单独索引。这两个索引可以同时满足 a,b的联合查询和基于a、b各自的查询。



除此之外，最左匹配原则当遇到范围匹配时就会失效。比如有一个 `(a,b)`的联合索引， `a=1 and b=2` 或 `b=1 and a=2` 都可以用到索引，顺序无所谓，优化器会调整 where 的顺序。但是当遇到类似 `a >1 and b=2` 时，就只有 `a` 能用到索引，会先快速定位到 `a>1` 的记录，此时 `b` 是无序的，只能遍历判断 `b` 是否满足。



示例：

1. 如果 sql 为

   ```sql
   SELECT * FROM table WHERE a = 1 and b = 2 and c = 3; 
   ```

   该怎么建立索引？

   答：第一反应是直接建立 `(a,b,c)` 的联合索引，但是这里要注意**区分度**，区分度高的放在前面。区分度越高，检索效率越高，因为快速定位更精准。像性别、状态这种区分度很低的字段，放到后面。

2. 如果 sql 为 

   ```sql
   SELECT * FROM table WHERE a > 1 and b = 2; 
   ```

   该怎么建立索引？

   答：因为是范围查询，如果建立 `(a,b)` 的索引，就只有 `a` 能用上索引。

   ​	    所以应该建立 `(b,a)` 的索引，优化器会调整条件的顺序，然后`b` 就能用上索引，在此基础上，`a>1` 也能用上。

3. 如果 sql 为

   ```sql
   SELECT * FROM `table` WHERE a > 1 and b = 2 and c > 3; 
   ```

   该怎么建立索引？

   答：首先肯定要以 `b` 开头，所以有两种选择：`(b,a)` 或 `(b,c)`，至于具体选择哪个，就看**区分度**和**字段长度**了。

4. 如果 sql 为

   ```sql
   SELECT * FROM `table` WHERE a = 1 ORDER BY b;
   ```

   该怎么建立索引？

   答：建立 `(a,b)` 联合索引，当 a=1 时，b 相对有序，可以避免再次排序。

   ​		那如果是

   ```sql
   SELECT * FROM `table` WHERE a > 1 ORDER BY b;
   ```

   ​		因为此时是范围查询， a>1 时 b 是无序的，所以没有必要再建立一个 `(a,b)` 的联合索引。

5. 如果 sql 为

   ```sq
   SELECT * FROM `table` WHERE a IN (1,2,3) and b > 1; 
   ```

   该怎么建立索引？

   答：还是建立  `(a,b)` 的联合索引，`IN 查询` 可视为等值查询，相当于 `a=1 or a=2 or a=3`，所以还是一样的思路。







#### 索引下推

索引下推并不是一个索引设计原则，它是一个索引查找的内部优化。

前提：因为范围查询不能使用联合索引，只能使用最左前缀。



以下面的 sql 举例：

```sql
select * from tuser where name like '张%' and age=10 and is_male=1;
```

以 `张` 开头的记录有 `(张三，10)，(张三，10)，(张三，20)，(张六，30)`，有一个联合索引 `(name,age)`。



在 MySQL5.6 之前，存储引擎提供的接口对于这种情况只允许传入`最左前缀`一个参数，即只能传入 `name ` 这个字段，所以需要回表4次用于判断 `age` 和 `is_male` 是否满足条件。

在 MySQL5.6之后，接口可以传入包含最左前缀的整个联合索引，即`name,age`字段。这样的话可直接在 `(name,age)` 索引树上就对 `age` 进行判断，提前过滤掉不满足条件的记录，最后只需要回表2次。

当 `explain 的 extra` 字段中显示 `Using index condition` 时则表示本次查询使用到了索引下推。



#### 不能走索引的反面示例

以下示例用到的表结构如下：

```sql
CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

	
CREATE TABLE `trade_detail` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `trade_step` int(11) DEFAULT NULL, /*操作步骤*/
  `step_info` varchar(32) DEFAULT NULL, /*步骤信息*/
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into tradelog values(1, 'aaaaaaaa', 1000, now());
insert into tradelog values(2, 'aaaaaaab', 1000, now());
insert into tradelog values(3, 'aaaaaaac', 1000, now());

insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');
insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');
insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');
insert into trade_detail values(4, 'aaaaaaab', 1, 'add');
insert into trade_detail values(5, 'aaaaaaab', 2, 'update');
insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');
insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');
insert into trade_detail values(8, 'aaaaaaac', 1, 'add');
insert into trade_detail values(9, 'aaaaaaac', 2, 'update');
insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');
insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
```



1. **索引字段上用了函数**

   会破坏索引的有序性，因此优化器会决定放弃**走树搜索功能**

   如：

   ```sql
   select count(*) from tradelog where month(t_modified)=7;
   ```

   `t_modified` 上虽然有索引，但由于用了 `month函数`，破坏了索引的有序性，导致没办法快速定位。

   注意，**只是不使用树搜索功能，并不是放弃使用这个索引。**比如这个例子，虽然放弃了树搜索快速定位，但是对比主键索引树和 `t_modified索引树`后，发现后者更小，优化器最终还是会选择遍历 `t_modified索引树`，也即是全索引扫描。

   `explain`如下：

   ![image-20211208221822466](/blog/img/image-20211208221822466.png)

   `key` 为 `t_modified`，说明用到了 `t_modified` 索引。`rows` 为 100335(测试数据有十万条)，说明是全索引扫描。

   这个例子要使用快速定位的话，就得把索引上的函数去了:

   ```sql
   select count(*) from tradelog where
       -> (t_modified >= '2016-7-1' and t_modified<'2016-8-1') or
       -> (t_modified >= '2017-7-1' and t_modified<'2017-8-1') or 
       -> (t_modified >= '2018-7-1' and t_modified<'2018-8-1');
   ```

   还有一些例子，我们可能会理所应当的以为优化器会优化，但是并没有，它还是一视同仁：

   ```sql
   select * from tradelog where id + 1 = 10000;
   ```

   虽然 `id+1` 并不会改变索引的有序性，但优化器并不会重写这类语句，一视同仁，必须得改成 `where id = 10000-1` 才行。
   
   <br/><br/>
   
   

2. **隐式类型转换**

   比如这条sql：

   ```sql
   select * from tradelog where tradeid=110717;
   ```

   `explain` 显示走的是全表扫描。原因是因为`tradeid` 是 varchar 类型，参数是 int 类型，明显会需要一个类型转换。

   在MySQL中，**当字符串和数字做比较的时候，是由字符串转为数字**。如果记不住这个规则，可用 `select '10' > 9` 验证一下：结果为1，表示字符串转为了数字。

   所以，上面的sql其实相当于：

   ```sql
   select * from tradelog where  CAST(tradid AS signed int) = 110717;
   ```

   那原因就很明显了，同样是因为索引列上用到了函数，导致不能快速定位。

   <br/><br/>

   

3. **隐式字符编码转换**

   比如这句sql：

   ```sql
   select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;
   ```

   ![image-20211208224214325](/blog/img/image-20211208224214325.png)

   表 `l` 只扫描了一行，表示用了主键索引快速定位，快速定位到了 `id=2` 这一行。

   表 `d` 走了全表扫描，且没用上 `tradeid` 索引。`key=NUll` 是表示走的主键索引遍历。

   简单拆解一下这句sql的执行步骤：

   1. 从表 `l` 中找到 `id=2` 这一行数据，从中取出 `tradeid`；
   2. 从表 `d` 中找到 `tradeid=上一步查询到的值`的数据。

   所以第2步就是：

   ```sql
   select * from trade_detail where tradeid = 上一步查询到的tradeid; 
   ```

   那为什么用不上 `tradeid` 索引呢？

   细心点可发现两个表的字符编码不同，表`l` 是 `utf8mb4`，表`d` 是 `utf8`。`utf8mb4`是`utf8` 的超集，类型转换的时候都是子集转超集，所以上面的sql相当于：

   ```sql
   select * from trade_detail  where CONVERT(traideid USING utf8mb4) = 上一步查询到的tradeid; 
   ```

   所以，原因还是一样，索引字段上加了函数操作导致不能快速定位。

   

   作为对比验证，下面这句sql就能都用到索引快速定位：

   ```sql
   select l.operator from tradelog l , trade_detail d where d.tradeid=l.tradeid and d.id=4;
   ```

   还是和上面一样的分析步骤，只不过这次的连接顺序倒了过来。先找到表 `d` 中 `id=4` 这一行，从中取出 `tradeid`，再到表 `l` 中去匹配，第2步的sql就相当于：

   ```sql
   select operator from tradelog  where traideid = 上一步查询到的tradeid; 
   ```

   由于需要做字符编码转换，记住转换是子集转超集，所以又相当于：

   ```sql
   select operator from tradelog  where traideid =CONVERT(上一步查询到的tradeid); 
   ```

   区别就在于函数操作是加在值上面，所以可以先计算出来，然后索引就能快速定位到。

   

   这类问题解决办法有两个：

   1. 最简单直接的就是把对应 `utf8` 编码的字段改为 `utf8mb4`，这样从根上避免了字符编码转换：

      ```sql
      alter table trade_detail modify tradeid varchar(32) CHARACTER SET utf8mb4 default null;
      ```

   2. 如果不能更改字符编码，那只能手动改下sql，手动来做这个编码转换，如上面的sql可改为：

      ```sql
      select d.* from tradelog l , trade_detail d where d.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2; 
      ```

      这里手动把 `l.tradeid` 转为了 `utf8`，保证了编码一致，避免了编码转换。

      **注意**：手动转换要确保不会丢失精度才行。
      
      <br/><br/>
   
4. 字符串截断

   比如有这么一张表：

   ```sql
   CREATE TABLE `t` (
     `id` int(11) NOT NULL,
     `b` varchar(10) DEFAULT NULL,
     PRIMARY KEY (`id`),
     KEY `b` (`b`)
   ) ENGINE=InnoDB;
   ```

   假设现在表里面，有 100 万行数据，其中有 10 万行数据的 b 的值是 '1234567890'，然后查询是这样写的：

   ```sql
   select * from table_a where b='1234567890abcd';
   ```

   细心一点会发现，传进去的值超过了字段 `b` 定义的长度，理想情况下根本就不用去查询，直接返回空就行了。事实上这句sql执行了很长时间才会返回空，它的步骤是这样的：

   1. 由于超过了字段 `b` 定义的长度，会首先做字符串截断，最终传到引擎层的值变成了 '1234567890'；
   2. 匹配到10万条数据，然后由于是 `select *`，还需要回表10万次；
   3. 每次回表后查出整行，到 Server 层一判断，不等于 '1234567890abcd'，最后返回空。

   对于这种情况，最好就是先在应用层判断过滤一下，MySQL并不总是那么智能。

<br/><br/><br/>

------

<br/><br/><br/>





# 选择唯一索引还是普通索引？

从读和写两方面来分析。

- 读：二者区别就在于是否唯一。唯一索引找到记录后即可返回，普通索引还需继续向后遍历检查是否满足条件。但此时数据页已在内存中，而且很大概率都是页内遍历(通过二分法)，这点差异对于现在的CPU来说可以忽略不计。所以可认为二者在查询方面差异不大。

- 写：需要考虑目标数据页是否在内存中，下面以 `insert` 举例说明
  - 目标数据页在内存中：对于唯一索引来说，找到插入位置，判断到没有冲突，插入，语句执行结束；对于普通索引来说，找到插入位置，插入，语句执行结束。这种情况下直接更新内存即可，性能也没有多大差异。
  - 目标数据页不在内存中：
    - 由于唯一索引需要判断唯一性，所以必须要将数据页从磁盘读到内存。
    - 而普通索引没有这个要求，所以可直接在内存中记录下一条 "insert" 操作，语句就执行结束了。记录这个操作的区域叫 `change buffer`， 由于只需要写内存，避免了磁盘的随机读(磁盘的随机IO是数据库中成本最高的操作之一)，这种情况下普通索引性能就远远优于唯一索引，尤其如果是机械硬盘的话。

综上所述，如果业务可以接受的话，从性能角度出发，应该选择普通索引。



### change buffer

`change buffer` 是 `buffer pool` 的一部分，默认占比为 25(%)，最大占比可设为 50(%)。

```sql
show global variables like 'innodb_change_buffer_max_size';
```

在MySQL5.5之前的版本，只支持缓存`insert`操作，所以最初叫 `insert buffer`(很多地方见到的 `ibuf` 指的就是它，后来也一直延用了下来)。后来也加入了对 `update`、`delete` 的支持，便改名为了 `change buffer`。



#### 作用

**当目标数据页不在内存中时，普通索引更新类操作的提速器。注意：只能作用于普通索引，不能作用于唯一索引**。

原理上面已经简单分析过，就是通过**减少磁盘的随机读**来提升写的性能。

上面是通过 `insert` 来举例，`update` 和 `delete` 有点点区别。

如果`update` 和 `delete`  中的 `where` 条件走普通索引的话，由于需要获取受影响行数，所以还是需要先把数据页读到内存中，然后直接更新内存，这里就用不上 change buffer 了。

如果 `where` 条件走主键索引或其他普通字段(其实最终也是走主键索引树)，并且需要更新普通索引字段的话，通过主键索引树获取受影响行数，对普通索引的更新就直接写到 change buffer 中了，这种情况是能用 change buffer 来提速的。



#### 怎么保证数据被正确更新？

上面说到，普通索引的更新写到 `change buffer` 中就结束了，那后续的查询是怎样的？

还是对应到上面的2种情况：目标数据页在不在内存中。

- 如果目标数据页在内存中，意味着更新操作是直接更新的内存。那此时内存中的数据页一定是更新后的数据，虽然磁盘上还是老的数据，所以直接从内存返回即可；

- 如果目标数据页不在内存中，需要先把数据页从磁盘读入内存，然后应用 `change buffer` 中的操作日志，生成一个正确的版本后返回，这个过程称为 **`merge`**。

  > `merge` 的时机：
  >
  > - 查询时。这时会把目标数据页从磁盘读到内存中；
  > - 作为后台任务定期运行。`innodb_io_capacity` 和 `innodb_io_capacity_max` 用于设置 Innodb 后台任务(刷脏页、merge)的 IOPS，可调整该数值来控制 merge 的频率；
  > - 在崩溃恢复期间，会从系统表空间(ibdata1)中读取 `change buffer`，然后当把数据页从磁盘读到内存中时，会进行 merge；
  > - 重启后；
  > - `slow shutdown` 时。可通过 `--innodb-fast-shutdown=0` 开启 `slow shutdown`。





#### 怎么保证更新不丢失？

如果写完 `change buffer` 后断电了或意外宕机了，重启后 `change buffer` 和数据会丢失吗？

不会。`change buffer` 也会被记到 `redo log` 中(**redo log 中包含了数据页的变更和change buffer的变更**)，回想之前讲过的两阶段提交协议，`redo log` 和 `binglog`  落盘才代表事务成功提交。所以，如果一个事务已提交，则代表 `change buffer` 已经写到 `redo log` 中，且 `redo log` 已落盘，崩溃恢复时会根据 `redo log` 来恢复 `change buffer`。



#### 适用场景

`change buffer` 简单来说就是把对普通索引的更新缓存了下来，然后在适当的时候进行 `merge`。所以在 `merge` 之前， `change buffer` 中记录的变更越多，收益就越大。

所以对于**<u>写多读少</u>**类业务，数据页在写完之后马上被访问到的概率很小，也就是说不会马上进行 `merge`，这种情况下 `change buffer` 的效果最好。比如账单类、日志类等。

相反，如果是写后马上进行查询的业务，由于马上要访问数据页，会立即触发 `merge`。这种情况不仅不会减少随机IO的次数，反而会增加维护 `change buffer` 的代价，反而起到了副作用，这种情况下可以关闭 `change buffer` ：

```sql
show global variables like 'innodb_change_buffering';
```

默认值为 `all` ，设为 `none` 即可关闭 `change buffer`。



#### 官方文档

[InnoDB Change Buffer](https://dev.mysql.com/doc/refman/8.0/en/faqs-innodb-change-buffer.html#faq-innodb-change-buffer-merging)

------

<br/><br/><br/>







# 给字符串字段创建索引的几种方法

1. 直接创建完整索引，这样可能比较占空间

2. 创建前缀索引

   即可以只定义字符串的一部分作为索引

   ```sql
   alter table user add index index_email(email(6));
   ```

   优点是：节省空间

   缺点是：

   - 可能会增加额外的扫描次数

     ​	比如执行这样一句查询：

     ```sql
     select id,name,email from user where email='zhangssxyz@xxx.com';
     ```

     ​	对于完整索引，在 `email` 索引树上定位到 `zhangssxyz@xxx.com`，然后回表取出对于记录即可，只需扫描一行；

     ​	对于前缀索引，在 `email` 前缀索引树上定位到 `zhangs`，回表判断 `email` 是否等于 `zhangssxyz@xxx.com`，是的话将记录加入结果集，继续在 `email` 前缀索引树上遍历下一条记录，再回表判断，重复此过程，直到遍历的下一条记录不等于 `zhangs`。所以前缀索引可能会增加记录的扫描行数。

     

     **如何优化？**

     关键在于**增加前缀的区分度**。区分度越高，过滤掉的记录就越多，需要回表的次数就越少。

     可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。

     先统计索引上有多少个不同的值：

     ```sql
     select count(distinct email) as L from user;
     ```

     再依次选取不同长度的前缀来对比区分度：

     ```sql
     select 
       count(distinct left(email,4)）as L4,
       count(distinct left(email,5)）as L5,
       count(distinct left(email,6)）as L6,
       count(distinct left(email,7)）as L7,
     from user;
     ```

     数值越大表示对应长度的前缀区分度越高，效果越好。

     

     

   - 不能使用覆盖索引

     比如执行这样一句查询：

     ```sql
     select id,email from user where email='zhangssxyz@xxx.com';
     ```

     因为只需要查询 `id,email`，对于完整索引来说，使用覆盖索引即可，不需要再回表；

     对于前缀索引，则必须要回表判断 `email` 的值，即便前缀索引包含了全部字段(email(18)，假设`email` 有18个字符)，因为系统不确定前缀索引的定义是否截断了完整信息。

   

3. 倒序存储

   对于前缀区分度不够好的情况，可以考虑使用倒序存储。

   比如身份证，同一个区域内的身份证前面几位都是相同的，如果按照上面的方法建立前缀索引，这个前缀的长度可能会比较长。

   这时可把身份证倒过来存，因为身份证的尾部都是不同的，区分度足够高，查的时候转换一下：

   ```sql
   select field_list from t where id_card = reverse('input_id_card_string');
   ```

   这时为身份证建立前缀索引需要的长度就会短很多，具体多长可通过上面的方法来确定。

   缺点：**不支持范围查询**，因为是倒序存储，没办法按顺序遍历。

   

4. 新增一个 hash 字段

   专门新增一个 hash 字段用来做索引。

   > 相较于倒序存储查询性能相对稳定一些，因为倒序存储毕竟还是前缀索引，或多或少还是会增加扫描行数。而crc32(或其他哈希算法)冲突的概率总体还是非常小的，可认为每次查询的平均扫描行数接近1。

   ```sql
   alter table t add id_card_crc int unsigned, add index(id_card_crc);
   ```

   每次插入新纪录的时候，都用 `crc32()` 计算出一个哈希值填到这个新字段中。

   查询的时候计算一下，**同时因为 `crc32` 会有冲突(虽然概率也非常小)，所以还需要在 `where` 中校验一下原值**：

   ```sql
   select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'
   ```

   这时索引的长度就只有4个字节(crc32的长度)，相较于身份证长度大大减少了。

   缺点：和倒序存储一样，**不支持范围查询**，因为哈希字段对应的原值完全是无序的，没办法在哈希索引上按顺序遍历。

   

   
