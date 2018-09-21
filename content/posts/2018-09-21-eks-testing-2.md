---
title: "eks 评测 part-2"
date: 2018-09-21T10:28:17+08:00
categories:
    - tech
tags:
    - k8s
    - eks
    - aws
---

上文测试了一下 EKS 和 cluster autoscaler, 本文记录对 persisten volume 的测试.

# PersistentVolume

创建 gp2 类型的 storageclass, 并用 annotations 设置为默认 sc, dynamic volume provision 会用到:

    
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
        name: gp2
        annotations:
            storageclass.kubernetes.io/is-default-class: "true"
    provisioner: kubernetes.io/aws-ebs
    reclaimPolicy: Retain
    parameters:
        type: gp2
        fsType: ext4
        encrypted: "true"

因为 eks 是基于 1.10.3 的, volume expansion 还是 alpha 状态, 没法自动开启(没法改 api server 配置), 所以 storageclass 的 allowVolumeExpansion, 设置了也没用.
这里 `encrypted` 的值必须是字符串, 否则会创建失败, 而且报错莫名其妙.

## 创建 pod 的时候指定一个已存在的 ebs volume

    
    apiVersion: v1
    kind: Pod
    metadata:
        name: test
    spec:
        volumes:
            - name: test
              awsElasticBlockStore:
                  fsType: ext4
                  volumeID: vol-03670d6294ccf29fd
        containers:
            - image: nginx
              name: nginx
              volumeMounts:
                  - name: test
                    mountPath: /mnt

`kubectl -it test -- /bin/bash`  进去看一下:


    root@test:/# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    overlay          20G  2.2G   18G  11% /
    tmpfs           3.9G     0  3.9G   0% /dev
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/xvdcz      976M  2.6M  907M   1% /mnt
    /dev/xvda1       20G  2.2G   18G  11% /etc/hosts
    shm              64M     0   64M   0% /dev/shm
    tmpfs           3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
    tmpfs           3.9G     0  3.9G   0% /sys/firmware

那块 volume 的确绑定在 `/mnt`


## 通过 PersistentVolume 和 PersistentVolumeClaim 将存储细节和 pod 解耦

这里的 PersistentVolume 由管理员创建. 


    kind: PersistentVolume
    apiVersion: v1
    metadata:
        name: jenkins
    spec:
        accessModes: 
            - ReadWriteOnce
        capacity:
            storage: 1Gi
        awsElasticBlockStore:
            fsType: ext4
            volumeID: vol-03670d6294ccf29fd
        persistentVolumeReclaimPolicy: Retain

`persistentVolumeReclaimPolicy` 指定了当 pvc 被删除后, pv应该如何处理(默认为 Retain, 设置为 Delete, pv 也会自动删除), 这里的 capacity 必填, 但实际上只是个声明,
并不会去检查底层 ebs 的实际容量.

PersistentVolumeClaim 由用户在创建 Pod 的时候创建.


    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: jenkins-claim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
          requests:
              storage: 1Gi

这里 request 了一个 1G 的 volume, 如果 request  2G, 由于之前只有一个 1G 的 pv, pvc 得不到满足,会处在 pending 的状态. 成功绑定, pvc 和  pv 都会处在 Bound 状态.

创建一个 pod 并将 pvc 挂载上去, 因为底层的 ebs 是预先手工创建的, 要确保 worker node 和 ebs 处在同一个 available zone:

    
    apiVersion: v1
    kind: Pod
    metadata:
        name: test
    spec:
        volumes:
            - name: test
              persistentVolumeClaim:
                  claimName: jenkins-claim
        containers:
            - image: nginx
              name: nginx
              volumeMounts:
                  - name: test
                    mountPath: /mnt

试了一下在 pvc 挂载到 pod 的状态下, 执行 `kubectl delete pvc jenkins-claim`, pvc 会一直处在 `Terminating` 状态, pod 不会受到影响, 再把 pod 删除, pvc 就自动删除了.

一块 ebs 只有在第一次挂到 pod 的时候才会被 attach 对应的 instance 上, 之后 delete pod, delete pvc, ebs 都不会被释放, 要将 pv 删除, ebs 才会回到 available.

当 pv  的 persistentVolumeReclaimPolicy 设置为 Delete, pvc 删除后, pv 也会自动删除(底层的 ebs 也会 自动删除).

如果一个 pv 没有被 bound 到 pvc 上, 手工删除 pv 的话, 底层的 ebs 不会被删除.


## Dynamic volume provision

pvc 设置了 storageClassName 就可以进行 dynamic provision, 自动创建一个和 request 大小一样的 pv (和对应的 ebs).

dynamic 创建的 pv, reclaim policy 取决于创建时使用的 storage class

如果设置了 default storageclass(通过 sc 的 annotations), 并且在创建 pvc 的时候没有设置 volumeName 和 selector, 会用 default sc 进行 dynamic provision. 

如果创建 pvc 时不指定  storageclass (并且没有设置 default sc), 会去找满足 request 的 pv, 找不到会处在 pending 状态.

# 总结

pv,pvc, dynamic provision, StorageClass, 还有 ebs 是否自动删除, 有点搞:

- storage class 是用来给 pvc 做 dynamic provision 的.
- pvc 的 volumeName 和 selector 用来挑选 pv, 指定了就不会进行 dynamic provision.
- pvc 没指定 volumeName, selector, 并指定 storageClass, 会进行 dynamic provision.
- pvc 没指定 volumeName, selector, storageClass, 如果设置了 default storageclass, 会进行 dynamic provision, 没有则会 pending .
- dynamic provision 创建的 pv, reclaim policy 取决于 storage class.
- 手工创建的 pv, reclaim policy 取决于 persistentVolumeReclaimPolicy.
- pv 的 reclaim policy 是 Retain 的, 手工删除 pv, ebs 也会保留.
- pv 的 reclaim policy 是 Delete 的, 在 pv 被删除后 ebs 会被自动删除.
- pvc 还存在的情况下删除 pv, pv 会一直处在 terminating 状态直到 pvc 被删除.
