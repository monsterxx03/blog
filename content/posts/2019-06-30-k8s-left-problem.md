---
title: "K8S: 剩下的问题"
date: 2019-06-30T16:11:06+08:00
categories:
    - tech
tags:
    - k8s
    - eks
    - server-infra
---

准备工作都差不多了, 没意外下周就该开始把线上的服务往 k8s 上迁移了. 记录几个问题，暂时不 block 我的迁移进程,
但需要持续关注.


## DNS timeout and conntrack

看到有个关于 DNS 的issue: [#56903](https://github.com/kubernetes/kubernetes/issues/56903)

现象是 k8s cluster 内部 dns 查询间歇性会 5s 超时, 大致原因是 coredns 作为中心 dns 的时候,
要通过 iptables 把　coredns 的 cluster ip, 转化到它真实的可路由 ip, 中间需要 SNAT, DNAT, 并在
conntrack 内记录映射关系. 

这可能会带来两个问题:

### conntrack table 被 udp 的 dns 查询填满

udp 是无连接的, tcp 关闭链接就会清理 conntrack 内记录, udp 不会，只能等超时, 默认 30s(`net.netfilter.nf_conntrack_udp_timeout`)
短时间内大量 udp 查询可能填满 conntrack, 导致丢包.

怀疑是这个原因的话, 首先看系统允许的最大 conntrack 是多少(默认值由内存大小决定,不同发行版和内核版本上好像参数名不太一样, 我在 amazon linux 2,kernel 4.14 下测试):

    sysctl net.netfilter.nf_conntrack_max

查看当前已使用 conntrack 数目:

    sysctl net.netfilter.nf_conntrack_count

### SNAT, DNAT 的资源竞争导致丢包

在多线程 dns 查询下, SNAT,DNAT 可能会有个资源竞争的 bug, 导致部分 udp 包被 drop, 体现在 client 端就是5s 超时后重试, 详细解释和可能的 workaround 可以看这两篇:

- https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts
- https://blog.quentin-machu.fr/2018/06/24/5-15s-dns-lookups-on-kubernetes/

检查 conntrack -S:

    cpu=0           found=24202529 invalid=1 ignore=1419323 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=8754
    cpu=1           found=23588937 invalid=1 ignore=1418117 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=8585


看 `insert_failed` 那项, 如果不为0, 就是这个原因造成的.

### 测试

conntrack 填满的问题对我影响不大, 在 server 上抓了下包, 查询量不大, 4GB 内存的测试节点上, `nf_conntrack_max` 值默认有 13w 多, 完全够用.

SNAT, DNAT 资源竞争的问题比较搞, 写了个简单的程序来 benchmark dns 查询: https://gist.github.com/monsterxx03/8217a664d301a1b3c07c983eefcb1a3b

用法:

    ./dnsbench -c 100 -n 10000 -d google.com -s 5


并发数100, 进行 10000 次 dns 查询, 如果 response 超过 5s 就打印出来.

先在 host 上进行了测试, 没有一次超过 5s, 在 容器内跑, 会出现大量超过 5s 的请求. 但是在 host 上 conntrack -S 没看到任何 `insert_failed`, conntrack table
也没满. 

怀疑是 coredns 的问题，把 pod 的 dnsPolicy 改为 Default, 这样 /etc/resolv.conf 里就会直接用 vpc 的 dns. 还是出现了大量 5s.　

这就比较郁闷了, 现象和已知 issue 不符. 但我不会有很大量的 dns 查询, 这个问题就先放一边了.

如果是 conntrack 引起的问题, 在 1.13 里有个 addon: [NodeLocalDNS](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20190424-NodeLocalDNS-beta-proposal.md) 来做 workaround.

它大致做的是, 用 coredns 做 daemonset 来做本地的 dns cache, 在 iptables 里对去往 local dns 的 udp 查询加上 NOTRACK(可以 bypass conntrack), local dns 用 tcp 向上游 coredns 查询结果返回客户端.

### tips

如果 pod 的 dnsPolicy 是 ClusterFirst, /etc/resolv.conf 里是这样的:

    nameserver 172.20.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local ec2.internal
    options ndots:5

会有多个 search domain, 如果此时查询一个外部域名, eg: google.com, 会依次查询:

- google.com.default.svc.cluster
- google.com.svc.cluster.local
- google.com.cluster.local
- google.com.ec2.internal
- google.com

要第5次才能得到结果, workaround 是使用 `google.com.` 这样以点号结尾的 fqdn, 会跳过 search domain.

如果是 cluster 内域名, 跨 namespace 使用完整域名: `<service>.<namespace>.svc.cluster.local`

## Ingress 的问题

目前用的比较多的是 nginx ingress controller. 它实际就是创建了一个 type 为 Loadbalancer 的 service,
在 AWS 上会 provision 一个 ELB(4层lb), 在所有 worker 节点上创建一个 NodePort, 挂载到 ELB 后面.

流量进来的时候，会通过 ELB 路由到任意一个 node 的 NodePort, 问题是 nginx ingress controller 创建的 nginx
pod 不一定在该节点上, 需要下一跳找到 nginx 所在节点, 然后 nginx 再根据 ingress 规则转发到后端的 pod.

实际路径就可能是这样的:

    ELB -> random node -> nginx -> service pod

和传统方案比多了一跳, 流量可能要到随机节点上转一圈, 还不能控制哪些节点可挂载到 ELB 后面, 如果集群中任
一节点负载过高, 对 latency 会造成影响.

Workaround 的方式是可以把 nginx-ingress-controller 的创建的 service 的 [externalTrafficPolicy](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip) 改为 `Local`(默认为 Cluster).
这样 ELB 会修改 health check 的目标(Cluster 模式下是直接检查 nginx pod 的 NodePort), nginx pod 所在节点的 kube-proxy 会另
起一个端口作为 ELB health check 的目标, 其余节点的 kube-prox 不会有这个端口, health check 就会失败，不会把节点挂上去. 

Local 模式也有缺点, 多个 pod 在多个 node 上可能分布不均匀, 但 ELB 只能对 node 做 round robin 的分发, 有可能导致 pod 负载不均衡.

比较理想的方式是直接把 pod ip 挂到 Cloud 的 LB 后面. [aws-alb-ingress-controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller) 配合 aws 的 vpc cni plugin 可以做到.

vpc cni plugin 会给每个 pod 分配一个 vpc 中的 ip, 这样对 aws 的 service 来说, pod 就直接可达.
默认情况下, alb ingress controller 也是给 service 在 全部 node 上创建一个 NodePort, 全部挂到
ALB 后面去, 但可以在 ingress 上设置 [annotation](https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/ingress/annotation/#target-type): `alb.ingress.kubernetes.io/target-type: ip` (默认为 instance).
ALB 就能直接把 pod ip 加到 target group 里去.

但目前 aws-alb-ingress-controller 有个问题是, 你每创建一个 ingress, 它都会傻傻得给你 provision 一个 ALB, 太浪费了, 有个 ingress group 功能解决这个问题, 但还没 merge: [#914](https://github.com/kubernetes-sigs/aws-alb-ingress-controller/issues/914)．


## Service topology routing

k8s 内的 service 通过 iptables 做了简单的 loadbalancing， 举个栗子:

kube-dns 设置两个 replica, 相关的 iptables 规则如下：

    -A KUBE-SERVICES -d 172.30.0.10/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
    -A KUBE-SVC-TCOU7JCQXEZGVUNU -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-C6KPJNW6I7F56RNE
    -A KUBE-SVC-TCOU7JCQXEZGVUNU -j KUBE-SEP-GSC4P52JXK5C7AQB
    -A KUBE-SEP-GSC4P52JXK5C7AQB -p udp -m udp -j DNAT --to-destination 10.0.36.78:53
    -A KUBE-SEP-C6KPJNW6I7F56RNE -p udp -m udp -j DNAT --to-destination 10.0.34.112:53

KUBE-SVC 是 service 的 chain, KUBE-SEP 是 service 对应 endpoint 的 chain.

在 service chain 里用 iptables 的 statistic 模块实现了个简单的 round robin 算法, 相关代码在这里: https://github.com/kubernetes/kubernetes/blob/v1.15.0/pkg/proxy/iptables/proxier.go#L1193 (syncProxyRules 这段代码真是茫茫长...)

这么做太简陋了, 实际业务中一个 service 的 pods 往往分布在不同的 node, available zone, 甚至跨 region 也可能.

在我的业务里, 一个前端的 web service 后面会调用 N 个 rpc service, 但主要流量都是去往其中某一个 service, 所以我希望通过设置 podAffinity, 让 web service pod 和那个 rpc service pod
尽量调度到一个 node 上, 并优先访问本地的 pod, 如果本地没有或挂了再访问同一 az 中其他节点的 pod, 还不行就再去找其他的 az 的 pod.

目前 k8s 中并没有对应的实现, 但有个 KEP 所提的内容和我想要的是一样的: https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/0033-service-topology.md

设计是在 service spec 中增加一个 `topologyKeys`, 用法例子:

    kind: Service
    metadata:
      name: service-local
    spec:
      topologyKeys: ["host", "zone"]

优先找同 host 的 pod, 找不到就找同一 zone 的, 还没有就让 connection 失败(hard topology).

如果 topologyKeys 设置为 ["host", ""], 就优先找同 host pod, 找不到就找其他节点的， 不让 connection 失败(soft topology).
