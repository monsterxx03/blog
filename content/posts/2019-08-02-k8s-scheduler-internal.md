---
title: "kube-scheduler internal"
date: 2019-08-02T16:30:10+08:00
categories:
- tech
tags:
- k8s
---

追了一下 kube-scheduler 的源码, 记录一点, 基于 tag `v1.16.0-alpha.2`.

一句话概括 kube-scheduler 的职责是: 找到 pending 的 pod, 挑选一个合适的 node, 将 pod bind 上去.


## Get pending pod

在 scheduler 的初始化过程中给 `pod/node/pv/pvc/service/storageClassInformer` 添加回调函数, 功能大致都是在这些资源发生变化时更新本地的 cache 和 ScheduleQueue [scheduler.go:New](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/scheduler.go#L207).

ScheduleQueue 是关键, 内部实现是一个 PriorityQueue, 它有三个 sub queue:

- activeQ 用来存放等待 schedule 的 pods, kube-schedule 实际工作时候从这个 queue 中取 pod, 实现上是一个 heap, 如果 pod 定义了 priority, 则按照 priority 由高到低排序, 否则按 pod 到达的时间排序: [activeQComp](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/internal/queue/scheduling_queue.go#L154)
- podBackoffQ 存放正在经历 backoff 的 pod, 也是 heap, 按 pod 上次 backoff 的时间排序: [podsCompareBackoffCompleted](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/internal/queue/scheduling_queue.go#L651)
- unschedulableQ 存放 schedule 失败的 pod, 只需要能根据 pod 的 identity (`podName_namespace`) 找到 pod 就行, 不需要排序, 内部是个 map.

由 podInformer 的 eventHandler 将新的 pod 加到 activeQ 中.

ScheduleQueue 会尝试每秒把 podBackoffQ 中的 pod 刷到 activeQ 中, 每 30s 把上次 schedule 失败超过 60s 的 pod 刷到 activeQ 中 [run](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/internal/queue/scheduling_queue.go#L199) 

kube-scheduler 每次会尝试从 activeQ 中取一个 pod 进行 schedule, 如果没有就 pending 在那. [scheduler.go:scheduleOne](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/scheduler.go#L462) . 给 pod 找 host 的过程是同步的, 一次只 schedule 一个 pod, 将 pod bind 到 host 上的过程是异步的.

## Choose host for pod

默认使用内置的 generic scheduler, check 逻辑跳过不看, 主要是: predicates 和 prioritizing 两个步骤: [generic_scheduler.go:Schedule](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/core/generic_scheduler.go#L184)

predicates 步骤通过一系列 predicate 函数对所有的 node 进行并行评估, 如果评估过程中找到的合格 node 数超过预先配置的一个百分比就结束评估跳过剩下的节点(默认50%), 默认开启的 predicate 函数定义在 [algorithmprovider/defaults/defaulgs.go](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/algorithmprovider/defaults/defaults.go#L41) 中, 会 check node 的 volume 数, memory, disk等信息(比如不同的 volume backend 对是否允许多个 pod mount 一个 volume 的实现不同), 也会检查 node label 是否符合 nodeSelector, affnity 等条件.

predicates 结束后得到的符合条件的 node 数可能多于一个, prioritizing 过程就是对选出的节点进行优先级评估, 从中找出得分最高的节点.

prioritizing 过程也类似 predicates, 用一系列 priority 函数对候选节点进行并行评估, 每个 priority 函数会算出一个 0 - 10 之间的score, 每个 priority 函数可能还是不同的权重, sum(score * weight) 就得到了每个节点的总分. prioritizing 过程结束后, 找到得分最高的 node, 如果最高分有同分的, 就用 round-robin 的方式挑一个.

默认开启的 priority 函数定义在 [algorithmprovider/defaults/defaulgs.go](https://github.com/kubernetes/kubernetes/blob/v1.16.0-alpha.2/pkg/scheduler/algorithmprovider/defaults/defaults.go#L119) 中, 包括 node/pod affinity, 节点上是否存在 pod 需要的 image, cpu 和内存利用的均衡性等.

kube-scheduler 还支持自定义外部 expander, 可以介入 scheduler 的 predicate 和 prioritizing 过程, scheduler 向 expander 发起一个 http POST request, expander 返回自己的决策结果.
