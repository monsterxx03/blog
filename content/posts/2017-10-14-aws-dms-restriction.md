---
title: "AWS DMS notes"
date: 2017-10-14T22:33:36+08:00
categories:
  - tech
tags:
  - AWS
  - DMS
---

AWS's DMS (Data migration service) can be used to do incremental ETL between databases. I use it to load data from RDS (MySQL) to Redshift.

It works, but have some concerns. Take some notes when doing this project.

## Prerequisites

Source RDS must:

- Enable automatic backups
- Increase binlog remain time, `call mysql.rds_set_configuration('binlog retention hours', 24);`
- Set `binlog_format` to `ROW`.
- Privileges on source RDS: `REPLICATION CLIENT `, `REPLICATION SLAVE `, `SELECT` on replication target tables

## DDL on source table

Redshift has some limits on change columns:

- New column only must be added in the end
- Can't rename columns

So for DDL on source MySQL, you can't add columns at non end postition, otherwise data in target table will corrupt. I disabled ddl changes target db:

        "ChangeProcessingDdlHandlingPolicy":{  
            "HandleSourceTableDropped":false,
            "HandleSourceTableTruncated":false,
            "HandleSourceTableAltered":false
        },
    
If source table schema changed, I just drop and reload target table on console.

## Control write speed on Redshift

Since Redshift is an OLAP database, write operation is slow and concurrency is low, streaming data directly will have big impact on it.

And we have may analysis jobs running on redshift all the time, directly streaming will lock target table and make my analysis jobs timeout.

So I need to batch apply changes on DMS. Follow settings need to tweak in task settings json:

        {
            "TargetMetadata": {
                ...
                "BatchApplyEnabled": true   # enable batch mode
            },
            "FullLoadSettings": {
                ...
                "MaxFullLoadSubTasks": 1    # control concurrent full load tables
            },
            "ChangeProcessingTuning": {
                "BatchApplyPreserveTransaction": false  # I don't have foreign keys on source db    
                "BatchApplyTimeoutMin": 3600,   # control write frequency to redshift
                "BatchApplyTimeoutMax": 3600,
                "BatchSplitSize": 0
                ...
            }
        }

## Problem with table transform rules

DMS support transform rules in table mapping, you can `add-prefix`, `remove-prefix`... on a table.

But it doesn't support multiple transformations on same entity, means you can't add a prefix and a sufix for a same table, and its doc doesn't mention it.

After contact AWS support engineers, they confirmed it, and saied:

    they are working on implementing the order of transformations(cascading transformations) or validating customer's input (e.g. we can't have more than one transformation for the same table/column).

## Problem with terraform

I use terraform to manage AWS infrastruction (v0.10.7), one problem is after I created DMS tasks via terraform, do `terraform plan` again will still see changes on `replication_task_settings` option,  

Problem is `CloudWatchLogGroup` and `CloudWatchLogStream` in task settings json don't work. Even you specific it,
AWS still creates a log stream with random name for you.

One workaround is go to AWS console, get the `CloudWatchLogGroup` and `CloudWatchLogStream` settings, modify your local one.

Related issue: https://github.com/terraform-providers/terraform-provider-aws/issues/1513