---
title: "从k8s deprecating docker 说起"
date: 2020-12-04T10:49:40+08:00
categories:
- tech
tags:
- docker
- k8s
---

k8s 1.20 的 release note 里说 deprecated docker: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation

对 docker 和 k8s 关系比较了解的人一看就知道是废弃 dockershim, 正常操作, 具体有什么影响, 建议阅读:

- https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/
- https://kubernetes.io/blog/2020/12/02/dockershim-faq/

但容器圈里那堆名词的确很让人困惑, docker, dockershim, containerd, containerd-shim, runc, cri, oci, csi, cni, cri-o, 或许曾经还听说过 runv, rkt, clear containers, 后来又来了 kata containers, firecracker, gvisor. 论造名词, 造项目, 都快赶上前端圈的脚趾头了. 

这篇比较水, 就解释下它们大致的关系, 前提是你大概知道 docker, namespace, cgroup, container, k8s pod 之间的关系, 不做额外解释.

## 从 docker 说起

在我现在这台机器上的 ubuntu 18.04, 安装 docker, 添加官方源之后, 安装的 deb package(version 19.03) 是 docker-ce

`apt show docker-ce`  看依赖:

     Depends: docker-ce-cli, containerd.io (>= 1.2.2-3), iptables, libseccomp2 (>= 2.3.0), libc6 (>= 2.8), libdevmapper1.02.1 (>= 2:1.02.97), libsystemd0


主要就是 docker-ce-cli　和 containerd.io 这两个包, docker-ce-cli 只是那个 cli, 忽略.


先看下 docker-ce 里有哪些文件: `dpkg -L docker-ce`, 主要看 /lib/systemd 下面的 systemd service unit　文件和/usr/bin 下的 binary.


    /lib/systemd/system/docker.service
    /usr/bin/docker-init
    /usr/bin/docker-proxy
    /usr/bin/dockerd

systemd 的启动文件 `/lib/systemd/system/docker.service`, dockerd 进程就由它管理.

