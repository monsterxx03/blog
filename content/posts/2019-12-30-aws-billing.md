---
title: "聊聊 AWS 的计费模式"
date: 2019-12-30T12:08:47+08:00
categories:
  - tech
tags:
  - aws
  - server-infra
---

网上经常有人诟病 AWS 的计费模式复杂, 喜欢国内那种打包式的售卖方式, 这个可能受限于每个公司的财务流程, 预算制定方式, 合不合国情,本文不讨论.
仅从开发者的角度介绍下 AWS 部分常用 service 的计费方式.

PS: 那些为了蹭一年 free plan 然后抱怨什么偷跑流量, 偷偷扣费的大哥就省省吧, AWS 根本不是给个人用的, 老老实实用 lightsail 得了. 

## EC2

EC2 的价格是最复杂的, 一台 EC2 instance 的价格组成:

- instance fee, 实际支付的是 CPU+RAM 的费用
- EBS fee, server 的根分区都是 EBS volume, 按 EBS 计费(31GB 的 volume 使用 1 小时, 和 1GB 的volume 使用 744 小时价格相同).
- data transfer fee, 这部分组成比较复杂，简单讲, 入流量免费, 出流量按 GiB 计费, 如果出流量是到 AWS 其他 region, 价格比一般公网便宜, 在内网传输流量, 同一个 available zone 是免费的,
跨 az 收费.  
- EIP fee, 每台 instance 挂一个 EIP 是免费的(eip 使用状态不收费, 闲置收费), 超过一个 eip, 每个按小时再收费.

instance fee 又分 on-demand/resersed/spot instance.

- On demand: 比如 linux, us-east-1, m5.large(2 vCPU, 8GiB memory) 0.096$/h, linux 实例不足一小时按秒付费(最少1min), 其他需要
license 的系统(rhel, sles, windows...) 按小时收费. 
- Spot instance: 使用 aws 的闲置资源,价格比 on demand 便宜很多, 最少能到1/10, m5.large 等常用实例价格基本为 on-demand 的 1/3,
但 spot instance 随时会被关闭(aws 需要资源的时候), 会提前 2min 通过 cloudwatch 发送一个通知让你做清理工作, 比较适合跑 hadpoo 等
task 可以被重调度的工作, 或作为 k8s 的 worker node 跑 stateless 的服务.
- reserved instance: 预留实例, 预定 1 年或 3年的使用期, 可以选择预付部分或全部款项, 预留时间越长,预付金额越多越便宜. reserved instance 又分两种:
standard 和 convertible.
   -  standard class 的instance 可以在同 family 内免费互转, 比如 2 台 m5.large 可以转成 1 台 m5.xlarge, 1台 m5.xlarge 可以拆成两台
m5.large, 但不能跨 family 转(比如不能转c5,r5系列的), standard instance 的剩余日期可以在 aws 的 reserved instance market 上按月卖.
   -   convertible 类型的单价比 standard 贵大概 10%, 好处是可以跨 family 互转, 转换规则是:只补差价, 不退款.
比如 c5.large 比 m5.large 便宜, 拿一台 c5.large 换一台 m5.large 只要补足差价就行, 一台 m5.large 换 一台 c5.large 不行, 只能换两台 c5.large 然后补足差价.
convertible class 的 instance 不能卖.


ec2 instance 有个属性叫 tenancy, 默认情况下是和其他人共享宿主机的, 如果有业务需求(比如一些 compliance), tenancy 设成 dedicated, 就能独享宿主机,
使用 dedicated instance 需要注意的有:

- 只要在一个 region 里有一台 running 的 instance 是 dedicated, 就要额外收 `2$/h` 的的 dedicated fee, 不叠加(你有 100 台也是收 `2$/h`)
- 同类型 instance, dedicated 比默认 tenancy 贵 10% 左右.
- 不是所有类型的 instance 都支持 dedicated, 比如 t 系列的不行.
- reserved instance 也有 tenancy 属性, 注意必须和你实际使用的 instance match 才会生效.
- spot instance 也能请求 dedicated 的, 但我尝试的时候几种常用类型都申请不到capacity.

鉴于 reserved instance 的计费方式比较复杂, 前不久 aws 推了个 saving plan, 实际单位价格和 reserved instance 是一样的, 不过不是按 instance 台数计费, 而是承诺每小时的最低使用费,
详细的可以看: https://aws.amazon.com/blogs/aws/new-savings-plans-for-aws-compute-services/

