---
title: "EkS 评测 part-3"
date: 2018-09-26T10:16:42+08:00
categories:
    - tech
tags:
    - k8s
    - eks
    - aws
---

这篇记录对 ingress 的测试.

ingress 用来将外部流量导入　k8s 内的　service. 将 service 的类型设置为 LoadBalancer / NodePort 也可以将单个 service 暴露到公网, 但用 ingress 可以只使用一个公网入口,根据　host name 或　url path 来将请求分发到不同的 service.

一般　k8s 内的资源都会由一个 controller 来负责它的状态管理, 都由 kube-controller-manager 负责，　但 ingress controller 不是它的一部分，需要是视情况自己选择合适的 ingress controller.

在 eks 上我主要需要 [ingress-nginx](https://github.com/kubernetes/ingress-nginx) 和 [aws-alb-ingress-controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller). 注意, nginx inc 还维护一个 [kubernetes-ingress](https://github.com/nginxinc/kubernetes-ingress), 和官方那个不是一个东西， 没测试过.

这里主要只测试了 ingress-nginx, 看了下内部实现, 数据的转发真扭曲....

## ingress-nginx

用 helm 安装很简单: `helm install nginx-ingress --namespace ingress`

看看安装后多了些什么:

    service:

    ingress       service/mean-quetzal-nginx-ingress-controller        LoadBalancer   172.20.220.213   internal-adcb...   80:30312/TCP,443:32273/TCP   2m
    ingress       service/mean-quetzal-nginx-ingress-default-backend   ClusterIP      172.20.157.137   <none>             80/TCP                       2m

    deployment:

    ingress       deployment.apps/mean-quetzal-nginx-ingress-controller        1         1         1            1           2m
    ingress       deployment.apps/mean-quetzal-nginx-ingress-default-backend   1         1         1            1           2m
    
    ReplicaSet:

    ingress       replicaset.apps/mean-quetzal-nginx-ingress-controller-7fb65bbd87        1         1         1         2m
    ingress       replicaset.apps/mean-quetzal-nginx-ingress-default-backend-75696ffd5d   1         1         1         2m

    Pods:

    NAMESPACE     NAME                                                              READY     STATUS    RESTARTS   AGE
    ingress       pod/mean-quetzal-nginx-ingress-controller-7fb65bbd87-jc9kv        1/1       Running   0          2m
    ingress       pod/mean-quetzal-nginx-ingress-default-backend-75696ffd5d-x7ld5   1/1       Running   0          2m

pods 和　replicaset 都是由 deployment 创建的，实际上就是创建了 `nginx-ingress-controller` 和　`nginx-ingress-default-backend` 的 deployment 和 service.

`nginx-ingress-default-backend` 提供了 404 页面和 health check url, 不需要暴露到公网, 所以 service type 是 ClusterIP

`nginx-ingress-default-backend` service type 是 LoadBalancer, 在 aws 上默认会创建一个 classic ELB.

这个 ELB 的 liseners 配置如下:


| Load Balancer Protocol | Load Balancer Port | Instance Protocol |  Instance Port  | SSL Certificate |
|------------------------|--------------------|-------------------|-----------------|-----------------|
| HTTPS                  | 443                | HTTP              | 32273           | xxxxx (ACM)         |
| HTTP                       | 80                    | HTTP                  | 30312                 | N/A                 |

在 helm charts 的默认值基础上做了些修改:

- service.annotations 里配置了 ssl cert, 用的是 aws 的 ACM.
- service.annotations 指定 backend end 协议是 http, 让 ELB 来做 ssl termination.
- service.targetPorts 里将 https 的 的targetPort 设置成 http (这样会把 https 的流量解密后发送给 nginx pod 的 80 端口)

跟踪下大致的流量走向:

public internet -> ELB port 80

ELB port 80 -> ec2 instance (30312)

在 instance 上　lsof -i:30312, 监听的进程是 kube-proxy.

但实际接受流量的并不是 kube-proxy, 流量被　iptables 劫持了.

`iptables -t nat -L | grep 30312`:

    KUBE-MARK-MASQ  tcp  --  anywhere             anywhere             /* ingress/mean-quetzal-nginx-ingress-controller:http */ tcp dpt:30312
    KUBE-SVC-YLASP4PF67FEI63M  tcp  --  anywhere             anywhere             /* ingress/mean-quetzal-nginx-ingress-controller:http */ tcp dpt:30312

`iptablse -t nat -S KUBE-MARK-MASQ`:

    -N KUBE-MARK-MASQ
    -A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000

只是给报文打上标记.

`iptables -t nat -S KUBE-SVC-YLASP4PF67FEI63M`:

    -N KUBE-SVC-YLASP4PF67FEI63M
    -A KUBE-SVC-YLASP4PF67FEI63M -m comment --comment "ingress/mean-quetzal-nginx-ingress-controller:http" -j KUBE-SEP-TMFHHIS4O35AFOOH

转发到了 `KUBE-SEP-TMFHHIS4O35AFOOH`

继续往下看 `iptables -t nat -S KUBE-SEP-TMFHHIS4O35AFOOH`:

    -N KUBE-SEP-TMFHHIS4O35AFOOH
    -A KUBE-SEP-TMFHHIS4O35AFOOH -s 10.0.47.59/32 -m comment --comment "ingress/mean-quetzal-nginx-ingress-controller:http" -j KUBE-MARK-MASQ
    -A KUBE-SEP-TMFHHIS4O35AFOOH -p tcp -m comment --comment "ingress/mean-quetzal-nginx-ingress-controller:http" -m tcp -j DNAT --to-destination 10.0.47.59:80

被转发到了 10.0.47.59 的 80 端口. 这个 ip 是 cni 分配的 vpc secondary ip, 找到该 ip 所在节点, 在宿主机上: `curl http://localhost:61678/v1/pods`, 能看到这个　ip 被分配给了 `mean-quetzal-nginx-ingress-controller-7fb65bbd87-jc9kva`

这个 container　里跑了两个进程, 一个 `nginx-ingress-controller`, 负责监听 ingress　的创建事件，并生成对应的 nginx 配置文件, reload nginx. 另一个进程就是　nginx,　监听 container　的 80 和 443　端口.

试着部署一个 kube-dashboard, 并为它创建 ingress, helm charts 的 values.yaml 里设置:

    ingress:
        ## If true, Kubernetes Dashboard Ingress will be created.
        ##
        enabled: true

        ## Kubernetes Dashboard Ingress annotations
        ##
        annotations:
            kubernetes.io/ingress.class: nginx
            nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
            nginx.ingress.kubernetes.io/ssl-redirec: 'true'
        hosts:
            - k8s.example.com
        tls: # kube dashboard 默认用自签名的证书开启了tls,　需要让 nginx　访问 kube dashboard 的时候使用对应的 key. 
            - secretName: kubernetes-dashboard-tls
              hosts:
                - k8s.example.com
 
deploy 后,　对应的vhost　信息应该会被同步给 nginx, 进 nginx-ingress-controller 的  container 看下 nginx.conf

`kubectl exec -n ingress -it mean-quetzal-nginx-ingress-controller-7fb65bbd87-jc9kv -- cat nginx.conf | less`

能看到多了一节:

    ## start server k8s.example.com
    server {
        server_name k8s.example.com ;
        listen 80;
        ...
        proxy_pass https://upstream_balancer;
    }
    ## end server k8s.example.com

这里的 `upstream_balancer` 是由 luna 脚本来选择 upstream:

    	upstream upstream_balancer {
            server 0.0.0.1; # placeholder
            balancer_by_lua_block {
                balancer.balance()
            }
            keepalive 32;
        }

代码在 `/etc/nginx/luna/balancer.lua` 里, 大致逻辑是根据 `$proxy_upsteam_name` 来挑选一个 backend service, 默认算法 `round_robin`.

可用 backend 可以在 container 内部通过 `curl -vv http://127.0.0.1:18080/configuration/backends` 查看.

默认开启了 dynamic configure, backend 列表保存在 nginx 内存中. 由 `nginx-ingress-controller` 定时访问 apiserver, 得到该 ingresss controller 对应的 service 列表, 并通过 post nginx 的 `localhost:18080/configuration/backends` 来写入.

由于 backend service 都是 ClusterIP, nginx　转发的流量到了宿主机要再走一遍　iptables 转发到对应的 node.

ingress-nginx 部署的时候可以选择以　daemonset 方式，也可以以 deployment 的形式部署, 如果是 deployment　方式,　因为对应的　pod　不一定在收到　request　的　worker node 上， 可能会多一跳, 这个要注意.

最后把　k8s.example.com 的域名 CNAME　到　ELB 的 dns, 就能通过 这个域名访问到 kube-dashboard 了.

## internal loadbalancer

默认 ingress-nginx 默认创建的 ELB 是一个 public ELB, 如果我想创建一个 internal ELB, 之后再通过 VPN 去访问 service 该怎么弄?

只要在 ingress-nginx 的 `service.annotations` 里加上 ` service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0` 就行. 但这里有个问题, 最早创建 eks cluster 的时候我同时勾选了 public subet 和 private subnet, 这样会导致创建 ELB 的时候随机挑选两个 subnet, 要强制 internal lb 在 private subnet 中创建的话,要给 private subnet 加上 tag `kubernetes.io/role/internal-elb=true`, 来告知 k8s 哪些 subnet 是 private 的.

ELB 默认的 security group 开放允许所有地址访问 80/443, 可以通过 `loadBalancerSourceRanges` 来限制来源 ip.