---
title: "EKS 评测"
date: 2018-09-11T15:02:22+08:00
categories:
    - tech
tags:
    - k8s
    - eks
    - aws
    - server-infra
---

EKS 正式 launch 还没有正经用过, 最近总算试了一把, 记录一点.

## Setup

AWS 官方的 Guide 只提供了一个 cloudformation template 来设置 worker node, 我喜欢用 terraform, 可以跟着这个文档尝试:https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html 来设置完整的 eks cluster 和管理 worker node 的 autoscaling  group.

设置完 EKS 后需要添加一条 ConfigMap:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: arn:aws:iam::<account-id>:role/eksNodeRole
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes

这样 worker node 节点才能加入集群.

## 网络

之前一直没有在 AWS 上尝试构建 k8s 的一个原因, 就是不喜欢 overlay 网络, 给系统带来了额外的复杂度和管理开销, VPC flowlog 看不到 pod 之间流量, 封包后 tcpdump 不好 debug 应用层流量.

随着 EKS 的发布，AWS 开源了基于 VPC 的 CNI 插件(https://github.com/aws/amazon-vpc-cni-k8s)，之前也大致写过: https://blog.monsterxx03.com/2018/04/09/aws-%E7%9A%84-k8s-cni-plugin/

这个项目分成两部分, cni 插件 `aws-cni` (给 kubelet 调用) 和 `aws-k8s-agent`, 会以 daemonset 的形式运行(运行时 pod 名字是 aws-node),实际上是一个L-IPAM (local ip address management), 负责将 eni 绑定到 ec2 instance, 将 ip 地址绑定到 eni.

这样 pod 之间就能用真正的 vpc ip 进行通信了.

不同类型的 ec2 instance 可以分配的 eni 数目，每个 eni 能绑定的 ip 地址数目都不同, 具体可见: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html

一台 ec2 可分配的 pod 总数目是 `# of eni * (# of ip-per-eni - 1) + 2`, 这里 `-1`, 是因为每个 eni 上有个　ip 是　primiary ip, 不可分配给 pod, +2 是每个 node 启动时候固定会启动的两个 pod `aws-node` 和 `kube-proxy`, 这两个 pod 不需要和其他 pod 通信, 没有额外占用 ip, 实际用了 eni 的 primiary ip.

比如 m4.large，可分配 eni 是2, 每个 eni 能绑定 ip 数是 10, 所以一台 m4.large 最多可以承载 `2 * (10 - 1) + 2 = 20` 个 pod, m4.xlarge 就是 `4 * (15 -1) + 2 = 58`  

启动 worker node 之后，可以看下 pod 状态, `kubectl get pods -n kube-system -o wide`:

    
    NAME                                                    READY     STATUS    RESTARTS   AGE       IP            NODE
    aws-node-455j8                                          1/1       Running   1          56m       10.100.33.212   ip-10-100-33-212.ec2.internal
    cni-metrics-helper-56485f8b9d-tpmcm                     1/1       Running   0          58m       10.100.32.89    ip-10-100-33-212.ec2.internal
    foolhardy-fly-aws-cluster-autoscaler-599b6f9b68-tlfln   1/1       Running   0          58m       10.100.32.41    ip-10-100-33-212.ec2.internal
    honest-kitten-metrics-server-754cddf4f9-9sgcp           1/1       Running   0          58m       10.100.38.70    ip-10-100-33-212.ec2.internal
    kube-dns-64b69465b4-h42nz                               3/3       Running   0          58m       10.100.33.194   ip-10-100-33-212.ec2.internal
    kube-proxy-nj6lc                                        1/1       Running   0          56m       10.100.33.212   ip-10-100-33-212.ec2.internal
    tiller-deploy-78ddf4757c-ljs7p                          1/1       Running   0          58m       10.100.33.20    ip-10-100-33-212.ec2.internal

`aws-node` 和 `kube-proxy` ip 都是 `10.100.33.212`, 这个就是当前 eni 的 primary ip.

为了让一个 node 上启动的 pod 数目不超过可分配　ip 数目，需要告诉 kubelet 限制 `--max-pods`, 目前的做法是在 build 镜像的时候把　instance type 和 最大 pod 数映射文件放进去, 在 ec2 启动的时候,
用 user-data 注入脚本, 根据 instance type 来自动限制, 详情可以看: https://github.com/awslabs/amazon-eks-ami/blob/master/files/bootstrap.sh

debug 方法:

- `aws-node` 这个 pod 默认会启动一个 http server, 监听61678 端口, 提供两个接口 `/v1/enis` 和 `/v1/pods`,　用来显示 ip 分配和 pod 之间的关系.
- log 会输出在 worker node 的 `/var/log/aws-routed-eni` 
- 可以启动一个收集　cni metrics 的 pod, `kubectl apply -f https://raw.githubusercontent.com/liwenwu-amazon/amazon-vpc-cni-k8s-1/metrics1/misc/cni_metrics_helper.yaml`, 之后 `kubectl logs cni-metrics-xxx -n kube-system` 可以看到 pod　的一些　metrics 信息.


## Metrics server 的问题

为了测试　auto scaling, 需要在 EKS 中部署 metrics server.  EKS 最早发布的时候不支持　metrics server(因此 HPA 也不支持), EKS　的　PlantformVersion 需要是 'eks.2'.

之前不支持的原因是 metrics server 基于　`api aggregation` , core api server 只能用 client 证书认证, EKS 为了和 IAM 交互用的是 webhook. 后来　EKS　的人修了下: https://github.com/kubernetes/kubernetes/pull/66394/files#diff-59f09be65a7ae1feb263f6bce1d856b1  

详情看: https://aws.amazon.com/cn/blogs/opensource/horizontal-pod-autoscaling-eks/

用https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8%2B 提供的配置尝试部署 metrics-server.

现在　metrics server 能正常启动了，但 `kubectl top` 还是取不到数据, metrics server 的　log 里能看到:

    E0911 06:18:58.805099       1 manager.go:102] unable to fully collect metrics: unable to fully scrape metrics from source kubelet_summary:ip-10-100-39-198.ec2.internal:
     unable to fetch metrics from Kubelet ip-10-100-39-198.ec2.internal (ip-10-100-39-198.test.com): Get https://ip-10-100-39-198.test.com:10250/stats/summary/: dial tcp: lookup ip-10-100-39-198.test.com on 172.20.0.10:53: no such host

