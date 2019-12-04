---
title: Redshift as data warehouse
author: will
date: 2016-07-16T16:11:39+00:00
url: /2016/07/16/redshift-as-data-warehouse/
categories:
  - tech
tags:
  - AWS
  - Database
  - Redshift
  - glow
  - data-infra

---
Glow 的 server infrastructure 全部搭建在 AWS 上，一般要选择一些基础服务的时候，总是先看 AWS, 只要功能和成本符合要求，不会特意选择开源方案。

数据仓库我们选择了 AWS 的 Redshift.

在一年多的使用过程中 Redshift 的性能和稳定性都不错, 当然也有一些坑, 这里整理下在使用 redshift 的过程中的一些经验和遇到的坑.

<!--more-->

## Columnar storage

就是常说的列式存储, 这里只谈应用场景, 理论详情可以看: http://docs.aws.amazon.com/redshift/latest/dg/c\_columnar\_storage\_disk\_mem_mgmnt.html

常用的开源数据库 MySQL 和 PostgreSQL 都是传统的行式数据库, 和 columnar storage 在使用场景下都有什么区别呢?

SQL 查询语句一般会分成两种:

  * OLTP (On-line Transaction Processing)
  * OLAP (On-line Analytical Processing)

通常线上业务逻辑就是是常说的 CRUD 都是 OLTP 型 SQL, 对访问速度和并发量的要求很高, 但是 SQL 本身都不会很复杂. 一般也会利用索引来快速定位目标行.

而用来做数据挖掘和 BI 分析的 SQL 一般都是 OLAP 的. 逻辑很复杂, 大量使用 aggregate function 和 window function, 几乎都是查询操作, 要对某一条精确记录进行修改删除的情况很少. 结果需要扫过大量的数据集后进行运算得到.

Columnar storage 适合 OLAP 的场景. 做分析的时候我们只对特定的列感兴趣(这也和我们的表设计有关, 下面会提到), 数据库只会 scan 在 SQL 中指定过的列.

Columnar storage 带来的另一个优势是高压缩比, 可以为每个字段指定不同的压缩算法, 它也提供了方法从当前数据中自动分析出最合适的压缩算法. AWS 在文档中提到他们的客户普遍的压缩比在1: 3 左右, 实际的使用过程中, 我们的数字要高于这个, 最开始使用的时候在1: 7, 最近的的压缩比也在1: 4.8 左右.

一般文件系统的 block size 是 4KB, 而一些用来处理大数据的文件系统会将 block 设置得比较大, 比如 HDFS 是64MB, 而 redshift 是1MB, 在一个 block 中只会存储一个column 的数据,
  
这样相同类型的数据可以在一个 block 中得到很高的压缩比.

## Scalability

Redshift 可以从一个最小 160 GB 的 SSD node 起步,一路扩展到上百 TB. 需要做的也就是在 console 上点几下.

Cluster 会在扩容开始和结束的时候各重启一次, 这时候所有的 DB Connection 会断开. 在扩容过程中 cluster 是 readonly 状态.

但是扩容过程很慢,上次从7节点扩到9节点花了13个小时.

## Price

Redshift 现在有四种类型:

![price][1]

使用 reserved instance 的话, 价格可以更省, 详细的价格对比可以看 https://aws.amazon.com/redshift/pricing/ , 三年期的合约折扣有75%, 一年期有42%

顺便提下, 使用下面的 SQL 可以查看系统实际的磁盘空间:

         select owner as node, diskno, used, capacity
            from stv_partitions
        order by 1, 2, 3, 4;
    

在 2TB node上的输出:

    node | diskno | used  | capacity
    -----+--------+-------+----------
      0  |    0   | 0     |  1906185
      0  |    1   | 0     |  1906185
      0  |    2   | 0     |  1906185
    

2TB 的 HDD node 实际有3块 1861GB 的磁盘, 共 5584.5GB

160GB 的 SSD node 实际给的是两块186GB 的磁盘, 共 372GB

多出来的空间用户不可用, 应该是用来做 replication 的.

## Table design

用来做分析的 metrics 数据曾经是存在 MySQL 里的, 分 DB 做了 horizon sharding, 并按天做了 vertical sharding, 使用起来比较麻烦, 只能调用一些固定的 python 函数, 不能直接写 SQL, 更复杂
  
的分析只能另外写代码, 也没办法和其他的业务表做 join. 并且数据增长非常快, RDS 和 EBS 成本增长很快, 而现在一个月的数据量比迁移前一整年的量都多. 一个
  