## RDS

RDS 算是一个典型, 其他托管的 elasticsearch, redis cluster, emr 计费方式都和它部分重合,只是更简单一点. 详细讲一下吧.

RDS 分两大类, 一类是传统的基于底层 EBS 复制提供 HA 的 RDS(multi AZ), 还有 aurora(基于 redo log 复制), 两者的详细区别, 以前写过: https://blog.monsterxx03.com/2018/10/31/aws-aurora-db/

### 传统 RDS 的计费

- instance fee: 不开启 multi az 就没有 ha, 按列表价收费, 价格比同算力 ec2 贵了80%左右(啧啧), 开 multi az 就 x2. 可以和 ec2 一样买 reserved instance, 只是没有 convertible class, 也不能卖.
- storage fee: db 的 storage size 由 EBS 决定, 和普通的 EBS 一样可以选 gp2, provisioned io 等类型, 但价格稍贵一点, 如果开了 multi az, 再 x2.
- backup fee: backup 的 size <= storage size 时免费, 超出部分按 GiB 收费.
- data transfer: 同 ec2.

multi az 里 standby 的 server 不能用作 read replicator, 需要 replicator, 要另外起, 和 master 之间通过标准 binlog 同步.

### Aurora 的计费

Aurora　只支持 MySQL 和 PostgreSQL, 价格组成:

- instance fee: 同算力价格比 rds 还要贵一点点, 同样可以买 reserved instance. aurora 的 read replicator 可以同时用来做 failover 对象, 和 master 通过底层 redo log 同步.
- storage fee: aurora 的 storage size 按照 10G 一个 segment 增长，不需要像传统 rds 那样预先分配很大的 storage(gp2 类型 ebs `iops = size *3`), 存储上比较省钱, 但每百万 io 要额外收费.
如果使用跨 region 的 aurora 实例还要收跨 region 的 io 钱.
- backup fee: 同传统 rds, 价格更便宜.
- backtrack: 使用 binlog 来回档数据, 按每百万 change 收钱.
- data transfer: 同 ec2.

只看价格, 在存储上 aurora 比传统 rds 要省很多钱, 分歧关键在 io 上, aurora 每一百万 io requests 要 `0.2$`, 挺贵的. RDS 的 iops, 如果是 gp2 类型的 ebs 的话 = size * 3. 500GiB 的数据库, 
它的 base line iops 就是 1500, 但 size 小于 1TB 的 gp2 类型的 ebs 有 brust mode, 当实际 iops 小于 1500 的时候, ebs 会积累 I/O credits, 上限是 5.4 million, 在高峰时候可以让 iops 上限临时增高到 3000,
最多可持续 5.4M / 3000 = 0.5hour. 这对于中小型数据库的突发负载, 比如 ETL, 很有用. 详情可以看: https://aws.amazon.com/blogs/database/understanding-burst-vs-baseline-performance-with-amazon-rds-and-gp2/

RDS 切换到 aurora 能不能省钱? 不一定, 只能分析业务的工作负载来估算.

## DynamoDB


以前 DynamoDB 里一张表的读写能力由 WCU/RCU 决定, 以前也写过: https://blog.monsterxx03.com/2017/12/15/dynamodb/.

WCU 决定表的每秒写入能力, RCU 决定每秒读能力, WCU 价格是 RCU 5 倍. 可以买 reserved WRC/RCU 来减少 cost.

这种计费方式叫 provisioned capacity, 后来支持了 on-demand 方式, 按每百万读写 reqeust 收费, 写价格依旧是读的5倍. 每张表可以在两种计费模式之间切换.
on-demand 当然比 provisioned capacity 单位价格贵不少, 但它对突发的负载比较友好, provisioned capacity 即使开了 auto scaling 也需要一段时间反应过来.

一些额外 feature 的计费方式略过不提(backup, DAX, stream, storage), 那些都比较好理解. 


## 其他

aws 的计费非常详细, 善用[价格计算器](https://calculator.s3.amazonaws.com/index.html)来规划开销, 另外也可以用一些第三方的服务来做 aws 的 cost 分析, 比如: http://cloudhealthtech.com/.
其他一些 service, 比如 s3, cloudfront, glacier, ELB/ALB/NLB 以后有空再谈吧.
