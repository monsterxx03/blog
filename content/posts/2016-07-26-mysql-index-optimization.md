---
title: MySQL 索引优化
author: will
type: post
date: 2016-07-26T16:13:54+00:00
url: /2016/07/26/mysql-index-optimization/
categories:
  - tech
tags:
  - Database
  - MySQL

---
什么是索引,索引怎么建这些基本的就跳过不谈了,整理一些前段时间优化线上 SQL 查询时碰到的一些问题. 主要解决下面几个问题:

  * 建立索引怎样选择合适的列.
  * 怎样让 SQL 能有效利用索引.
  * 如果对 SQL 效率进行评估(即设置索引前后是否真的有性能提升).

<!--more-->

## 索引的开销

我们都知道利用索引能提高查询效率, 但也不能盲目得随意添加索引, 索引也是有成本的.

### 磁盘空间开销

对于下面的表:

    CREATE TABLE `User` (
        `id` bigint NOT NULL AUTO_INCREMENT,
        `name` varchar(255),
        `email`  varchar(255),
        `location` varchar(255),
        `gender` tinyint,
        `age` int,
        `time_created` bigint,
        `time_removed` bigint,
        PRIMARY KEY (`id`)
    )ENGINE=InnoDB;
    

如果我们要添加一个索引 **KEY &#96;idx_email&#96; (&#96;email&#96;)** 加快对email列的查询, MySQL 实际上做的是把 email 字段复制一份来构建索引列, 此外对每个索引, 主键也会附加在后面, 所以每个索引的实际磁盘开销是被索引列+主键, 因此设计表的时候最好选择一个简单又短的主键列.

### 索引维护开销

这里讨论的索引只限于 innodb 的 B 树索引, 不包括 hash 索引和 fulltext索引.

B 树是平衡树的一种, 数据存储在 leaf node 上, 在 INSERT, DELETE, UPDATE 的时候需要移动 leaf node 来保持树的高度. 这就是索引维护的成本, 因此过多的索引会增加 INSERT, DELETE, UPDATE时候的开销, 对写多于读的表慎用索引.

## 选择合适的列建立索引

### LeftMost

索引查询遵循 LeftMost 原则, 假设上面的 User 表有索引:

         Key `idx_user` (`name`, `email`, `time_removed`)
    

以下查询是能利用到索引的

SELECT * FROM User WHERE name=&#8221;zhang&#8221;

SELECT * FROM User WHERE name=&#8221;zhang&#8221; and email=&#8221;z@g.com&#8221;

SELECT * FROM User WHERE name=&#8221;zhang&#8221; and email=&#8221;z@g.com&#8221; and time_removed=0

如果查询 where 语句中只包含 email 或 email + time\_removed, 或 time\_removed 是不会走索引的.

如果 where 语句中包含 name + time\_removed, 还是会走索引, 但起作用的只是 name 部分, 并没法根据索引来跳过 time\_removed 列.

假如索引是:

        KEY `idx_location_gener` (`location`, `gender`, `time_removed`)
    

按照 LeftMost 原则, 如果我们的 where 中只包含 location + time\_removed, 最后的 time\_removed 并不会被用到.