比较复杂的4周 retention 分析, 曾经在 MySQL 上要花半小时(没法直接做, 需要 pull 数据出来再用 python 进行运算), 相同的逻辑现在用纯 SQL 实现,只需要 20s 左右, 这个过程中大概会处理6亿行
  
左右的数据.

我们以事件的方式定义数据, 比如一个用户注册的事件:

        USER_SIGNUP = {
            'source': string,
            'status': string,
            'time': int
        }
    

这个事件就有三个字段 source, status, time, 而整个系统中已定义的事件目前接近2000种, 总共有近400个不同的字段定义, 怎么存?

曾经在 MySQL 里的做法是, 只在 table 里定义通用的 str\_1, str\_2, int\_1, int\_2 这样的字段,然后再定义各个事件的时候把每个
  
字段的 mapping 也写上, 比如:

        USER_SIGNUP = {
            str_1: source,
            str_2: status,
            int_1: time
        }
    

问题有:

  * 容易写错, 每个事件的 mapping 是由开发人员各自定义的, 常会出现把 int 写到 string 里去的情况
  * 添加新字段麻烦, 是按天分表的, 有的事件字段特别多,会导致总字段数变多, 但大部分又是空着的, 浪费空间.
  * 数据库中数据不直观,必须用代码做一次 translate 才能查询.

在 redshift 中的做法很简单粗暴, 把400个字段都放一张表里, 然后按月做sharding.

为了对使用者屏蔽 sharding 细节, 会再创建 union view:

        create view log_latest_two_month_view as
            select * from logs_2015_01
        union all
            select * from logs_2015_02
    

会创建最近2个月,3个月&#8230;等等常用时间范围的 view, 使用者可以像在操作一张表一样操作 view, 如果使用者很清楚时间范围,也可以去查特定的表.
  
这些 view 用 cronjob 在每月的月初刷新.

添加字段的方式, 我现在的做法是将代码中定义的所有字段抽出来, 再从 redshift 的系统表中查出所有的当前字段, 找出两者的 delta, 然后生成相应的
  
Alter SQL 语句, 批量 patch 到所有的表上去.

### distkey & sortkey

设计表的时候 distkey 和 sortkey 需要在一开始就想好, 无法在创建表后更改.

由于 redshift 是一个分布式数据库, 所有的数据自动 distribute 在所有的 nodes 上, 它有三种分布方式, 详见文档: http://docs.aws.amazon.com/redshift/latest/dg/c\_choosing\_dist_sort.html

我们使用的是按 key 做分布, 所以需要选择一个 distkey, 一般选择最有可能要和其他 table 做 join 的字段, 比如 _user_id_, 如果同一个用户的数据全部在一个 node 上, 查询过程中就不需要重分发数据, 否则 redshift 会自动迁移数据到一个节点上, 带来额外的网络开销
  
