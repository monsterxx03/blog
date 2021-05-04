---
title: "重构推送服务"
date: 2020-04-07T10:54:12+08:00
categories:
- tech
tags:
- code-infra
---

最近对业务里发送 apple APNS, google FCM 部分的代码进行了重构, 抽出了一个单独的 service, 本文记录下整个过程.

## 存在的问题

我们有好几个 mobile app, 每个 app 会有一套对应的 server 端 service 做业务逻辑, 因为历史原因, 每个 service 里面其实有很多重复代码, 大多只是一些配置和错误处理上有差异.
给 app 发推送是个典型, 原来的做法是当要发推送的时候, 往 python 的 celery 队列里扔一个 task, 由 celery 异步得去发.

有如下问题:

- celery 性能不佳, worker class 用的是 gevent, 但在高峰时候任务队列里还是有大量 pending, 代码上之前做过多次优化, 但收效甚微.
- 当 pending 了大量发推送的 task 之后, 会导致其他异步 task 跟着延时, 造成用户体验上的一些问题.
- 历史代码的问题, 每个 app 里对接 APNS/FCM 都有一套单独的代码, 在部分错误处理和统计 metrics 上有差异.

## 解决方案

- 问题1, 目前已经迁移到 k8s, 其实可以无脑开 HPA, 做 auto scaling, 来提高发送效率. 问题是 python 性能实在太差, 峰值时候估计会 scale 到原先的好多倍, 而 celery 里
除了发推送还有很多 task 有 db 操作, scale 过多, 会带来其他问题, 那是另一个问题, 不想在当前这个问题里解决.
- 问题２, celery 里可以设 routing, 简单说可以用单独的 celery instance 来专门处理推送相关 task, 也可以规避1里的 db 问题.
- 问题3, 单纯代码问题, 统一用一套代码重构就行了.

讲真, 碰到的 celery 的 bug 实在太多了, 存量代码用用就行了, 新做业务实在不想用. 而发推送是个很独立的业务, 最后打算拿 go 来单独跑推送.

有现成的项目: https://github.com/appleboy/gorush, 但不太符合我的需求:

- 业务和它交互的时候用的是 rest 或 grpc, 我更倾向中间走个任务队列, 减少业务端的 latency.
- 根据 apple/google 的响应, 有些业务代码还是需要的.

也考察过 [nsq](nsq.io), 问题是, 我的发送端还是 python, nsq 的 python client 又是基于 tornado 的, 而我线上是基于 gevent 的, 不是说不能用, python 的几套异步方案谁都没比谁强多少, 混在一起够你折腾的, 算了吧.

最后基于 https://github.com/adjust/rmq 这个 lib 单独做了个 service, 原理其实和 celery 用 redis 做 broker 是一样的, 基于 redis 的 list 类型做了个任务队列, 正好也能复用现成的 redis. 发送端就是一个简单的 redis lpush, 任何语言都不用复杂包装.

## 上线

基于 rmq 的开发过程没什么好讲的, 也就是对接了下 APNS/FCM 的 api, 调试下, 两三天就完事了. 过下在 k8s 上跑一个完整的 stateless service, 除了开发完业务逻辑之外还有那些是需要的, 熟悉 k8s 生态的这些应该都是常规.

### Visibility

- error report 发送到 sentry: https://github.com/getsentry/sentry-go
- prometheus 的 metrics endpoint. prometheus 我是用 prometheus-operator 建立的, 只需要创建对应的 ServiceMonitor 就能让 prometheus 自动 pull metrics.
- grafana dashboard 可视化 prometheus 中数据.
- 通过 [redis-exporter](https://github.com/oliver006/redis_exporter) 监控 redis 中 pending task 数目, alert manager 中建立对应报警.

### Scalability

- scalability: 根据 CPU 设定好 k8s 的 HPA, 开始上线可以宽松点, 后续看负载调整策略.

### Deployment

- 编写 Dockerfile(runtime 不需要 golang, 可以用 docker 的 multi-stage builds)
- 编写 helm chart, 创建对应的 k8s deployment, hpa, servicemonitor... 
- jenkins pipeline 做后续的镜像更新, 改了代码可以在 jenkins 上部署, 不需要有权限操作 kubectl.

## 效果

原先 celery 处峰值时候 pending 的 task 减少了一大半. 新上的 go service, 常规内存占用不到 10M, 峰值时候 CPU 占用不过 15%, 基本没来不及发的推送, 效果非常好.
