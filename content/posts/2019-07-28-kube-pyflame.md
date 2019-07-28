---
title: "Pyflame 的 kubectl plugin"
date: 2019-07-28T12:47:23+08:00
categories:
- tech
tags:
- python
- k8s
---

[pyflame](https://github.com/uber/pyflame) 可以比较方便得生成 python 进程的调用函数栈火焰图, 来 debug
一些性能瓶颈, 做了个 kubectl 的小插件, 来方便得对 k8s pod 中的 python 进程进行 debug: https://github.com/monsterxx03/kube-pyflame
直接把 svg 文件下载到本地.

要对 pod 中的 python 进程进行 profiling, 大致思路有两种:

- 直接在 container 内使用 pyflame, 但这样要把 pyflame 做到所有的 base 镜像里去, 而且目标 container要在 SecurityContext 加上 `SYS_PTRACE`
- 在 host 上用 pyflame debug 对应进程, pyflame 自身是能识别跑在 container 里的进程, 自动执行 `setns` 的.

我希望保持线上环境干净, 最后的做法是, 把 pyflame 单独做一个镜像, 先用 kubectl 找到目标 pod 所在节点, 然后用 nodeSelector 在对应节点上起一个
pyflame 的 pod, 因为要能看到其他 namespace 的进程, 需要设置 `hostPID: true`, pyflame 要能执行 `setns`, 这个 debug pod 要设置 `privileged: true`.
执行完成后把 svg 下载下来, 并删除 debug pod.

使用很简单:

    kubectl pyflame -n namepace -p target_pod -m gunicorn

`-m` 用来配置要查找的进程名, 因为实际可能有多个进程符合查找规则(prefork 出来的), 会打印匹配到的进程列表(附带 cpu, 内存使用信息).
选择对应的 pid 后再执行 profiling.

有个更新更活跃的项目 [py-spy](https://github.com/benfred/py-spy), 可以做一样的事情, 但是试用下后有些问题:

- profiling container 中进程报错, 看代码它也是会自动 setns 的, 但实际测试下还是有问题. 
- 对使用了 gevent 的程序输出的火焰图不准确.