(用 explain 可以查出 SQL 执行过程中是否发生了数据重分发: http://docs.aws.amazon.com/redshift/latest/dg/t\_explain\_plan_example.html)

一般数据库中索引的原理都是将需要索引的字段复制一份构建一个 B tree 或 B+ tree, 而 redshift 不支持索引, 使用 sortkey 的概念.

sortkey 定义了在导入数据的时候按什么顺序给数据排序, 前面说过1个 block 只存一个 column 的数据, 指定过 sortkey 的话,在 block 的 metadata 中就会存数据的 min 和 max 值,
  
当把 SQL 中把 sortkey 加入 where 条件时, redshift 可以把查询条件和 min,max 值做比较, 决定能不能快速跳过这个 block.

一张表可以设置多个 sortkey, 目前 sortkey 有两种 `compound sortkey` 和 `interleaved sortkey`. 刚开始 setup 的时候, interleaved sortkey 这个特性还没有发布,
  
所以一直用得都是 compound sortkey, 这个类似 MySQL 的 secondary index, 遵循 LeftMost 的规则.

interleaved sortkey 在官方的这篇博客中有比较详细的解释: https://aws.amazon.com/blogs/aws/quickly-filter-data-in-amazon-redshift-using-interleaved-sorting/

按我的理解, compound sortkey 如果在只使用 leading key 的情况下, 性能比 interleaved sortkey 好, 但如果做一些 ad hoc 查询, filter 条件通常不固定, 可能是很多字段的排列组合,
  
这时候使用 interleaved sortkey 可以取得比较好的平均速度.

一张表可以有多个 sortkey, 但不能同时使用 compound sortkey 和 interleaved sortkey.

## Performance optimize

Redshift 中性能的关键是选择合适的 distkey 和 sortkey, 通过 explain 看是否发生 redistribute 来确定 distkey 的选择是不是合适,
  
skip 的 block 够不够多来确定 sortkey 的选择合不合适, 但也有其他的 feature 来进一步调优.

### Workload Management

redshift 中有 query queue 的概念, query queue 控制了并发上限, 默认只有一个 query queue, 并发度是5, 所以默认情况下你只能同时 run 5个 query.

通过给不同类型的 query 创建不同的 queue, 可以提高系统的并发性能, 比如将耗时长的 query 和短的 query 放到不同的 queue 中去, 再给两个 queue 分配不同
  
的内存比, 这些都能在 WLM 中设置.

一些限制:

  * 整个 cluster 的并发总上限只能50.
  * 用户总共能定义8个 query queue
  * 官方推荐单个 query queue 的并发设置在15以下, 过大的并发数会增加资源竞争限制总体的吞吐量.

### Slot

slot是memory和cpu的资源的单位，通过增加query使用的slot数目，可以增大query的可使用的内存和 cpu 资源.

简单得说一个 query 能使用的 slot 数目越多, 速度越快, 临时改变 slot 数目的方法:

            set wlm_query_slot_count to 10;
            select ....;
            set wlm_query_slot_count to 1;
    

### Vacuum

vacuum 命令有两个作用: 排序和空间回收.

在导入数据的时候, 如果数据不是按照 sortkey 导入的, 可用 vacuum 来对数据进行重排序, 这样可以大大优化查询速度, 但是有个问题导致我们现在并没法使用 vacuum 做优化,后面会提.

如果用 delete 删除了数据, redshift 中的空间并不会释放, 需要做一次 `vacuum delete only` 来回收, 如果要清空一张表的数据不要用 delete,
  
用 `truncate table` 或 drop 重建.

## Loading data into cluster

Redshift 使用 `COPY` 命令来导入数据, 数据源可以是 s3, dynamodb, ssh host, 格式可以是 csv, avro, json, 源数据也可以用 bzip2,gzip,lzop 的方式进行压缩.

但是如果目标 table 指定了 sortkey, 那 redshift 会在导入过程中对数据进行排序, 排序过程会占用额外的磁盘空间. 而官方并没有给出一个明确的公式来让你提前确定需要预留的空间,
  
这个问题多次给他们提过 ticket, 但都没有很明确的回复.

只看过一个说法是要预留 raw data 2.5倍以上的空间, 实际使用中, 需要的空间比这个数字只高不低, 而且两者并不是线性相关的.

解决方法: 要么提前 resize 一个比较大的 cluster, 导完再 resize 回去, 要么将数据分批进行导入.

PS: 用 `vacuum` 命令进行重排序的时候也会碰到这个问题导致磁盘剩余空间爆掉.

## Some limit

Redshift 也有一些硬性限制, 需要在采用之前就清楚是否适合自己的应用场景.

在 workload management 中提到过 cluster 的最高并发只有50, 这种数据仓库服务的直接使用者一般都是公司内部的分析人员, 线上业务不应该直接在 redshift 中运行 SQL, 要使用一般
  
也会通过 cronjob 或队列服务来异步得获取数据.

column 数目限制, 由于我们的做法是将所有的事件的字段全部定义在一张表中, 所以受限于总字段数, 单张表的字段上限是 1600.

char 类型只接受单子节的UTF-8字符，上限是127（即ascii字符集），varchar类型可以接受多字节的UTF-8字符，最多4字节，用copy导入数据时，如果不符合此规则会失败，可以给copy加上ACCEPTINVCHARS, 来跳过有问题的字段.

## DeadLock

对某表做查询的时候hang住了， 但是在stl_locks中没有看到记录，可以通过以下步骤debug：

        select oid from pg_class where relname='tablename' 获取表的oid
        select * from pg_locks where relation='{oid}' 获取锁的详细信息,其中包含产生锁的pid(相当于用户session), 和xid(事务id)
        select * from svl_statementtext where pid='{pid}' or xid='{xid}' 获取导致锁表的详细 SQL 语句.

 [1]: https://blog.monsterxx03.com/wp-content/uploads/2017/05/price.png