/usr/bin/docker-init 是docker 自带的init 进程, 当 `docker run` 带上 `--init` 选项的时候会调用, 取代 run 后面的命令成为容器内 1 号进程(ps aux | grep docker-init 确认), 实际用的代码是 [tini](https://github.com/krallin/tini), 

/usr/bin/docker-proxy 用来做容器端口转发, `docker run` 带上 `-p` 参数时候会调用.

dockerd 是 docker 的 daemon 进程(常说的 docker engine), docker cli 命令发起的请求由它接受(通过 /var/run/docker.sock), 走 http rest api.

在目前的 k8s 版本 <=1.20, 用 docker 做 runtime 的时候, docker-init, docker-proxy 都没有使用, 需要 init 进程, 自己在 image 里放 tini 做 entrypoint, 端口转发由 kubelet 完成.

## 再说 containerd

containerd 是 docker 在 1.11 版本拆出来的container runtime 组件, 之前的 dockerd 进程功能比较复杂, 除了 container runtime 之外, 还要负责 volume, overlay 网络等功能. 当初 docker 做这个拆分, 估计也是为了docker swarm, 进军容器编排市场, 毕竟 docker 自带的 volume, network 在分布式环境下都很鸡肋, 不如把runtime 拆出来单独用. dockerd 和 containerd 之间通过 /run/containerd/containerd.sock 走 grpc 通讯.

再看下 `containerd.io` 这个包里有什么, `dpkg -L containerd.io`

    
    /lib/systemd/system/containerd.service
    /usr/bin/containerd
    /usr/bin/containerd-shim
    /usr/bin/containerd-shim-runc-v1
    /usr/bin/containerd-shim-runc-v2
    /usr/bin/ctr
    /usr/bin/runc

ctr 是 containerd 的管理 cli, 类似 docker 和 dockerd 的关系.

runc 是管理 container 的命令行工具(创建, 删除), 但它只是对一系列syscall 的 wrapper, containerd 才是真正的 daemon 管理进程(通过containerd-shim调用 runc). 当说 container runtime 的时候, 其实有点困惑, 有时侯指 containerd, 有时候指 runc．准确说法是 runc 是 OCI runtime spec 的一种实现, containerd 是 CRI 的一种实现.

containerd-shim 是 containerd 和 runc 之间的中间层, docker run 启动一个 container 后, 三者关系可以通过 ps auxf 查看:

    
    root     26595  0.0  0.1 1577032 29680 ?       Ssl  Dec03   1:16 /usr/bin/containerd
    root     19850  0.0  0.0 108596  5328 ?        Sl   13:41   0:00  \_ containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/e1349492f39def8f34a81aebd1a329b1c45a678d81cc4d91a719b3f066e3e2d3 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
    root     19876  1.8  0.0 244352  9936 pts/0    Ssl+ 13:41   0:00      \_ /usr/sbin/syslog-ng -F


为什么需要一个 shim 层?　升级或重启 containerd 的时候不需要重启容器. 并且 shim 层会管理容器的stdout,stderr, `docker exec -it`, `kubectl exec -it` 等命令需要操作容器的stdin, 也由shim 层完成.

关于 containerd 和 CRI 的关系要注意一些时间点:

- containerd 在 2016年初被拆出来
- CRI 标准在2016 年末出来的
- containerd 在2017年3月进入CNCF之后才添加了CRI支持.

## kubelet & dockershim & CRI & OCI

kubelet 是 k8s 在 host 节点上的 daemon 进程. 在早期, kubelet 直接和 dockerd 进行交互. 2014年的时候, coreos 提出了rkt, 取代 docker, 主打安全, k8s native, 现在已经停止开发了, 我也没用过, 不多说. 2016 年的时候 k8s 支持 rkt 作为容器的 runtime. 因为要同时兼容多个 runtime, k8s 那边发现维护工作量太多了, 所以提出了 CRI (contianer runtime interface). 主要定义了一组 grpc 接口, 来封装容器操作(create,start,stop,remove...)

dockershim 也是在2016年的时候加进 kubelet 的: https://github.com/kubernetes/kubernetes/tree/v1.20.0-rc.0/pkg/kubelet/dockershim. 作为 kubelet 和 docker 之间的兼容层. kubelet 发出符合CRI 的grpc请求, dockershim 把它转化为 dockerd 的 http rest 请求. 不过 dockershim 也是跑在 kubelet 进程里的(自己发自己接).

为什么kubelet 不直接调用 containerd?, 看上一节的时间点, 那会containerd 并不支持 CRI 标准. 感觉2016年的时候各个公司之间应该进行了一番政治斗争...

所在在当前版本的k8s(<=1.20) 里, 使用 docker 作为 runtime 的时候，实际启动一个容器的过程是:

    kubelet -> dockershim(kubelet) -> dockerd -> containerd -> containerd-shim -> runc

层层套娃, dockershim 给 k8s 带来了维护负担, 需要跟着 docker 变. 

去掉 dockershim 后， 还是能用 containerd 作为 runtime manager, 调用过程变成了:

    kubelet -> containerd -> containerd-shim -> runc


containerd, CRI-O 都是 CRI 标准的实现, 它们通过一个OCI 标准的实现(runc)来创建容器.

OCI(Open Container Initiative) 标准包含两部分:

- runtime-spec, 如何创建，删除容器, 主要是封装了一些 syscall
- image-spec, 规定了容器镜像的结构, 目前兼容 docker image: https://github.com/opencontainers/image-spec

runc 是 OCI 的一个实现, 利用 linux kernel 中已经存在的 namespace, cgroup 等机制来进行隔离, 但创建的容器之间是share kernel 的, 对安全非常敏感的场景还是有顾虑.

kata container(OpenStack), gvisor(google), firecracker(aws) 等方案使用轻量级虚拟机来进行隔离, 应用之间完全使用独立内核.　他们和 runc 处在同一层, 一般也提供符合 OCI 标准的 wrapper, 比如 gvisor 的 runsc,
firecracker 的 firecracker-containerd 来和上层的 containerd 交互. 

runv(hyperhq) 和 clear containers(intel) 合并成了 kata container.

## Misc

CSI (container storage interface) 是存储标准, 用于创建 k8s 下的 PV, 在 aws 上对接 ebs, gce 上对接 persistent disk等等.

CNI (container network interface) 是网络标准, 这块也很复杂, 有 overlay 网络的 flannel, cloud vendor native 的 aws cni plugin 等等等.

cloud native 这块技术变化很快, 不提这些底层的 runtime, 上层的 service mesh 也是半年变个样. 

如果换掉 docker, 对用户层的影响主要是:

- host 的 provision 配置要重新做, 原先容器的日志可能是 dockerd 来进行 rotate 的.
- 原先在 k8s 用 docker 做 image build, 一般是把 /var/run/docker.sock mount 到容器内, 因为 docker build 依赖 dockerd 进程, 现在不行了, 可以换buildah 或 kaniko.
