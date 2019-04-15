---
title: "K8s Volume Resize on EKS"
date: 2019-04-12T13:23:54+08:00
categories:
  - tech
tags:
  - k8s
  - eks
  - aws
---

从 k8s 1.8 开始支持 [PersistentVolumeClaimResize](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/). 但 api 是 alpha 状态, 默认不开启, eks launch
的时候版本是 1.10, 因为没法改 control plane, 所以没法直接在 k8s 内做 ebs 扩容. 后来升级到了
1.11, 这个 feature 默认被打开了, 尝试了下直接在 EKS 内做 ebs 的扩容. 

注意:

- 这个 feature 只能对通过 pvc 管理的 volume 做扩容, 如果直接挂的是 pv, 只能自己按传统的 ebs 扩容流程在 eks 之外做.
- 用来创建 pvc 的 storageclass 上必须设置 `allowVolumeExpansion` 为 true

在 eks 上使用 pv/pvc,　对于需要 retain 的 volume, 我一般的流程是:

- 在 eks 之外手工创建 ebs volume.
- 在 eks 中创建 pv, 指向 ebs 的 volume id
- 在 eks 中创建 pvc, 指向 pv

示例 yaml:

    ---
    kind: PersistentVolume
    apiVersion: v1
    metadata:
      name: test
    spec:
      storageClassName: gp2
      persistentVolumeReclaimPolicy: Retain
      accessModes:
        - ReadWriteOnce
      capacity:
        storage: 5Gi
      awsElasticBlockStore:
        fsType: ext4
        volumeID: vol-xxxx   # create in aws manually
    ---
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: test-claim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      volumeName: test


假如现在要扩容到 10Gi, 流程是:

`kubectl edit pvc test-claim`,　将 requests 的 storage 改成: `10Gi`

`kubectl get pvc test-claim -o yaml`　会看到 condition type 是 `resizing`,
此时看 ebs 的 console, 能看到 ebs status 那栏是黄色的: `available - optimizing (1%)`,
这个过程会比较慢.

完成之后 pvc 的 condition type 会变成: `FileSystemResizePending`.

但此时挂载该 volume 的 pod 看到的 size 还没更新, 需要删除 pod, 重新创建, pod 就会自动更新 volume size, `kubectl exec`　
进去, `df -h` 看下, size 已经被更新了. 1.11 开始有个 alpha 的 feature `ExpandInUsePersistentVolumes` 可以不重新创建 pod
的情况下更新 volume size.

完成后 `kubectl get pv test`, 看一下, 发现对应 pv 的 capacity 也被更新成 10Gi 了.

注意:

一定要更改 pvc 的大小, 而不是 pv 的大小, 如果改了 pv, pvc 就会检测不到 size 的变化, 底层 ebs 并没被扩容,
但 k8s 里看 pv/pvc 大小已经被改了. 因为不能缩容, 如果改了 pv size, 只能把 pv/pvc 都删除, 重新创建(指向同一个 ebs volume id),
再将 pvc resize.