看上去是因为我在　vpc 中设置了 dhcp option, 默认 domain 为 test.com, ec2 worker node 启动后　hostname 自动变成了 ip-10-100-39-198.test.com, metrics server 启动后在 pull node 信息的时候，走了 node 的 hostname .这个域名自然是不存在的, 所以连不上. 

解决办法可以把　dhcp option 设回　ec2.internal, 或者让 metrics-server 用　ip 来连接:

      command:
       - /metrics-server
       - --kubelet-insecure-tls
       - --kubelet-preferred-address-types=InternalIP

还有一个问题是 eks worker node　的　/etc/resolv.conf 里的内容变成了:


    search test.com us-west-2.compute.internal
    nameserver 10.100.0.2


test.com 是　dhcp option 分发的域名, us-west-2.compute.internal 哪来的? 而且我的测试节点是启动在 us-east-1 的.

## 配置 helm

给  helm 创建　service account, 和 cluster-admin　的　role 绑定:

    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
        name: tiller
        namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
        name: tiller-clusterrolebinding
    subjects:
    - kind: ServiceAccount
      name: tiller
      namespace: kube-system
    roleRef:
        kind: ClusterRole
        name: cluster-admin
        apiGroup: ""

用 tiller 帐号初始化 helm 的 tiller server `helm init --service-account tiller`

`helm ls` 下试试权限有没有问题.

## AutoScaling on EKS

梳理下要在 EKS 上做　auto scaling, 我们需要些什么. 这里的　auto scaling 分两部分, pod 的 auto scaling, worker node 的 auto scaling.

pod 的 scale 由　`HorizonalPodAutoscaler`(aka: HPA) 负责, HPA 通过　metrics-server 采集到的数据, 计算要将 pod 的某个 metrics 维持在一个 threshold 额外需要多少个 pod.
然后会去更新 ReplicationSet 的　replica 字段.

当前 node 资源不足以 allocate 新的 pod 时, 新的 pod 会处在 pending 状态. `cluster autoscaler` 检测到 pending 状态的 pod, 就会修改 aws autoscaling group 的 desired 数目来 launch 
新的节点.  

用 helm 来部署　`metrics-server` 和 `cluster-autoscaler`.


    helm install stable/metrics-server --namespace kube-system

成功获取 metrics 信息:

    kubectl top nodes

    NAME                          CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
    ip-10-100-39-200.ec2.internal   33m          1%        468Mi           5%  

配置 cloud-autoscaler charts, 这里我用 audodiscovery 的方式让 cloud-autoscaler 找到 aws 的 autoscaling group, 需要提前在 auto scaling group 上设置 tag: `k8s.io/cluster-autoscaler/enabled`, `kubernetes.io/cluster/<ClusterName>`, value 随意.


    autoDiscovery:
      clusterName:  <ClusterName>
    sslCertPath: /etc/kubernetes/pki/ca.crt
    rbac:
      create: true

注意这里的 `sslCertPath` 默认是 `/etc/ssl/certs/ca-certificates.crt`, eks 放的位置不太一样.

cloud-autoscaler 还需要特定的 iam 权限, 可以把它绑定到 worker node 的 instance role 上:

     {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "autoscaling:DescribeAutoScalingGroups",
                    "autoscaling:DescribeAutoScalingInstances",
                    "autoscaling:DescribeLaunchConfigurations",
                    "autoscaling:DescribeTags",
                    "autoscaling:SetDesiredCapacity",
                    "autoscaling:TerminateInstanceInAutoScalingGroup"
                ],
                "Resource": "*"
            }
        ]
    }

`helm install stable/cloud-autoscaler --namespace kube-system`

简单测试一下　`cluster-autoscaler`, 现在的测试节点里只有一台 m4.large, 总共可以放 20 个 pod, 已经预先跑了一些系统 pod(aws-node, kube-proxy, tiller, metrics-server, cluster-autoscaler...), 尝试部署一个 20  replica 的 nginx, 同时用 `kubectl get pods -w -o wide` 观察 pod 的分配情况, `kubectl get nodes -w` 观察新 node:

    kubectl run nginx --image=nginx -r 20

可以看到在原先的 node 上创建了一部分 nginx pod, 然后新的 node 被 cluster-autoscaler 启动,　并加入集群, 接着剩余 pod 在新 node 上创建.

HPA 就不测了, 关于 k8s 的资源管理, 包括 scale in 时候的策略, pod 的 reallocate 是个挺复杂的话题, 之后再讲吧.
