---
title: Use redshift spectrum to do query on s3
author: will
date: 2017-07-21T03:10:58+00:00
url: /2017/07/21/use-redshift-spectrum-to-do-query-on-s3/
categories:
  - tech
tags:
  - AWS
  - Redshift
  - spark

---
# 使用 redshift spectrum 查询 S3 数据

通常使用 redshift 做数据仓库的时候要做大量的 ETL 工作，一般流程是把各种来源的数据捣鼓捣鼓丢到 S3 上去，再从 S3 倒腾进 redshift. 如果你有大量的历史数据要导进 redshift，这个过程就会很痛苦，redshift 对一次倒入大量数据并不友好，你要分批来做。

今年4月的时候， redshift 发布了一个新功能 spectrum, 可以从 redshift 里直接查询 s3 上的结构化数据。最近把部分数据仓库直接迁移到了 spectrum, 正好来讲讲。

## 动机

Glow 的数据仓库建在 redshift 上， 又分成了两个集群，一个 ssd 的集群存放最近 4 个月的数据，供产品分析，metrics report, debug 等等 adhoc 的查询。4个月之前的数据存放在一个 hdd 的集群里，便宜容量大，查询慢。

但是时间长了 hdd 的集群也是有扩容需求的，而使用频率又实在是不高，其实很浪费, 这就是迁移到 spectrum 的动机。

## 使用 Spectrum

Redshift spectrum 底层其实是基于 AWS 的另一个服务 athena 的。athena 是个 Presto 和 Hive 杂交产物， DDL 用 Hive 语法， 查询用的 sql 由 Presto 支持, 感觉怪怪的，这里不多展开讲 athena, 知道 redshift spectrum 其实是通过 athena 对接的 s3 就行了。

### 在 redshift 中创建 external schema

    create external schema spectrum_schema from data catalog 
    database 'spectrum_db' 
    iam_role 'arn:aws:iam::123456789012:role/MySpectrumRole'
    create external database if not exists;
    

这里的 data catalog database `spectrum_db` 其实就是 athena 里的，创建完后可以在 athena 的面板上看到， iam_role 需要提前创建， 用来给 redshift 授权访问 athena.

如果你已经有了一个 hive 的 metastore, 也可以绕过 athena 直接对接 hive, 详情看文档: http://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-schemas.html

### 在 external schema 中创建 external table.

external table 只能创建在 external schema 中，可以外链到 s3, 支持 partition, 每个 partition 就是 s3 上的文件夹。

    create external table spectrum.logs(
    user_id integer,
    ...
    )
    partitioned by (_dt date)
    stored as parquet
    location 's3://bucket/partitions/';
    

存储在 s3 上的数据我用了支持 columnar 的 parquet 格式，spectrum 支持: `avro`, `parquet`, `rcfile`, `sequencefile`, `textfile`. 注意它的 textfile 并不是 csv, 只是简单用 delimiter 分割数据，不支持 escape。

partition 需要一个个手工添加:

`Alter table spectrum.logs add partition(_dt='2017-01-01') location 's3://bucket/partitions/_dt=2017-01-01/'`

需要注意的是这里的 partition key `_dt` 是一个虚拟的 column, 你可以在查询的时候用它来过滤数据，但它不能出现在 DDL 的 column 中.

查询就很简答啦: `select count(*) from spectrum.logs where _dt='2017-01-01'`

## 使用 parquet 存储数据

parquet 是一种支持 columnar 的格式，在 hadoop 生态圈很常用, 一方面可以在查询的时候只扫描需要的列，大大提高查询速度，另外，spectrum 是按照查询的时候扫描过的数据量收费的(5$/TB), 使用 columnar 的格式可以大大降低成本.

原始数据在 redshift 里，要使用spectrum，需要转存到 s3 上. redshift 里导出的数据是 gzip 压缩的 csv 格式，这里需要做一次转换. 我起了一个 emr 的集群，用spark来转换的，非常方便。

核心步骤就两步，读取 csv 得到 spark 的 dataframe 对象，转存成 parquet 格式到 s3.

    schema = StructType(
    [
        StructField('user_id', LongType(), True),
        StructField('event_name', StringType(), True),
        ....
    ]
    )
    df = spark.read.schema(schema).option("sep", "|").csv("s3a://bucket/raw/*")
    df.write.option("compression", "gzip").parquet("s3a://bucket/parquet/000.parq")
    

几个要注意的点:

  * parquet schema中定义的数据类型要和 redshift external table 定义的数据类型匹配，不然查询会出错, 不支持隐式转换。
  * redshift unload 出来的数据，boolean 字段会用`f`, `t`, 表示，spark 不认，所以 schema 对 boolean 字段要先定义成 string,再强制转换成 boolean: `df = df.withColumn("col_name", df["col_name"].cast("boolean"))`
  * 如果csv 数据字段中有换行符，需要用 &#8220;&#8221; 引起来，不然即使你 escape 了，spark 的 csv parser 也处理不了.
  * spark 转存 parquet 的时候, 默认使用 snappy 压缩数据，snappy 压缩解压速度都最快，但压缩比比较低，如果你的查询不是很频繁，可以换成 gzip, 体积更小.

## Performance

使用 parquet 后，体积进一步减小，原本一个 4 GB 的 gzip 文件，转存成 parquet(压缩还是 gzip), 体积就只有 2 GB.

对一张有10亿行的表中某个字段做个简单的 group by count 操作，在原来的 HDD 集群上要 40s, 在 spectrum 里只要 6s.

综上，把 redshift 里的历史数据搬到 s3 上，用 spectrum 查询，可以减少成本，提高查询速度，还省了 ETL 的步骤.
