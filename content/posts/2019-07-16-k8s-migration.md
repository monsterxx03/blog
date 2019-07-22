---
title: " K8s Migration issue"
date: 2019-07-16T12:32:08+08:00
draft: true
tags:
    - k8s
    - eks
categories:
    - tech
---

开始把线上流量往 k8s 集群里面导了, 中间碰到了茫茫多的问题 ...... 记录一下(大多都不是 k8s 的问题). 

## nginx ingress controller 的问题

### zero-downtime pods upgrade 

默认配置下, nginx ingress controller 的 upstream 是 service 的 endpoints, 在 eks 里, 就是 vpc cni plugin 分配给 pod 的 vpc ip(不是 cluster ip),
和直接使用 service cluster ip 比, 好处是:

- 可以支持 sticky session
- 可以用 round robin 之外的负载均衡算法

具体实现是, 当 service 的 endpoint 列表发生变化时, nginx ingress controller 收到通知, 对它管理的 nginx 进程发起一个 http request, 更新 endpoint ip
list(nginx 内置的 lua 来修改内存中的 ip list)

这样的问题是, 从 pod 被干掉到 nginx 更新之间有个时间差, 部分请求会挂掉, 解决方法可以给 pod 设置一个 preStop hook, sleep 几秒, 等 nginx ingress controller
更新完成. 

或者给 ingress 设置 annotation 让它使用 cluster ip + port:  https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#service-upstream, 当然这样就只能靠 iptables 做 round roubin 了, sticky session 也不支持.

### ExternalTrafficPolicy Local 不工作

这个问题比较奇怪, 我两个 eks cluster, 一个设置了 Local policy 之后 没有问题, 另一个却导致了 elb 后所有节点 healthcheck 失败.

实在没什么头绪, 跟踪下 ExternalTrafficPolicy 更改时候做了些什么吧, 一路跟踪到这里(eks 版本是 1.13.7) https://github.com/kubernetes/kubernetes/blob/v1.13.7/pkg/proxy/endpoints.go#L221
    
    isLocal := addr.NodeName != nil && *addr.NodeName == ect.hostname

NodeName 必须和 hostname 相等, 看到这, 想起了早些时候测试 metrics-server 碰到的问题, 也是因为 hostname 造成的. 登到 worker node 上看下 hostname, 果然不是默认的 `ip-x-x-x-x.ec2.internal`, 变成了 `ip-x-x-x-x.yyy.net`, 想起 yyy.net 是早些时候在 vpc 的 dhcp option 里设置的 search domain, 应该是 cloud init 在 launch 的时候设置的.   

自己给自己埋了个大坑, 去掉 dhcp option, 重新 launch work node, hostname 正常了, ExternalTrafficPolicy 设置成 Local 也没问题了.

## datadog-agent 的 bug

当我开始把部分流量切到 k8s 上时, 发现以 daemonset 运行的 datadog-agent(一开始没有设置 cpu limit) cpu 占用异常高(800m+), 而且只在跑 nginx ingress conroller 的节点上高. 
进去运行一下 `agent status`, 发现 network check 花了30s, 其他节点都只要几百ms.

datadog agent 的 check 都是 python 写的, 比较好调试, 发现是在获取当前 conntrack 数目的地方耗时很久,　实现是遍历挂载上去的 `/host/proc/net/nf_conntrack` 内容, 统计 conntrack 数目, nginx ingress controller 的节点上高峰时 conntrack 数目大概有 9w 多, 怪不得慢.

看了下 github, 新版已经修复了, 改成了读取 `/host/proc/sys/net/netfilter/nf_conntrack_count` 里的计数值: https://github.com/DataDog/integrations-core/commit/0497a14bbd1c0601f230c3e9f4a81aa2eed5ec7b#diff-dd651adb689f92282ee490f6dd77a033L418, 更新下镜像, 搞定.

## fluent-bit 的 bug

fluent-bit 作为 daemonset 运行, 一边收集本地 container log, 一边作为 fluentd 的 forwarder 工作, 用来收集应用记录的 metrics events.

现象是 fluentd 那边记录下来的 metrics events 有大量重复. 开始不太确定是 fluentd 的问题还是 fluent-bit 的问题. 先在 fluentd 上抓了下包, 收到的 event 和记录下的是能对的上的, 那就是 fluent-bit 发重复了. 看下 github 果然有对应 issue, 新版里面也修复了: https://github.com/fluent/fluent-bit/commit/20c9cda33c33073ec3d5c0868c1a2ca5e80b5fcb
 , 更新搞定.

## uwsgi 的问题

原先在 vm 上的结构是, backend 是几组 uwsgi 做容器的 rpc server(http), 前端的 web 项目调用它们(不用 grpc 纯粹是因为改不动了...), rpc server 前面套了层 nginx, 通过 `uwsgi_pass` 走 unix domain socket 和 uwsgi 进程通讯.

流量切换的时候, 在 k8s 内的 service 用 nginx ingress controller 暴露一个 load balancer, 在 route53 上配置权重 dns, 让 vm 和 k8s 同时 handle rpc 请求, 所以 k8s 内 service
前面也走了 nginx. 等测试稳定后, 把 k8s 内部分 service 的 base url 换成 k8s 内域名, 跳过 elb 和 nginx, 直接走 iptables 负载均衡.

结果一切换, 很快报了大量 502. 看了下 k8s 内 service 的 response header, 没有 Content-Length, 也没有 `Transfer-Encoding: chunked`, 原因是 uwsgi 配置错误, 我用了 `http-socket`, 这种模式需要前面加 nginx, 改成 `http`, 这才是 native 的 http 支持.

正确配置:
    
    http = :8080
    http-keepalive = 75
    http-auto-chunked = true
    add-header = Connection: Keep-Alive

测试 http keepalive:

    curl -v http://localhost:8080 > /dev/null http://localhost:8080 > /dev/null

看到 `* Re-using existing connection! (#0) with host localhost` 就对了.

修改配置再次上线, 502 倒是没了, latency 却比之前套 nginx 的时候更高了, 明明网络少了 2 跳(elb + nginx).

经过大量 benchmark 测试, 应该是 uwsgi 的 http router 性能不行, 模拟原先 vm 上的结构, 在 rpc 的 pod 里放一个 nginx container 来解 http request, 再 `uwsgi_pass` 给 python, latency 立马恢复了正常.
