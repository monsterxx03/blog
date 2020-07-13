---
title: "Upgrade Redis From 3 to 6"
date: 2020-07-08T17:08:40+08:00
draft: true
category:
- tech
tags:
- redis
- k8s
---

线上有个 redis 的缓存集群, 跑的还是 3.0, 前面套 twemproxy 做 sharding. 跑了好几年了都很稳定, 但一直有些很不爽的地方, 最近有点时间,决定
升级到redis 6, 并迁移到 redis cluster 方案. 

### 动机

我们对 redis 的需求完全是当 cache, 没有持久化需求, aof 和 rdb dump 都是关闭的, 基于成本考虑, 其实连 slave 都没用(我是想要的...).

目前 redis 3.0 + twemproxy 方案的主要不爽在:

- redis 3.0 无法回收内存碎片, 目前一个 redis 实例设置的 maxmemory 是 5GB, 内存碎片率在1.3, 实际占用物理内存 6.5 GB 左右, 想减少内存碎片只能重启. redis 4.0 开始有 active-defrag 功能, 可以主动进行内存回收.
- twemproxy 中间加了一层, 导致 latency 增加.
- twemproxy 以 node 为 hash 的单位, 添加节点的时候会造成大量 cache miss, cluster 方案可以按 slot 迁移, 迁移过程中已经迁移完成的 client 的查询会被重定向到新节点. 不会 cache miss.
- 无 downtime 得更换节点非常麻烦, twemproxy 下是挂个 slave 上去, slave 关闭 readonly, 切个 dns, 或者改代码配置指向 slave, 或者用 aws 的 sencondary ip实现类似keeplived vip 漂移的功能(没用 sentinel 是因为日常不挂 slave). redis cluster 只要加一个 slave 节点, 手工触发一次 failover 就好了.


测试将一个 redis instance 迁移到 redis6, 内存占用: 5GB -> 4.35 GB

### 在 k8s 上部署 redis cluster

kubelet --allowed-unsafe-sysctls 'kernel.msg*,net.core.somaxconn' ...

minikube start --extra-config="kubelet.allowed-unsafe-sysctls=net.core.somaxconn" --kubernetes-version="1.16.10" --driver="virtualbox"


### Operation

添加节点

    redis-cli --cluster add-node new_host:new_port  existing_host:existing_port
    redis-cli --cluster rebalance host:port  --cluster-use-empty-masters --cluster-pipeline 100

替换节点

    # add slave
    redis-cli --cluster add-node slave_host:slave_port existing_host:existing_port --cluster-slave --cluster-master-id node-xxx
    # failover slave to master
    redis-cli -h slave_host -p slave_port cluster failover
    # remove slave
    redis-cli --cluster del-node host:port old-master-xxxx


移除节点

    redis-cli --cluster rebalance host:port --cluster-weight  node_id=0 --cluster-pipeline 100
    redis-cli --cluster del-node host:port node-xxxx
