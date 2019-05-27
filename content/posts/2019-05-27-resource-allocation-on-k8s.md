---
title: "2019 05 27 Resource Allocation on K8s"
date: 2019-05-27T11:51:39+08:00
draft: true
categories:
    - tech
tags:
    - k8s
    - eks
---


查看节点资源分配情况:

    kubectl describe node nod1


scheduler 根据 requests 的 resource 来调度而不是当前使用率.


如果 pod1 request 了 100M 的 cpu, pod2 request 200M 的 cpu, 总 cpu 是1000M.

如果两个 pod 都全力使用 cpu, 剩下的 700M cpu 会在两个 pod 之间以 1:2 的比例分配.

如果一个　pod2 是 idle, pod1 全力跑, pod1 仍旧可以使用剩下的全部 cpu 资源.


extended resource

QoS class: Guaranteed > Burstable > BestEffort

单 container 的 pod, container 的 QoS class 就是 pod 的 QoS class.

Requests/Limits 都没设置, QoS class 是 BestEffort.

Requests < Limits, 或只设置 Requests, QoS class 是 Burstable.

Requests 和Limits 都设置了,并且相等, QoS class 是 Guaranteed.

只设置 Limits, 没 Requests, Requests 自动设置成等于 Limits, QoS class 是 Guaranteed.

多 container 的 pod, 如果所有 container QoS class 都一样, pod 的 QoS class 也一样.

任意一个 container 的 QoS class 不一样, pod 的 QoS class 就是 Burstable.

当 node 内存耗尽时, 低 QoS class 的 pod 会被先踢走.

当 QoS class 相同的 pod 有多个时, OOM score 高的进程会被踢走.

OOM score 由两部分组成:

- 进程消耗它 requests 的 memory 的百分比
- 由 QoS class 决定的一个固定值.

所以同 QoS class 时候, 消耗了 80% requests memory 的 pod 会先比消耗 70% requests memory 的 pod 先被踢走,即使它实际使用的内存比较少.
