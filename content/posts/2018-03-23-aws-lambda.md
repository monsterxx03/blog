---
title: "AWS lambda 的一些应用场景"
date: 2018-03-23T17:40:54+08:00
categories:
  - tech
tags:
    - AWS
    - lambda
---

这几年吹 serverless 的比较多,  在公司内部也用 lambda , 记录一下, 这东西挺有用, 但远不到万能, 场景比较有限.

lambda 的代码的部署用的 [serverless](https://serverless.com) 框架, 本身支持多种 cloud 平台, 我们就只在 aws lambda 上了.

我基本上就把 lambda 当成 trigger 和 web hook 用.

## 和  auto scaling group 一起用

线上所有分组的机器都是用 auto scaling group 管理的, 只不过 stateless 的 server 开了自动伸缩, 带状态的 (ElasticSearch cluster, redis cache cluster) 只用来维护固定 size. 

在往一个 group 里加 server 的时候, 要做的事情挺多的, 给新 server 添加组内编号 tag, 添加内网域名, provision, 部署最新代码.

这些事都用 jenkins 来做, 但怎么触发 jenkins job 呢?

让 auto scaling group 在 scale out 的时候发一个 notification 到 SNS topic, lambda function 订阅这个 topic, lambda function 从 SNS message 中解析出 新 server 的 instance id, 调用 jenkins 的 api 触发 provision.

provision 完成后，等 service 起来了, ALB 的 health check 就能过了, 用户的流量就过来了, 目前 provision 的过程大概要 2 mins.

![asg](/posts/images/asg.png)

Notes:

- 如果把一台 auto scaling group 内的机器设置成 sandby 状态， 在把它放回去的时候, SNS topic 里的 Action 字段和新起 server 时候是一样的 LAUNCH, 如果不想有重复的 provision, 需要自己判断下 description 字段.
- 因为 jenkins 部署在内网, lambda function 需要设置好相应的 vpc 和安全组 才能访问.
- auto scaling group scale in 时候的关闭策略, 我选的是 newest server, 虽然 provision 的代码每次改动都很小心, 但万一有问题, 需要把新加的 server 销毁, 这样的策略比较方便.


## 对 log 做 audit

Redshift 的 audit log 会被定时上传到 S3, 用 lambda 做了一个 S3 upload object 的 trigger, 每次上传会检测日志内容, 如果有登陆失败, 或账号创建删除等事件的 log 时候, 发邮件出来.


## 做 webhook

和第三方服务有一些交互, 他们会定时 post 一些数据过来, 需要把这些数据导入 redshift.

结合 api gateway 和 lambda, 可以很方便地做 webhook, 收到 post 消息, 后导入 redshift.

一般一个 lambda function 只处理一件事， 如果有好几个 webhook 举要设置好多个 lambda, 管理起来比较麻烦, 可以用 [service-wsgi](https://github.com/logandk/serverless-wsgi), 这样一个 lambda function 可以做成一个 python wsgi app, 就能直接用 flask 之类的框架来做路由, 内部根据 url path 处理多个 webhook. 

## 做 external health check

lambda 支持像 cronjob 一样定时运行, 配合 aws 的多 region, 可以很方便的从全球各个地区对 server 做 health check.


## lambda 的一些限制

- 本地临时目录最多写入 512MB
- 单个 function 内存最多分配 3008 MB
- FD 最多 1024 个
- 最长运行时间 300s (所以不适合当作cron 来作 ETL)
- 单账户最多并发 1000 (太低了...不适合跑业务)
- 单个 deployment package 未压缩前大小不能超过 250MB, 压缩后不能超过 50MB.

...

限制还是挺多的, 实际使用下来, lambda 适合跑一些简单, 功能独立的代码, 复杂业务逻辑放上去, debug 会很麻烦, 设置一致的本地开发环境也很麻烦.
