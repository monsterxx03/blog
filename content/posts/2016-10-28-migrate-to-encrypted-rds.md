---
title: Migrate to encrypted RDS
author: will
type: post
date: 2016-10-28T16:17:30+00:00
url: /2016/10/28/migrate-to-encrypted-rds/
categories:
  - tech
tags:
  - AWS
  - Database
  - MySQL

---
最近公司在做 HIPAA Compliance 相关的事情，其中要求之一是所有db需要开启encryption.

比较麻烦的是rds 的encryption 只能在创建的时候设定，无法之后修改, 所以必须对线上的db 做一次 migration.

<!--more-->

主要有以下两种方法:

  1. 通过RDS 的snapshot 做迁移。
  2. 通过mysqldump 手工迁移。

准备工作, 在原db上执行:

  * show variables like &#8216;sync_binlog&#8217;, 确保设置成1, 否则binlog可能会不全(通过db parameter group 设置).
  * call mysql.rds\_set\_configuration(&#8216;binlog retention hours&#8217;, 24) , 默认rds 不保留binlog 文件, 让它保留24 小时，方便我们迁移。

## Snapshot 方式

  1. 将线上应用停止
  2. 将db 设置为read only 
      * FLUSH TABLES WITH READ LOCK;
      * SET GLOBAL read_only = ON;
  3. 获取当前数据库的binlog 文件和位置: `show master status`
  4. 对当前 db 打snapshot
  5. 恢复线上数据库, 和应用程序 
      * SET GLOBAL read_only = OFF;
      * UNLOCK TABLES;
  6. 将完成的snapshot 执行 Copy, Copy 的时候可以选择使用 KMS 进行加密。
  7. 使用完成的加密 snapshot launch 新的 db instance, 注意 security group 和db parameter group 只能在db 创建后修改，为了让 db parameter group
  
    生效，还要重启一次新db.
  8. 登陆新db, 设置replication 对象为原 db: `call mysql.rds_set_external_master('<src_db_host>', 3306, '<repl_user>', '<password>', 'log_file', position, 0)`.
  
    log_file, 和 postition 由第3步获得，最后一个参数0表示不使用ssl.
  9. 开始replication, `call mysql.rds_start_replication`
 10. 在新 db 上用 `show slave status`, 检查 `Seconds_Behind_Master` 字段，如果变成0，表示slave已经追上了master.
 11. 准备执行切换，再次停止所有的应用，然后停止replication: 
      * call mysql.rds\_stop\_replication;
      * call mysql.rds\_reset\_external_master
 12. 将应用程序的db url 指向新的db后重新启动应用。

## mysqldump方式

mysqldump 步骤和snapshot 差不多，只不过把创建snapshot的过程换成mysqldump 出文件，恢复过程变成先手工launch一台
  
加密的新db，然后在把文件倒入。

详细的过程可以参考 AWS 的这篇文档:

http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.NonRDSRepl.html

## 尽量减少server 的 down time

以上两种方式都需要将将live db 设置成 read_only, 以获得准确的 binlog 文件和position. 在做snapshot 和dump的过程
  
中需要停机较长的时间， 但这两种方式都比较保险。

还有一种方式在snapshot的基础上做一些改进，可以只stop，start 一次应用完成迁移, 中间不需要等。

在创建snapshot 的时候，不停应用，也不设置read_only. 然后从加密的 snapshot 创建新的db.

在原db 和新 db 上执行， `show binary logs`, 获取文件列表,

原db 上的输出:

    mysql-bin-changelog.138398  3221126
    mysql-bin-changelog.138399  4136769
    mysql-bin-changelog.138400  74066
    mysql-bin-changelog.138401  3637534
    mysql-bin-changelog.138402  6071405
    

新db上的输出:

    mysql-bin-changelog.138398  3221126
    mysql-bin-changelog.138399  4136769
    mysql-bin-changelog.138400  74066
    mysql-bin-changelog.138401  35261
    mysql-bin-changelog.138402  1768
    

其实就是找到binlog 文件 size 出现分歧的地方，这里是138401. 说明新db 的binlog是停在了这个文件。

我的做法是直接从138401 文件，position 0 开始做sync.

这样的做法会导致部分binlog在新db上被重复执行, 但我们线上的sql DML操作只有update/insert, 没有delete, 且insert 的表
  
都有 unique key 约束。

当插入重复数据的时候，replication 会停止， 报`show slave status` 能看到 duplicate entry error 的错误，此时可以直接 `call mysql.rds_skip_repl_error`, 来跳过错误往下执行。

同一row 被update多次也没有问题，反正最终值还是对的。

如果有delete语句就不能这么做了，当然如果你能忍受少部分数据不一致，也可以从下一个binlog 文件开始做sync, 否则只能乖乖停机按标准流程走。