---
title: "数据库系统杂谈"
date: 2018-08-05T22:18:16+08:00
draft: true
categories:
    - tech
tags:
    - Database
---

最近在重看 `<Designing Data-Intensive Applications>` , 书真的是好书, 深度广度刚刚好,第一遍算是开开眼, 第二遍依旧收获良多. 结合自己工作经历, 谈谈用过的几种数据库系统, 顺序按照工作中实际接触的顺序来, 大体可以体现一家公司业务从简单到复杂过程中会遇到的问题. 用词尽量白话一点, 写给一般开发人员看, 本文只讨论数据的存储, 关于分布式/scaling 的话题只简单带过, 一文写不完.

## OLTP 系统

最简单的系统前面几台 web 服务器后面直连一个数据库, 可能是 MySQL 或 PostgreSQL, 互联网业务特点, 读多写少, 写操作多是短事务. 这时候我们谈数据库系统性能的时候比较关心 QPS (query per second), TPS (transaction per second), 数据落到磁盘上的时候我们用 IOPS (input/output operations per second) 来衡量系统负载是否饱和.

互联网业务特点, 高并发, 低延迟, 处理这种业务的数据库系统分类上我们叫 OLTP (Online transaction processing).

数据库把数据存到磁盘上时, 以 page 为单位存储, page 一般比较小, 比如 MySQL 是 16KB, page 在磁盘上怎么组织? 最常见的方案是 B 树. B 树用在数据库系统里有以下优点:

- 保证数据有序存储.
- 自平衡, 树的高度为 O(log n), 一般系统3到4的高度就能存储所有数据了, 所以在树中查找数据也很快.
- 适合高效的范围查询和前缀匹配.

用 B 树构建索引时候, 可以把 key(索引值) 和 value (实际row) 存在一起, 这种索引叫 clustered index (聚族索引), 查询时候找到 key 就直接找到了 value. MySQL 中的主键索引就是 clustered index. 也可以在索引中只保存 value 的 reference id, 通过 reference id 再找到对应 row, 这种叫 sendary index (二级索引).

## OLAP

## Hash 索引 

##  LSM 树

## B 树
