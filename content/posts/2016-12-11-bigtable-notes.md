---
title: Bigtable notes
author: will
date: 2016-12-11T16:20:24+00:00
url: /2016/12/11/bigtable-notes/
categories:
  - tech
tags:
  - bigtable
  - distribute
  - paper

---
杂乱笔记，辅助读paper.

<!--more-->

BigTable 是一种多维稀疏有序map， 通过 row key, column key, timestamp 进行索引。value 是不可分割的字节数组。

比如用来存储网页的webtable，url是row key, 网页的一些属性作为column，内容存储在contents column下，timestamp是网页被抓取的时间。

**Rows**

table中的row key是任意字符串，长度最多64KB，通常为10 -100 bytes. 对一个row key的读或写是原子的。bigtable 按row key的字典序对数据作排列，一张表的row range动态作partition，每一片row range叫做tablet, 是作分布和load balancing的基本单元。 存储网页的webtable的 row key就是url的倒序(com.google.www&#8230;), 这样来自同一个domain的网页就能按顺序存储在一起，对特定domain进行分析的时候速度就很快。

**Column familiaries**

column key 被分组成 column family,属于同一个family的数据通常是相同类型的，同一个family内的数据会被压缩在一起。column family的数目尽量少（最多几百），而且很少改变。

column key的命名 family:qualifier, 比如languange 和 anchor都是family。

**Timestamp**

bigtable 中的每个cell都能包含同一数据的多个版本， 版本通过timestamp索引，timestamp是64 bit integers. 可以由bigtable指定或client自行指定。 不同的版本通过timestamp降序排列，保证最新的数据能被先读。

保存数据版本有两种策略: 1. 只保留最新的 n 个版本 2. 只保留够新的版本(在最近7天内被写过的数据)

**BigTable 的building block**

数据文件和log存储在GFS中，BigTable的数据文件以SSTable 格式存储，SSTable 提供 persistent, ordered immutable map from keys to values.

SSTable 内部由连续的block组成，通常是64KB, 在block的最后会存储一个block index, 用来定位block，当SSTable 被打开的时候， index会被加载进内存。一次查找只需要一次disk seek， 先在内存中对index作二分查找，然后从磁盘中读取对应的block。

Chubby 为BigTable 提供高可用的持久化 lock 服务。一个Chubby 集群由5个replica 组成，其中一个会被选举成master，并接受请求。当多数的replicas存活并能互相通信的时候，chubby就可用。Chubby 使用Paxos 来保证replicas的一致性。Chubby的namespace 包含目录和小文件，每个目录和文件都能作为一个锁，对文件的读写是原子的，每个chubby的client和server维护一个session，如果session过期，client就失去所有的锁，chubby的client也能在chubby的文件和目录上注册callbacks，当session变化或过期的时候发出通知。

Bigtable 用chubby来保证同一时间只有一个master，存储data 的bootstrap location，tablet server 的生命周期管理。存储bigtable的schema信息，存储访问控制列表。如果chubby不可用，bigtable也无法使用。

**BigTable的实现**

client library， 一个 master server， 多个tablet server. tablet server 可以从集群中动态得添加删除。

master 负责将tablets 分配到tablet server，检测tablet的变化，负载均衡，回收GFS中的垃圾文件。还处理表和column family的创建。

每个tablet server管理一部分tablets（一台server通常管理10到1000个tablet）。tablet server处理对tablet的读写，也对过大的tablet作切割。

bigtable中每张表由许多tablet组成，每个tablet包含row key在某个范围内的数据，每张表开始只有一个tablet，超过一定大小，自动分裂成多个tablets, 默认这个大小是100 到200MB。

**Tablet location**

tablet 的位置信息用三层结构存储: chubby file -> root tablet(1st metadata tablet) ->other metadata tablets

chubby中的文件存储了root tablet的位置， root tablet存储了所有metadata tablets的位置，每个metadata tablet 又存储了用户的tablet 的位置。 root tablet是系统中第一张metadata table，从不分裂，这样可以保证系统中的继承结构不超过3层。

**Tablet**

每个tablet在同时只被分配到一个tablet server上。master负责track table 和tablet server 的分配关系，包括哪个tablet还没被分配，master 向tablet server 发起一个tablet load request来执行分配。

bigtable 使用 chubby来track tablet server。每个tablet server 在启动的时候都从chubby处得到一个exclusive lock，这个lock file 保存在一个特定的目录中。bigtable的master 通过监控这个目录来发现tablet server。如果失去自己的exlcusive lock(由于network partition等原因，失去了和chubby的session), tablet server 会停止工作。当tablet server 停止的时候，会释放自己的lock， master 很快会重新分配它的tablets。

tablet的持久化存储在GFS中，update操作存进commit log中（可用于undo），内存中保留一块sorted buffer （memtable）用于保存最近更新操作，更久的记录存储在连续的SSTable中。 tablet server recover的时候，会将所有SSTable的indices加载进内存，还会通过commit log重建memtable。

当tablet server进行写操作的时候，commit之后，也会写入memtable。 read 操作是在SSTable和memtable的merge view上进行的，都以字典序存储，所以读操作很高效。

写操作越多，memtable的size 越大，超过一定尺寸后，会创建一个新的memtable，旧的table会转化成SSTable，存进GFS, 这个过程叫minor compaction。 major compaction会将所有的SSTable重写成一张SSTable，因为通过minor compaction产生的SSTable 会包含删除信息，需要清除这些数据。

压缩： tablet的压缩在SSTable 的每个block上单独进行，这样读取文件的时候不需要解压整个表。压缩分两步进行，第一步使用Bentley and Mcllroy&#8217;s schema 在一个比较大的window上压缩一些经常出现的string pattern. 第二步在16KB的window上利用一种快速算法寻找重复pattern。webtable的测试中，压缩比达到了10:1， gzip对html的压缩通常是3:1 或 4:1，
