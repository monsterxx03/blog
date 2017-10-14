---
title: MySQL partition table
author: will
type: post
date: 2017-04-05T16:23:32+00:00
url: /2017/04/05/mysql-partition-table/
categories:
  - tech
tags:
  - Database
  - MySQL

---
# Overview

MySQL has buildin partition table support, which can help split data accross multi tables,
  
and provide a unified query interface as normal tables.

Benefit:

  * Easy data management: If we need to archive old data, and our table is partitioned by datetime, we can drop old partition directly.
  * Speed up query based on partition key(partitoin pruning)

Limit:

  * For partition table, every unique key must use every column in table&#8217;s partition expression(include primary key)
  * For innodb engine, paritioned table can&#8217;t have foreign key,and can&#8217;t have columns referenced by foreign keys.
  * For MyISAM engine, mysql version <= 5.6.5, DML operation will lock all partition as a whole.

<!--more-->

# Partition live db table

MySQL have several different partition type: range, list, key, hash&#8230;

For my live db table, I just want to partition it by date, one table per month.

So I choose to use range partition by the `time_created` column (it&#8217;s a bigint).

origin table schema:

    CREATE TABLE `Record` (
        `id` bigint NOT NULL, AUTO_INCREMENT,
        `data` text DEFAULT '',
        `time_created` bigint NOT NULL,
        PRIMARY KEY (`id`)
    )
    

After partitioning, I only want to keep recent 3 months data. If I delete rows in original table first, it will take a lot of time, also rebuild primary key and partitioning will cost many time as well.

So I decide create a new partitioned table first, and select needed data from original table to it, and do a rename operation at last.

Steps:

  * create new partiton table `Record_new` (with corrent primary key and partition info)
  * insert into `Record_new` select * from Record where id > xxx and id < yyy. (copy needed data to new table, should do it chunk by chunk since the number is huge).
  * rename table `Record` to `Record_old`, `Record_new` to `Record`.

With this flow, we can have minimum impact on live server, and can cancel at any time during copy data.

Create sql for new partition table:

    CREATE TABLE `Record` (
        `id` bigint NOT NULL, AUTO_INCREMENT,
        `data` text DEFAULT '',
        `time_created` bigint NOT NULL,
        PRIMARY KEY (`id`, `time_created`)
    )  PARTITION BY RANGE(time_created)
         (PARTITION p_2016 VALUES LESS THAN (UNIX_TIMESTAMP('2017-01-01')),
         PARTITION p_2017_01 VALUES LESS THAN (UNIX_TIMESTAMP('2017-02-01')),
         PARTITION p_2017_02 VALUES LESS THAN (UNIX_TIMESTAMP('2017-03-01')),
         PARTITION p_2017_03 VALUES LESS THAN (UNIX_TIMESTAMP('2017-04-01')),
         PARTITION p_2017_04 VALUES LESS THAN (UNIX_TIMESTAMP('2017-05-01')),
         PARTITION p_default VALUES LESS THAN MAXVALUE);
    

`p_default` is used to hold data if no suitable partition is available, usually, it should be null. I use a montly cronjob to reoriganize it to genearate a new partition:

    ALTER TABLE Record REORGANIZE p_default into 
    (PARTITION p_2017_05 VALUES LESS THAN (UNIX_TIMESTAMP('2017-06-01')),
     PARTITION p_2017_06 VALUES LESS THAN (UNIX_TIMESTAMP('2017-07-01')));
    

# Partitoin management

Check how many rows every partitoin have:

    select PARTITION_NAME,TABLE_ROWS from information_schema.partitions where table_schema='test_db' and table_name='test';
    

Drop partition:

    ALTER TABLE Record DROP PARTITION p_xxx;