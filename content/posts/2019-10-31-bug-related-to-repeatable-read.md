---
title: "老代码里和 MySQL 的事务隔离相关的一个bug"
date: 2019-10-31T11:45:44+08:00
categories:
- tech
tags:
- mysql
---

这两天在调试代码的时候, 发现 db 层的代码在每次把 connection 放回 db pool 的时候,即使之前执行的是 select 语句,也会 rollback 一下, 
这代码很古老, 我也不知道为啥, 尝试把 rollback 去掉, 结果单元测试挂了一堆, 大多都是数据不一致的问题, debug 了一下, 最后发现这坑还挺大的.

为什么去掉 select 的 rollback 后会出现数据不一致?

pymysql 默认关闭了 autocommit, connection A 进行 select 之后, 其实 MySQL 内部为 select 也开启了一个 transaction(Repeatable Read),
可以通过 `SELECT * FROM information_schema.innodb_trx\G`  查看.

所以当 connection A 先 select 一次, connection B 在 transaction 内更新数据并 commit, connection A 再次 select (之前的 transaction 并没 rollback 或 commit),
得到了老的数据. MySQL 在 repeatable read 下, 为了 consistent read 会使用一个 snapshot, 时间是第一次 read 发生的时间: https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_consistent_read

connection A 执行一次 rollback 就能看到新的数据, 我去掉了 rollback 自然就挂了......

不想在每条 select 后都发一句 rollback, 只要打开 autocommit 就行了. 但这份老代码因为某些原因, connection 的建立过程包装了一下,是 lazy 的, 当一个 connection 调用 `begin()` 函数尝试开始一个 transaction 的时候, 底下真正的 dbc 还没建立, `begin()` 函数有可能空跑了一下,啥都没干. 如果开了 autocommit, 会导致一些 write 没法 rollback.

陈年老代码就是坑, 彻底改需要对这整套逻辑重构了.

