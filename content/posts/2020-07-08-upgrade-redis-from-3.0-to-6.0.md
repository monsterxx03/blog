---
title: "从 twemproxy 迁移到 redis cluster"
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

### twemproxy 的工作模式

twemproxy 的原理很简单, 后面运行 N 个 redis 实例, 应用连到 twemproxy, twemproxy 解析应用发过来的 redis protocol, 根据 key 做 hash, 打散到后面 N 个 redis 实例上.

具体打散的方式可以是简单的 hash%N, 也可以用一致性 hash 算法. hash%N 的问题是, 增减节点的时候所有 cache 必然 miss.

一致性 hash, 在实现的时候会先弄一个 size 很大的 hash ring(eg: 2^32),这里每个节点被叫做一个虚拟节点, 把虚拟节点的数目叫做 X, 然后将 N 个 redis 实例均匀分布到这个环上, 每个 redis 把它叫做实节点吧, 分配 key
的时候 hash%X, 得到虚拟节点, 然后顺时针找下一个最近的实节点, 就找到了相应的 redis. 因为虚拟节点的数目是不变的, 增减 redis 实例的数目是改变了实节点的分布, 顺时针找下个实节点的时候还是有一定几率落在以前的
redis 实例上的, 这在一定程度上减少了 cache 的 miss. 可以看作 hash%N 的优化版本,但不解决本质问题.

### redis cluster 的工作模式

twemproxy 的原理比较好理解, 缺点有三:

- 增减节点时候必然会发生 cache miss, 增大数据库压力. 手工做数据迁移也不是不行, 就是非常麻烦...
- 无法利用 redis sentinel 实现 redis 实例的 HA. 开启 `auto_eject_hosts` 可以自动把挂掉的 redis 实例从 hash ring 里踢出去, 但这个会导致 key 的重分布, 如果 redis 的检测失败是由于间歇性的网络问题造成的, 对于重 cache 的应用, 可能还不如等手工把问题解决了再恢复业务.....
- twemproxy 还要再想办法做 HA

redis cluster 的工作模式就高端一点, 把整个 key 空间划分到 16384 个 slot 上. 用 crc16(key)%16384, 来决定 key 属于哪个 slot. 每台 redis 实例各分配一段 slot.

开启了 cluster 模式的 redis 实例之间通过 gossip protocol 传递集群信息(谁是master, 谁是 slave, slots的分布情况), 而 master slave 的切换在内部实现了类似 raft 的选举算法(取代单机模式下的 HA 方案 sentinel).

每个节点都知道每个slot当前分配在哪个节点上, 当收到 client 的请求后, 根据 key 算出所处 slot, 如果 slot 分配在自己这, 直接返回结果, 否则返回 `Moved <slot> host port` 的结果, 告诉 client 去对应的节点取值, 
因为 crc16(key)%16384 这个算法是固定的, 所以 client 可以预计算出 key 属于哪个节点, 大多数时候并不会请求两次, 只有在发生 failover 的时候会收到 Moved, 此时再拉一次节点信息就好了, 当增减节点的时候 slot 会发生迁移, slot 的迁移过程中会给 client 返回一个 ASK 响应, 表示迁移还没完成, 你去另一个节点再去问问这个 key 的归属, 可以把 Moved 理解成 http 301, slot 已经永久迁移了, client 你可以记着, Ask 是 http 302, 临时重定向, slot的迁移还没完成, 你别急着更新自己的节点映射信息. 这种实现被叫做 smart client.

redis 3.0 刚引入cluster 模式的时候, 反应并不很好, 原因就是这种 smart client 的模式过于复杂, 对 client lib 的实现要求比较高, 在多语言环境下, 各语言 client lib 的成熟度不一样, 比较难整. 但大家又比较想要 redis cluster 的 sharding, failover 功能, 所以也有人会在中间加一层 proxy 来实现 smart client 的功能.  

我这次没在中间加 proxy, 是直接访问的 redis cluster, 大部分情况下都没问题, 因为少了 twemproxy, latency 还能得到改善. 