这里有个小技巧, 由于 gender 只有男女两种(好吧, 通常情况下&#8230;), 我们可以这么做来利用到最后的 time_removed 字段: `SELECT * FROM User WHERE location="China" and gender=0 or gender=1 and time_removed=0`

就是在这个查询中我们本来就是要选择所有的gender, 通过这种方式来让查询强制走索引.

可惜的是如果写 `gender in (0, 1)`, 并不会work,原因是 in 属于range query, 后详.

### 一个索引还是多个索引?

对于 User 表, 如果我们有如下的查询语句:

    SELECT * FROM User where name="Zhang" and email="zhang@gmail.com"
    

那我们应该是为name和email列各自建立一个索引,还是建立一个索引同时包含name和email呢?

通常情况下我们都应该选择建立一个 **KEY &#96;idx\_name\_email&#96; (&#96;name&#96;, &#96;email&#96;)**, 有以下原因.

由于主键在每个索引上会被复制一份,建立一个索引显然比两个索引要省空间, 当然也省了索引维护的成本.

还有是 MySQL 索引遵循 LeftMost 原则, 所以建立在 (name, email) 上的查询可以用在 where 条件中只包含name的情况, 和where 条件中同时包含name 和 email的情况, 只带email的 where条件则无法使用该索引.

那如果我在 name 和 email 上各自建立一个索引, MySQL 能不能同时用到这两个索引呢? 不一定. 取决于 query plan 的评估结果, 5.6.7之前的版本中如果 where 条件中如果某个列使用了 range 查询那就不会做 index merge. 由于两个 index 的查询结果做归并也是有成本的, 显然不如建立一个多列的索引. index merge的过程比较复杂而且不确定性很高,不建议依赖它来做查询优化.

### 对区分度高的列做索引

对于上面的 User 表, 如果在业务系统中,用户删除是一个极少会发生的行为, 可以考虑不对 time_removed 做索引.

比如大部分的查询可能都只包含name 或 email, 根据这两个条件查询出的数据集已经很小了(通常 email 还是 unique 的), 再加上一个time_removed列其实意义不是很大.

## 让查询能有效利用索引

前面已经讲过 LeftMost 原则, 但写查询的时候也有些 case 需要注意, 否则索引可能并不会生效.

### range query

MySQL 中的 range query 包括 `>`, `<`, `>=`, `<=`, `!=`, `in`, `between`, `like`, `is null`, `is not null`.

一些要注意的地方:

  * 对于 `in`, 如果集合里只有一个元素, 比如 `user_id in (1)`, 是会被优化成`user_id = 1`的,不算 range query.
  * 对于`like`, 只有like的的部分不以`%`开头才算 range query,否则是个完全没有优化的全表扫描操作.

如果对被索引字段做 range query, 索引扫描会停在第一个 range query 上, 后面的索引列不会被用到.

还是 User 表, 假设我们有索引**KEY idx\_age\_location(age, location)**

查询是: `SELECT * FROM User where age>13 and location="China"`

`age > 13` 就是 range query, 所以所以扫描就会停在age字段, location并不会被用到.

对于这个查询, 合理地索引应该是**KEY idx\_location\_age(location, age)**

但如果我们的查询中只有 `age > 13`, 没有location 字段, 这个新的索引又不会被用到, 这种情况需要对age单独做索引.

一般原则是: 被索引列中只能有一列含有范围查询,且该列要放最后.

### order by

有查询: `SELECT * FROM User where location="China" order by time_created desc limit 100`

我们想取 location 为 China 的最近创建的100个用户.

假设我们只有索引 `KEY idx_location (location)`, 对该 select 语句做 explain, 会发现 `rows` 字段远大于100. 理想情况我们希望该字段只扫描最后我们需要的100行数据.

需要建立索引 `KEY idx_location_recent(location, time_created)`, 此时再做explain, rows就是100行了.

要注意的是order by 也遵循 LeftMost 原则, 如果索引是 (location, gender, time_created), 对于我们上面的查询则不会对order by做优化.

对 order by, 还有个可以利用的小 trick, 前文说过每个索引都会将主键拷到该索引的最后, 所以 order by 其实是可以利用这个主键的.

由于 User 表的主键 id 列是个自增字段, id 大的 time_created 一定大, 所以我们完全可以 `order by id desc limit 100`, 这样就只需要对一个location 列做索引了.

## SQL profiling

通过对 query 做精确计时,我们可以知道更改索引后效果是否明显.

通过 mysql client 连接上 server:

`set profiling=1` , 打开profiling, 这个变量是 session level的,只对当前连接起作用.

然后执行一些query. 再执行: `show profiles`, 可以得到:

        1   0.01275200  SELECT  * FROM `User`  WHERE `user_id` =  AND `time_removed` = 0   ORDER BY `id` DESC  LIMIT 100 
    
        2   0.01018250  SELECT  E * FROM `Notification`  WHERE `user_id`=816121  AND `time_removed` = 0 AND `time_modified` >= 0  ORDER BY `id` DESC  LIMIT 100 OFFSET 0
    

可以看到很精确的 sql 计时.

`show profile for query 2`, 可以得到某一条查询在各个步骤的耗时. 输出结果如下:

        starting    0.000111
        checking permissions 0.000035
        Opening tables  0.000041
        init    0.000051
        System lock 0.000036
        optimizing  0.000038
        statistics  0.000144
        preparing   0.000051
        Sorting result  0.000033
        executing   0.000038
        Sending data    0.000043
        Creating sort index 0.009226
        end 0.000037
        query end   0.000041
        closing tables  0.000044
        freeing items   0.000174
        cleaning up 0.000041
    

使用 `show profile all for query 2`, 可以得到各个步骤在cpu, block io上更详细的统计信息.

关闭 profile: `set profiling=0`

### Check row examined

在 MySQL 的慢查询日志中有 rows\_sent 和 rows\_examined, 优化的目的一般都是要使这两个值尽量接近, 两个值完全相等就意味着为了得到最后的查询结果,没有扫描多余的列, 这是最佳情况.

有没有办法在慢查询之外的地方知道我们的 query 扫过了多少行呢?

可以通过检查 MySQL 几个 status 变量间接得知道 examine 的行数.

`show status like "Handler_read_%"`, 输出:

            Handler_read_first  0
            Handler_read_key    0
            Handler_read_last   0
            Handler_read_next   0
            Handler_read_prev   0
            Handler_read_rnd    0
            Handler_read_rnd_next   0
    

完全解释清这几个变量的含义比较麻烦, 举个例子, 对于 User 表, 我们假设有索引: `KEY idx_location (location)`

假设我们的查询是 `SELECT * FROM User WHERE location="China" order by time_created desc 100`

根据前文所述, 这里的 time_created 不在索引列中,所以query其实会扫描 location=&#8221;China&#8221; 的所有行, 对整个结果做排序后再取top 100.

首先将所有的状态变量归0: `flush status`, 然后执行query, 再查看状态变量, 看到 `Handler_read_next` 是188, 多于设的 limit 100, 188也是对location=&#8221;China&#8221; 做count的结果.

假设索引不变, 将条件变为 `WHERE location="China" order by id desc limit 100`, 先`flush status`, 再执行query, 查看变量,会发现 `Handler_read_next` 变成了0, `Handler_read_prev`变成了99, 这表明查询中的order by id得到了优化.

## 总结

实际业务中,可能并没法让每条查询都完美hit到索引,怎么选择索引需要根据实际 query 做取舍.