---
title: "Jenkins on K8S"
date: 2019-04-29T15:56:12+08:00
categories:
  - tech
tags:
  - k8s
  - jenkins
---

最近在把 jenkins 迁移到 k8s, 具体怎么 setup 的不赘述了(helm chart, jenkins home 目录挂pvc, jenkins kubernetes-plugin).

jenkins 跑 k8s 好处是可以方便得做分布式 build, 每次 trigger 一个 job 的时候自动起一个 pod 作为 jenkins slave agent, 结束了自动删掉.
在 aws 上结合 cluster-autoscaler 可以极大得扩展 ci 的并行能力, 降低成本.

记录一点过程中的坑.


装上 [kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin) 后,要想让 jenkins 的 job 在 pod 中跑, 必须用 pipeline 的方式编写 job 定义. script 和 declarative
两种方式都支持启动 pod. 如果用 shell 的方式编写, 不会跑在 pod 里，会直接在　master 的　workspace 里 build.

declarative 方式的例子:


    pipeline {
        agent {
            kubernetes  {
                label 'test-deploy'
                yamlFile 'test-deploy.yaml'
            }
        }
        stages {
            stage('stage test') {
                steps('tests') {
                    container('test') {
                        sh 'ls'
                    }
                }
            }
        }
    }

`ls` 命令就会在 test container 里执行, 如果不用 `container()` step 的话，默认会在 pod 的 default container 里执行.

每个 pod 里除了自己定义的 container 之外, 还会自动起一个 jnlp 的 container, 用来把 pod 变成 jenkins slave, 和 master 通信.

## 单元测试

原先 ci 上最麻烦的部分是跑单元测试, 环境配置很复杂, 包含:

- MySQL
- Redis
- dynamodb-local
- ElasticSearch
- goaws (用来模拟 aws 的 SNS + SQS)
- nodejs server (一个无状态的 server, 主要跑一些计算逻辑)
- pylint (在 ut 跑完后进行静态检查, 生成报告)
- sonar-scanner (做额外的静态检查, 并把 junit 和 pylint 生成的报告上传到 sonarqube server)

我会做一个 ut 专用的镜像, 里面包含所有 ut 需要的依赖库+pylint+sonar-scanner. 剩下的 service 每个一个 container. 全部跑在一个 pod 里.

ut 里面有些很 tricky 的地方, 实际项目是分成好几个 service, 互相之间通过 rpc 交互, 在 ut 里也没有把 rpc 调用的地方 mock 掉，而是在底层直接
动态 import 了对应 service 的 server 端代码直接本地执行. 所以难点在跑任意一个 repo 的 ut 实际需要能调用到它所有关联 service 的代码. 也想过直接做
集成测试, 但这样本地开发时候 ut 就没法跑了.

因为 repo 太大了, 即使用 shallow clone + sparse checkout, 每次跑 ut 拉一边所有代码还是太傻了. 最后我的做法是, 单独弄一个 pvc, 预先把所有 repo clone 进去.
ut 的 container 挂载这个 pvc, 并在 pod 启动的时候用 init container 把所有 repo 更新到需要的 revision.

这样的问题是 ut 没法并行跑, 每个 ut 都需要 lock 住那个 pvc, 但目前对 ut 的时间要求并不高, 慢慢排着, 只要不影响到非 ut 的 job build 就行.

因为每个 repo 跑 ut 的 pipeline 流程都差不多,只是参数区别, 所以可以把 ut 的 pipeline 写成 groovy shared lib, 从每个 repo 的 Jenkinsfile 里传需要自定义的参数就好了,
可以参考: https://jenkins.io/blog/2017/10/02/pipeline-templates-with-shared-libraries/

这里碰到了一个 bug, 我的代码目录是挂在 slave 的 jenkins home 外面的, 此时在 pipeline 里用 `dir()` step 切换到代码目录的时候会报 AccessDenied 错误: [JENKINS-54512](https://issues.jenkins-ci.org/browse/JENKINS-54512), dir step 只能在 workspace 内部切换位置. 同样的 `junit` step 也没法把 workspace 外的 report xml 发布成 ut report.


## 多 pod build

我还有一个需求是可以在一个 jenkins job 内并行启动多个 pod 进行 build.

场景是有个 repo 里面有 5 个前端项目, 当然可以分别建 5 个 jenkins job, 但能从一个 job 里选择要 build 哪几个项目会方便很多. 

用 script 的方式编写 pipeline 可以做到:

    parallel (
        portal1: {
            podTemplate(label: 'portal1', containers: [...]) {
                node('portal1') {
                    ...
                }
            }
        },
        portal2: {
            podTemplate(label: 'portal2', containers: [...]) {
                node('portal2') {
                    ...
                }
            }
        },
        ...
    )

可以很方便得动态生成要 build 的 pod.

因为现在 build 时候的 pod 都是动态 schedule 的了, yarn/npm 的本地 cache 就没意义, 我用 [npm-proxy-cache](https://github.com/runk/npm-proxy-cache)
做了一个代理 cache 部署到 k8s 里(挂块 pvc 上去做 storage dir). npm/yarn install 的时候指定 `--proxy --https-proxy`　来使用缓存(还要设置 strict-ssl false).
即使 npm registry 挂了也不影响 build.

## 和 cluster-autoscaler 集成

cluster-autoscaler 我也是用 helm chart 部署的, 设置成 autodiscovery 模式, 在 autoscaling group 上加上 tag:

    k8s.io/cluster-autoscaler/enabled: 'true'
    k8s.io/cluster-autoscaler/sandbox-eks:  ''

这样 cluster-autoscaler 就会在这些 asg 中启动 node.

aws 的 autoscaling group 原生是通过 cloudwatch 收集数据, 根据设定的 scaling policy(cpu 利用率等) 来决定 scanle up/down 的.

用了 cluster-autoscaler 就不是通过 cloudwatch 来做了.

每次在 yaml 中声明 container 的时候指定需要的 cpu/memory request, 这样 k8s 就会寻找当前系统中资源满足需要的 node, 找不到的话
pod 会处在 pending 状态, cluster-autoscaler 会看处在 pending 状态的 pod 的 requst resource, 根据设定的 expander policy(random,
most-pods, least-waste)  来决定怎么起新 node 放置 pod.

为了让 jenkins build 的 pod 不跑到我放置其他 service 的节点上, 我是通过 nodeSeletor 来做调度隔离的(当然也可以用 node/pod affinity 来做).

用 node selector 来做有个问题是, 当 auto scaling group scale down 到0 的时候, 如果要起 pod, cluster-autoscaler 会卡住无法启动 node,
需要把 node selector 里指定的 label 设置到 autoscaling group 上去才行, 比如 node selector 里写了 `jenkins-slave-node: true`,
那 autoscaling group 需要加上 tag: `k8s.io/cluster-autoscaler/node-template/label/jenkins-slave-node: true`.

要控制 cluster-autoscaler scale down 的频率可以修改 `scale-down-delay-after-add`, `scale-down-unneeded-time` 参数.