主要问题在 mget, mset 这样的 multi key 操作上, 只有操作的 key 属于同一个 slot 才可以执行, 否则会得到 CROSSSLOT 的 error, 即使这些 slot 属于同一个 node 也没办法, 因为当发生 slot 的迁移的时候, 一条命令里包含不属于同一 slot 的 key  时,没法给 client 返回正确的 Ask. redis-py-cluster 里对 mget 的实现就是个简单的 for loop get. 另外 pipeline, transaction 的功能也一样. 可以用 hashtag 的功能强制一些 key 归属到同一个 slot, 不过这个场景限制太多了, 我的业务里很难用得上. 只能说做个 trade offf 吧, 看自己的业务是否能忍受这些情况.


### 动机

我们对 redis 的需求完全是当 cache, 没有持久化需求, aof 和 rdb dump 都是关闭的, 基于成本考虑, 其实连 slave 都没用(我是想要的...).

目前 redis 3.0 + twemproxy 方案的主要不爽在:

- redis 3.0 无法回收内存碎片, 目前一个 redis 实例设置的 maxmemory 是 5GB, 内存碎片率在1.3, 实际占用物理内存 6.5 GB 左右, 想减少内存碎片只能重启. redis 4.0 开始有 active-defrag 功能, 可以主动进行内存回收.
- twemproxy 中间加了一层, 导致 latency 增加.
- twemproxy 以 node 为 hash 的单位, 添加节点的时候会造成大量 cache miss, cluster 方案可以按 slot 迁移, 迁移过程中已经迁移完成的 client 的查询会被重定向到新节点. 不会 cache miss.
- 无 downtime 得更换节点非常麻烦, twemproxy 下是挂个 slave 上去, slave 关闭 readonly, 切个 dns, 或者改代码配置指向 slave, 或者用 aws 的 sencondary ip实现类似keeplived vip 漂移的功能(没用 sentinel 是因为日常不挂 slave). redis cluster 只要加一个 slave 节点, 手工触发一次 failover 就好了.


测试将一个 redis instance 迁移到 redis6, 内存占用: 5GB -> 4.35 GB

### 在 k8s 上部署 redis cluster

redis 启动时候需要修改 net.core.somaxconn 这个内核参数, 在容器环境下 可以通过 securityContext 进行, 但要在 kubelet 的配置里把这个值加进白名单:

    kubelet --allowed-unsafe-sysctls 'kernel.msg*,net.core.somaxconn' ...
    minikube start --extra-config="kubelet.allowed-unsafe-sysctls=net.core.somaxconn" --kubernetes-version="1.16.10" --driver="virtualbox"

使用 bitnami 的 helm chart 部署 redis cluster: https://github.com/bitnami/charts/tree/master/bitnami/redis-cluster/

会给 redis cluster 创建一个 headless service, 实际使用的时候, 应用使用这个 headless service 的 dns, 可以解析到 redis pod 在 aws eks 上的 vpc ip, 不存在 NAT


    >>> dig redis-cluster-cache-headless.default.svc.cluster.local

    ...
    ;; ANSWER SECTION:
    redis-cluster-cache-headless.default.svc.cluster.local.	5 IN A 10.0.43.4
    redis-cluster-cache-headless.default.svc.cluster.local.	5 IN A 10.0.42.12
    redis-cluster-cache-headless.default.svc.cluster.local.	5 IN A 10.0.41.14

aws eks 上的 pod 重启后 vpc ip 是会变的, 在 redis cluster 里怎么处理这种情况? bitnami 的做法:

- 在 chart 里把所有节点的 headless domain 作为 env 注入 container: https://github.com/bitnami/charts/blob/master/bitnami/redis-cluster/templates/redis-statefulset.yaml#L103
- redis 的镜像里有脚本在启动的时候读取 `REDIS_NODES` 变量, 查询每个 pod 的 headless domain, 得到当前 vpc ip, 生成 `nodes.conf`  https://github.com/bitnami/bitnami-docker-redis-cluster/blob/6.0.6-debian-10-r0/6.0/debian-10/rootfs/opt/bitnami/scripts/librediscluster.sh#L229
    

这样, redis pod 重启后可以自动加入 cluster. 注意必须挂一块 pv 上去存 `nodes.conf`.

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

node failure(slots unassigned)
    
    redis-cli cluster forget old-node-id-xxx   # do it on every exising master node
    redis-cli cluster meet new_host:new_ip
    redis-cli --cluster add-node new_host:new_port existing_host:existing_port
    redis-cli --cluster fix host:port

