---
title: "Rolling Upgrade Worker Nodes in EKS"
date: 2021-04-01T14:40:30+08:00
categories:
- tech
tags:
- aws
- eks
- k8s
- redis
---

EKS control plane 的升级是比较简单的, 直接在 aws console 上点下就可以了, 但 worker node 是自己用 asg(autoscaling group) 管理的, 升级 worker node 又不想影响业务是有讲究的.

跑在 EKS 里, 且希望不被中断 traffic 的有:

- stateless 的 api server, queue consumer
- 被 redis sentinel 监控着的 redis master/slave
- 用于 cache 的 redis cluster

写了个内部工具, 把下面的流程全部自动化了. 这样升级 eks 版本, 需要更换 worker node 时候就轻松多了. 因为这个工具对部署情况做了很多假设和限制, 开源的价值不是很大.

## Stateless application

stateless 的应用全部用 deployment 部署.

一般建议的流程是:

- 修改 asg 的 launch configuration, 指向新版本的 ami
- 把所有老的 worker node 用 kubectl cordon 标记成 unschedulable
- 关闭 cluster-autoscaler
- 修改 asg 的 desired count, 让 asg 用新 ami 启动新的 worker node
- 用 kubectl drain 把老 worker node 上的 pod evict 掉, 让它们 schedule 到新的 worker node 上. 
- 重新开启 cluster autoscaler, 等它把老的闲置 worker node 关闭.

这里有些问题:

- kubectl cordon 会导致 ingress 的 health check 挂掉, 在 aws 下, 是 nginx-ingress-controller 所在节点会被从 elb 上摘掉, 导致所有 traffic 断掉: [#65013](https://github.com/kubernetes/kubernetes/issues/65013)
- 如果某个 deployment 有三个 pod, 其中两个不巧在同一个 node 上, evict 的时候会导致2/3的算力不可用(高版本的k8s可以用[Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/) 进行限制).

我的做法:

- 问题1, 不用 cordon, 给 node 打上一个自定义的 taint, eg: eks-rolling:NoSchedule. 这样也能阻止 pod 被 schedule 到老节点.
- 问题2, 给需要保持一定比例的 pod alive的 dpeloyment设置[pod disruption budget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets), evict api 会遵循 pdb, 比如设置了 max unavailable=1, 一个节点上有两个属于同一 deployment pod 的情况, 只会同一时间 evict 一个. 注意: delete api 是不看 pdb 的.
- 因为整个过程代码化了, 我就没用 cluster-autoscaler 来关闭节点(过程中先把 cluster-autoscaler replcas 设置为0, 防止意外), 直接用 asg 的 TerminateInstanceInAutoScalingGroup api 可以指定 instance id 来关闭

## Sentinel based redis master/slave

部署情况:

- master, slave 各用一个 statefulset 部署, master replicas 为 1, slave 按需控制 replicas. master 和 slave 部署在不同的 asg(处于不同的 available zone)
- sentinel 为三节点 statefulset. 配置里设置 master pod 的 headless svc 为 monitor 对象. sentinel 启动时候会 resolve 成 ip, 问题是如果 slave pod 重启, 在 eks 下 private ip 会变化,此时 sentinel 的监控列表里会残留一个一直是 `odown` 状态的 slave, 只能重启 sentinel 来刷新列表.

migrate slave pod:

- 老的 worker node 标记上 taint
- 删除 slave pod, 让它 re-schedule 到新的 worker node 上, 检查 replication 状态直到同步完成.
- 重启 sentinel pod, 刷新 slave ip.

migrate master pod:

- 老的 worker node 标记上 taint 
- failover 到 slave pod
- 删除原来的 master pod, 让它 rescheule 到新的 worker node
- 让重建的 master node 重新变成 slave 的 slave, 等待同步完成
- failover 回 master node
- 重启 sentinel 刷新 ip(原先的 master pod ip 留在了 slave 列表里)

一个老 worker node 上的所有 redis pod 被迁移走之后, 就可以用上一节里的方法关闭它.

## Redis cluster

部署情况:

- 两个 available zone, 里面各部署一个 statefulset, 两个 statefulset 中的 redis 共同组成一个 redis cluster

分成两个 statefulset 的原因是, 每个 redis 都挂载了一块 ebs(用来存储 nodes.conf, 这个不能丢), pod 和对应的 ebs 必须处于同一个 az, 不能跨 az 挂载.

migrate:

- 老的 worker node 标记 taint
- 创建一个临时的 redis pod(从已存在 redis pod 所属的statefulset 上拿到对应的 PodSpec)
- 将临时 redis pod 加入 redis cluster, 成为替换 pod A 的 slave.
- 将临时 redis pod failover 为 master, 取代 pod A.
- 删除 pod A, 让它在新的 worker node 上重建, 并等待和临时 redis pod 的同步完成
- failover 回 pod A
- 将临时 redis pod 离开 redis cluster
- 删除临时 redis pod
