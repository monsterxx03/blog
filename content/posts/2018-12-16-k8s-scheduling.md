---
title: "Kubernetes 中的 pod 调度"
date: 2018-12-16T13:09:20+08:00
categories:
  - tech
tags:
  - k8s
---

定义 pod 的时候通过添加 node selector 可以让 pod 调度到有特定 label 的 node 上去, 这是最简单的调度方式.
其他还有更复杂的调度方式: node-taints/tolerations, node-affinity, pod-affinity, 来达到让某些类型的 pod 调度到一起,
让某些类型的 pod 不跑一起的效果.

## Taints and Tolerations

如果 node 有 taints, 那只有能 tolerate 这些 taints 的 pod 才能调度到上面.

taint 的基本格式是: `<key><operator><value>:<effect>`

`kubectl describe node xxx` 可以看到节点的 taints, 比如 master 节点上会有:

    Taints:       node-role.kubernetes.io/master:NoSchedule 

这里 key 是 `node-role.kubernetes.io/master`, 没有等号和 value, operator 就是 `Exists` , effect 是 `NoSchedule`.

master 节点上的这条 taint 就定义了只有能 tolerate 它的 pod 能调度到上面, 一般都是些系统 pod.

比如看下 kube-proxy: `kubectl describe pod kube-proxy-efiv -n kube-system`

    Tolerations:     :NoExecute
                     :NoSchedule


`:NoSchedule` 表示 kube-proxy pod 可以 tolerate 所有 taints, 自然可以被调度到 master 节点上了.

taints 的 effect 有三种:

- NoSchedule, 只允许能 tolerate 这种 taint 的 pod 被调度到该节点上.
- PreferNoSchedule, **尽量**避免将 pod 调度到该节点上, 但如果没有节点可以选了, 还是会调度上去的.
- NoExecute, 前两种只影响 pod 的调度, NoExecute 会影响正跑在节点上的 pod, 除非 pod 能 tolerate NoExecute, 否则会被从上面踢走.

再看下 kube-dns 的 tolerations: `kubectl get pod kube-dns-aedfs -o yaml  -n kube-system`

    tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
        tolerationSeconds: 300
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
        tolerationSeconds: 300

`no-ready` 和 `unreachable` 这两种状态还有 `tolerationSeconds` 字段, 表示当该节点陷入这两种状态后, 该 pod 会在该节点上停留 5 分钟, 如果 5 分钟后节点还没有恢复, kube-dns 就会被从上面踢走.  这个用来在 node 启动期间或偶尔的 network partition 发生时, 避免 kube-dns 被踢走.

`not-ready` 和 `unreachable` 这两条 tolerations 是自动给所有 pod 加上的.

## Node affinity

node affinity 就是高级版的 node selector, 也是基于 node label 调度的, 以后会取代 node selector. 和 node selector 比, node affinity 支持更复杂的匹配语法, 也允许为每条规则添加优先级. 如果 node selector 和 node affinity 同时被指定, 那必须同时满足 pod 才会被调度到节点.

`<kubernetes in action>` 里的一个例子:

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: pref
    spec:
      template:
        ...
        spec:
          affinity:
            nodeAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:    
              - weight: 80                                        
                preference:                                       
                  matchExpressions:                               
                  - key: availability-zone                        
                    operator: In                                  
                    values:                                       
                    - zone1                                       
              - weight: 20                                        
                preference:                                       
                  matchExpressions:                               
                  - key: share-type                               
                    operator: In                                  
                    values:                                       
                    - dedicated                                   

nodeAffinity 目前支持两种规则: `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution`
后半部分 `IgnoredDuringExecution` 目前是固定的(以后会支持 requiredDuringExecution), 表示规则不会影响 running 的 pod, 只会在 schedule pod 的时候起作用.

和上面的 taints 一样 `required` 表明这是必须满足的规则, `preferred` 则是优先规则, 但没有节点满足的情况下就会忽略该规则.

operator 支持: In, NotIn, Exists, DoesNotExist, Gt, Lt

## Pod affinity

假设一个系统有一个 web 的 frontend server 和 一个 rpc backend server, 这两个 pod 当然可以放在不同的节点上, 但为了减少 latency, 我们希望它们尽量能被 schedule 到同一个节点上(同 nodeAffinity, 这条规则可以是 required 也可以是 preferred).

Frontend pod 的定义:


    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: frontend
    spec:
      replicas: 5
      template:
        ...
        spec:
          affinity:
            podAffinity:                                        
              requiredDuringSchedulingIgnoredDuringExecution:   
              - topologyKey: kubernetes.io/hostname             
                labelSelector:                                  
                  matchLabels:                                  
                    app: backend                                
          ...

规则类型和 nodeAffinity 是一样的, 这里是强制要求(required), `topologyKey` 表明要让 pod 调度到 `kubernetes.io/hostname` 一样的节点上(就是同一个节点), 当然也可以选择一样的 region, available zone...

由该 podAffinity 规则可知, 当调度 frontend pod 的时候, 必须和带 label: app=backend 的 pod 调度到同一节点上. 

如果测试删除 backend pod, k8s 会对 backend pod 重新调度, 虽然 podAffinity 是定义在 frontend 上的, 还是会把 backend pod 调度到 frontend 上.

默认情况下 `labelSelector` 只会匹配同一个 namespace 的 pod, 要匹配其他的, 可以在同一级添加 `namespace` 字段.

pod antiAffinity 的定义是一样的, 只要用 `podAntiAffinity` 字段就行了, 此时 `topologyKey` 就表示 antiAffinity 的 pod 不能在同一个 node/region/zone.