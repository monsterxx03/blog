---
title: "EKS Test"
date: 2018-09-11T15:02:22+08:00
draft: true
categories:
    - tech
tags:
    - k8s
    - eks
    - aws
---

记录测试　EKS 过程中碰到的一些问题.

## Setup

AWS 官方的 Guide 只提供了一个 cloudformation template 来设置 worker node, 我用 terraform 来设置, 可以跟着这个文档尝试:https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html 来设置完整的 eks cluster 和管理 worker node 的 autoscaling  group.

一些要点:

Setup 完　EKS 后需要添加一条 ConfigMap:

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

## Metrics server 的问题

Heapster 已经废弃, 需要转到　metrics server, EKS 最早发布的时候不支持　metrics server(因此 HPA 也不支持), EKS　的　PlantformVersion 需要是 'eks.2'.

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


## Helm && metrics-server && HPA

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

可以用 helm 部署 metrics-server 啦:

    helm install stable/metrics-server --namespace kube-system

成功获取 metrics 信息:

    kubectl top nodes

    NAME                          CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
    ip-10-100-39-200.ec2.internal   33m          1%        468Mi           5%  

配置 cloud-autoscaler charts, 这里我用 audodiscovery 的方式让 cloud-autoscaler 找到 aws 的 autoscaling group, 需要提前在 auto scaling group 上设置 tag: `k8s.io/cluster-autoscaler/enabled`, `kubernetes.io/cluster/<ClusterName>`, value 随意.


    autoDiscovery:
      clusterName:  <ClusterName>
    sslCertPath: /etc/kubernetes/pki/ca.crt

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
