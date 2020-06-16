---
title: "解决 k8s 1.16 apiVersion deprecation 造成的 helm revision 冲突"
date: 2020-06-16T16:02:58+08:00
categories:
- tech
tags:
- eks
- k8s
- aws
---

最近开始把线上的 k8s 从 1.15 升级到 1.16, 1.16 里有一些 api verison 被彻底废弃, 需要迁移到新的 api version, 具体有: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.16.md#deprecations-and-removals

有两个问题:

- 集群中使用的一些第三方 controller(nginx-ingress, external-dns-controller...), 调用的 apiVersion 需要升级.
- 已存在集群中的 objects(Deployment/ReplicaSet...), 是否需要处理, eg: Deployment: extensions/v1beta1 -> apps/v1.


第一个问题好解决, 升级一下对应的 image 版本就行了, 只要还在维护的 controller, 都已经升级到支持 1.16. 自己写的工具链也排查下是否有还在使用老版本 api 的, 因为我用的是 aws eks, 开下 [control-plane-logs](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html) 里的 audit log, 可以看还有什么在调用老的 api.

第二个问题是不需要改, 已存在的 objects 无法修改 apiVersion, 集群升级到 1.16 后， 从新的 apiVersion 里能 pull 到之前的数据, apiVersion 字段自动就升级了.

如果在 1.15 上尝试先删除 objects, 用新 apiVersion 重新创建, 再 get 一次会发现 api 还是老的, 比如创建 `apps/v1` 的 Deployment, `kubectl get deployment -o yaml` 看到的
还是 `extensions/v1beta1`, 因为 kubectl 会搜索当前 apiserver 支持的所有 api group, 返回最老的那个, 用 `kubectl get deployment.v1.apps -o yaml` 可以看到新 api.

但如果用 helm 管理 k8s 上应用的话, 第二个问题会造成 helm 里保存的历史 revision 和 chart 里修改过的新 apiVersion 冲突, helm 总是会尝试执行 upgrade 操作, 然后失败.

默认情况下, helm 里每次更新的 revision 会保存成 k8s 里的一个 Secrets. 用 `kubectl get secret -l owner=helm,status=deployed  -n <namespace>` 可以列出.

        sh.helm.release.v1.jaeger-operator.v1              helm.sh/release.v1   1      180d
        sh.helm.release.v1.jenkins.v1                      helm.sh/release.v1   1      4d1h

helm upgrade 会找到该 chart 的最新 revision, 和本地要更新的 chart 做比较, 找出 diff, 对 k8s apiserver 做 update 操作(helm2 和 helm3 在这里不太一样, helm3 是 3-way strategic merge patches, 会同时看 k8s 中对应 objects 的当前状态, helm2 只会看 Secrets 里保存的状态).

所以只要将 helm Secrets 里保存的 apiVersion 修改成最新的, 就可以跳过 apiVersion 的 update.

导出 revision 内容:

        kubectl get secrets sh.helm.release.v1.jenkins.v1  -o json  | jq -r  '.data.release'  | base64 -d | base64 -d | gzip -d

手工修改后重新 apply 对应的的 Secrets　就能解决冲突.

另外还有个 kubectl 插件: https://github.com/hickeyma/helm-mapkubeapis, 可以自动化得做这事. eg: `kubectl mapkubeapis jenkins`

具体要映射那些 apiVersion, 它记录在一个 yaml 里: https://github.com/hickeyma/helm-mapkubeapis/blob/master/config/Map.yaml

像 rbac 相关 api 在 1.17 才废弃, 如果想在这次升级里一起做了, 可以手工把对应的 deprecatedInVersion 改到 1.15.

之后只要修改下对应 charts 里的 apiVersion 就好了, 不放心可以用 [helm-diff](https://github.com/databus23/helm-diff) 插件看下, 不输出对 apiVersion 的更改就对了.