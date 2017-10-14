---
title: MySQL innodb buffer pool
author: will
type: post
date: 2016-07-16T16:07:14+00:00
url: /2016/07/16/mysql-innodb-buffer-pool/
categories:
  - tech
tags:
  - Database
  - MySQL

---
最近在对公司的 MySQL 服务器做性能优化, 一直对 innodb 的内存使用方式不是很清楚, 乘这机会做点总结.

在配置 MySQL 的时候, 一般都会需要设置 _innodb\_buffer\_pool_size_, 在将 MySQL 设置在单独的服务器上时, 一般会设置为物理内存的80%.

之前一直疑惑 MySQL 是怎么缓存数据的(不是指query cache), 直觉应该是LRU, 但如果 query 一下从磁盘上读取大量的数据的话(全表扫描或是 mysqldump), 是不是很容易就会把热数据给踢出去?

<!--more-->

## Buffer Pool 的 LRU 机制

innodb buffer pool 在内存中其实是一个 list, 维护数据的确是用 LRU, 但是做了点优化.

buffer pool 将数据切割成两块, 头部是 young page list, 尾部是 old page list. 两者的空间比例默认是5: 3, 可以通过 _show variables like &#8220;innodb\_old\_blocks_pct&#8221;_ 查看, old block的比例即为 37%.

当数据第一次被从磁盘上读取, 要加入 buffer pool 时, 会先放在 old page list 的头部, 此时如果 old page list 满了,则会把它尾部的数据踢出去. 当数据在buffer pool中被再次访问的时候, 如果它在 old page list 中, 则会被转移到 young page list 的头部(如果数据已经在 young page list 中, 则只有它和头结点的距离超过一定长度时才会被转移到头部).

通过这种机制, 可以有效得避免临时的大量数据读取冲刷掉 hot page, 这些一次性的数据如果没有被访问的话,很快会被从old page list 中踢出去.

## 查看 buffer pool 的使用状况

使用 _show engine innodb status_, 可以获取 innodb 的运行信息, 查看其中 _BUFFER POOL AND MEMORY_ 一节, 可以看到 innodb buffer的使用情况.

示例:

    BUFFER POOL AND MEMORY
    ----------------------
    Total memory allocated 11948523520; in additional pool allocated 0
    Dictionary memory allocated 881470
    Buffer pool size   712576
    Free buffers       7638
    Database pages     676994
    Old database pages 250018
    Modified db pages  22961
    Pending reads 4
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 86737875, not young 2068799422
    188.89 youngs/s, 5482.68 non-youngs/s
    Pages read 390735994, created 1284180, written 17312013
    972.19 reads/s, 0.30 creates/s, 36.63 writes/s
    Buffer pool hit rate 975 / 1000, young-making rate 4 / 1000 not 141 / 1000
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 676994, unzip_LRU len: 0
    I/O sum[374688]:cur[1688], unzip sum[0]:cur[0]
    

几个需要注意的参数:

_Total memory allocated_: innodb 已分配的总内存数, 单位 bytes

_Buffer pool size_: buffer pool 大小, 单位是 pages, MySQL 的 page 默认是 16KB, 这里 pool 的总大小就是 10G 多.

_Free buffers_: 还没使用的 buffer size, 单位也是 pages.

_Database pages_: 总共已使用的 page 数目.

_Old database pages_: old page list 中的 page 数目.

_youngs/s_: 每秒对 old page list 的访问次数(不是page的个数), 这种访问会就会把page从 old list 里移动到 new page list 的头部.

_evicted without access_: 放进 buffer pool 一次都没访问过就被踢出去的 page 数目. 如果发生了全表扫描, 这个数值可能就会临时变高.

## 补充

因为多线程对 buffer pool 进行访问的时候会发生资源竞争, MySQL 支持对 buffer pool 进行 sharding, 从而提高并发性能, 参数是_innodb\_buffer\_pool_instances_.

每个 instance 的 size = _innodb\_buffer\_pool\_size/innodb\_buffer\_pool\_instances_

官方推荐 sharding 后单个 instance 的 size 不要小于 1G*B. 比如我现在 pool 的总 size 是10 GB, 我设置的 instance 个数就是8.

上面 engine status 只能看到 buffer pool 的汇总情况, 如果想看每个 instance 的详情,
  
可以从 _performance_schema_ 里拿:

    SELECT * FROM INFORMATION_SCHEMA.INNODB_BUFFER_POOL_STATS
    

References:

https://dev.mysql.com/doc/refman/5.6/en/innodb-buffer-pool.html
  
https://dev.mysql.com/doc/refman/5.6/en/innodb-performance-midpoint_insertion.html
  
https://dev.mysql.com/doc/refman/5.6/en/innodb-buffer-pool-monitoring.html
  
https://dev.mysql.com/doc/refman/5.6/en/innodb-multiple-buffer-pools.html