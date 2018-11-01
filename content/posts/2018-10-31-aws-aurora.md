---
title: "AWS Aurora DB"
date: 2018-10-31T15:23:45+08:00
draft: true
categories:
    - tech
tags:
    - aws
    - mysql
    - database
---

最近在把部分用 RDS 的 MySQL 迁移到 aurora 上去, 读了下 aurora 的 paper, 顺便和 RDS　的架构做些对比.

## Paper notes

- 存储计算分离
- redo log 下推到存储层
- 副本: 6 副本 3 AZ(2 per az), 失去一个 AZ + 1 additoinal node 不会丢数据(可读不可写). 失去一个 AZ (或任意2 node) 不影响数据写入.
- 10GB 一个 segment, 每个 segment 6 副本一个 PG (protection group), 一 AZ　两副本.
- 在 10Gbps 的网络上, 修复一个 10GB 的segment 需要 10s.

MySQL 一个应用层的写会在底层产生很多额外的写操作，会带来写放大问题:

![](/posts/images/aurora-mysql-replication.png)

redo log 用来 crash recovery, binlog 会上传 s3　用于 point in time restore.


在 aurora　里，只有 redo log 会通过网络复制到各个 replica, master 会等待 4/6 replicas 完成 redo log 的写入就认为写入成功 (所以失去3副本就无法写入数据了). 其他副本会根据 redo log 重建数据.

只同步 redo log 可以大大提高 MySQL 的写能力, 论文中给的数据是,和直接的基于 EBS mirror 的 MySQL 相比, aurora 的TPS 是35 倍, 每个事务产生的 IO 减少了7.7 倍.

每个 redo log 有一个单调递增的 LSN (Log Sequence Number). 一个 PG 里的副本通过 gossip 协议相连, 如果有一个副本缺少部分 redo log, 可以直接从同 PG 内其他副本同步过来.

## RDS MySQL

RDS　的　multi AZ 结构

![](/posts/images/rds-MultiAZ.jpg)

在两个　AZ 里各其一台 EC2 作为　MySQL server, 其中一台为 primary, 通过底层的 EBS　做同步.

应用通过一个 DNS 域名连接 MySQL, 总是指向当前的 primary, 当 failover 发生时, DNS 自动切换到 standby 的 MySQL, 这个域名的 TTL 很短(4s), 客户端重连后就到了新的 MySQL.

这种结构的问题:

- multi AZ 中的另一台 server 无法作为 read replica, 但要付和 primary server 一样的钱.
- 需要 read replica 的话,要另起一台, 它和 master 之间是通过 binlog replication 同步的, 当 master 在做很耗时的 DDL 操作时, 所有更改都会 pending, 等 master 做完 DDL, read replica 才能开始做 DDL, 然后才继续接受数据.
- EBS 的大小要自己指定, 不够要手工扩容.
- HA 固定在两个 AZ. read replica 最多 5 台
- 单机 size 16TB (EBS 的最大 size)

## Aurora MySQL

![](/posts/images/aurora-MultiAZ.png)

三个 AZ　中 6 副本, 每个 AZ 两副本.

存储计算分离的结构, 集群中 server　的数目不受副本数约束(rds　的 multi az 中就只有 2 副本 2 server, read replica　没法作为 failover 对象). 最多可有 15 个 replicas,　都可作为 failover 对象.


最坏情况可以承受失去一整个 AZ　外加一个额外的副本, 此时整个集群会处于 read only,　但数据不会丢. 失去副本数<3,并且没失去 整个AZ　的情况下, 不会影响读写.


aurora 提供三种访问的　endpoint 域名:

- cluster endpoint: 永远指向当前的 master server, failover 发生时, cluster endpoint 自动刷新.
- reader endpoint: 会自动在 read replicas 之间做 load balancer, 如果只有一台 master,　没　read replicas, reader endpoint 会指向 master,　此时也是可以写入的.
- instance endpoint: 每台 server 单独的 endpoint, 用于访问特定的 server, 但这个就没法做 failover　了. 

和 RDS MySQL 比的优势:

- 没有浪费的 standby server, 所有 read replicas　都能用来做 failover　对象,　也可读
- 副本和 master 之间在存储层同步, DDL　不会导致 read replica 的同步延迟, 
- 磁盘空间每 10GB 自动增长, 最多 64TB, 只收一份的钱(rds　的　read replica　磁盘要另收费).

参考:

- https://aws.amazon.com/cn/blogs/database/amazon-rds-under-the-hood-multi-az/
- https://www.allthingsdistributed.com/files/p1041-verbitski.pdf
- https://www.percona.com/blog/2018/07/17/when-should-i-use-amazon-aurora-and-when-should-i-use-rds-mysql/
