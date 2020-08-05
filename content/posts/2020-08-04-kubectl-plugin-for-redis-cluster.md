---
title: "Kubectl Plugin for Redis Cluster"
date: 2020-08-04T22:44:40+08:00
categories:
- tech
tags:
- redis
- k8s
---

在 k8s 上部署 redis cluster 后, 感觉 redis-cli 管理 redis cluster 非常别扭, 写了个 kubectl 的插件 [kubectl-rc](https://github.com/monsterxx03/kubectl-rc) 来辅助管理 redis-cluster.

## redis-cli 难用在哪

1. 不直观 & 不统一

部分 cluster 信息是直接通过 redis protocol 获得的, 比如 `cluster nodes`, `cluster slots`, 但部分管理命令又是通过 `redis-cli --cluster` 执行的.

`cluster nodes`, `cluster slots` 这些命令输出的又是 ip 和 node id, k8s 环境下我更关心实际的 pod name.

做 failover 的时候又不是通过 `--cluster` 执行的, 必须连到 slave 上通过 `cluster failover` 来执行

2. 传参在 k8s 环境下特别麻烦

举个例子, 添加节点:

    redis-cli --cluster add-node new_host:new_port existing_host:existing_port
        --cluster-slave --cluster-master-id <arg>

需要知道操作 pod 的 ip, 如果要变成某个指定 pod 的 slave, 又要传 node id.

在 k8s 环境下实际操作的时候流程就会变成:
- `kubectl get pods -o wide` 得到 new_host, existing_host 的 ip
- `kubectl exec -it <existing_pod> bash -- redis-cli cluster nodes`, 得到 ip 和 node id 的映射关系
- 小心别看花眼把参数都记对了再去执行 add-node ...

add-node 这个命令本身也不合理, 如果传了 `--cluster-slave` 不传 `--cluster-master-id` 新节点会变成随机现存 master 节点的 slave, 大多情况下直接变成前面传的 existing_host 的 slave 不就完了. 而且 cluster 模式下 slave 是不允许串联的, 如果传入的 master id 是一个 slave, redis-cluster 会报错, 但该节点却会成功加入 cluster, 成为一个没分配 slots 的 master, 先检查下目标是不是 master, 不是直接跳过不得了..

del-node 的参数又是 `host:port node_id`, 我能理解 redis-cluster 因为节点之间因为是通过 gossip protocol 的方式传递信息的, 所以第一个输入总是随机某个节点来作为 entry point, 后面的才是操作对像. 但实际情况, 直接把 del-node 的信息发送到要删除的节点不就好了, 只要输一遍参数.

## kubectl-rc 做了什么

目标是在 k8s 环境下统一上述操作中的冗余和容易出错的地方. 使用过程中完全屏蔽 ip 和 node id, 传入的操作对像全换成 pod name. 输出的集群信息里把 pod name 和 ip, node id 关联起来.

具体实现有两种方式, 一是对某个 pod 做端口转口, 然后就能在插件内通过 redis sdk 操作所有命令.
或者 exec 进某个 pod 执行 redis-cli, 把 pod name 自动翻译成对应的 ip 和 node id.

前者的优点是灵活, 啥都能干, 但工作量比较大, 等于要重新实现 redis-cli 里所有 cluster 管理命令, 像 `--cluster rebalance` 还挺复杂的, 时间比较紧, 后者能达成我 90% 的目标, 所以我用的 exec 方式.

## 使用

直接 `go get github.com/monsterxx03/kubectl-rc`, 或者 clone 下来执行 make.

生成的 kubeclt-rc binary 要放在 PATH 中.

`kubectl rc help`

        Used as a kubectl plugin, to get redis cluster info, 
        replace redis nodes on k8s.

        Usage:
        rc [command]

        Available Commands:
        add-node    Make a pod join redis-cluster
        call        Run command on redis node
        check       Check nodes for slots configuration
        create      Create redis cluster
        del-node    Delete a node from redis cluster
        failover    Promote a slave to master
        help        Help about any command
        info        Get redis cluster info
        nodes       List nodes in redis cluster
        rebalance   Rebalance slots in redis cluster
        slots       Get cluster slots info

        Flags:
            --config string      kubeconfig used for kubectl, will try to load from $KUBECONFIG first
        -c, --container string   container name
        -h, --help               help for rc
        -n, --namespace string   namespace (default "default")
        -p, --port int           redis port (default 6379)
        -v, --verbose            print debug info

        Use "rc [command] --help" for more information about a command.

k8s 的配置信息, 会优先去读 $KUBECONFIG 设置的, 没有的话通过 `--config` 选项传入.

列出所有 redis nodes: `kubectl rc nodes rc-0`, 只需传入一个 pod 作为 entry point 就行

         Pod          IP                                   NodeID                       Host IsMaster Slots
        rc-0 10.0.45.194 84f62928424e945dcf56fc12f59ceead7e0101cd ip-10-0-40-50.ec2.internal     true  5461
        rc-2  10.0.43.45 96e929fbd646c8386c9587b46e3d9a58a3fcf74e ip-10-0-40-51.ec2.internal     true  5461
        rc-1  10.0.44.38 10dafd8b7c5c40f22351cdb013b16295ae722b0f ip-10-0-40-53.ec2.internal     true  5462  

列出 slots 分配信息: `kubectl rc slots rc-0`, 输出不再是 ip 和 node id, 而是 pod name

            slots       master      slaves
            0-5460        rc-0   rc-3 rc-4
        10923-16383       rc-2       
        5461-10922        rc-1    


将 rc-3 加入 cluster 并成为 rc-0 的 slave: `kubectl rc add-node rc-0 rc-3 --slave`, 直观多了, rc-0 如果不是 master， 就直接报错不加入 rc-3
