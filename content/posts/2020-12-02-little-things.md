---
title: "二三事"
date: 2020-12-02T16:00:13+08:00
draft: true
categories:
- tech
- life
---

天气渐凉, 无奈得拿出了秋裤. 偷懒好久没写博客了, 回顾下这阵子做了什么.


## 工作


年底打算把之前用的 dedicated ec2 instance 全部换掉, 几年前为了 HIPAA 合规做的, 但 AWS 的 BBA 里后来不要求 dedicated instance 了, 换成普通的, 可以省掉每月1400多刀的固定 dedicated fee, 同类型的 ec2 instance 可以再省10%左右. 为了这个, 打算把部分遗留在 vm 上的东西迁移到 k8s 里, 减少之后更换 instance 的工作量.


### cronjob

尝试用 [argo](https://github.com/argoproj/argo) 来调度 cronjob.

还是有不少坑的:

- cron workflow 到点后有一定几率会 trigger 重复的workflow, 目前还没有修复[#4558](https://github.com/argoproj/argo/issues/4558), 临时在所有 cronjob 的代码里加了个保护, 在 redis 里上个锁, 没抢到锁的 cronjob 直接抛异常失败. 目前看每天还是有 7,8 次重复的 workflow 会触发.
- WorkflowTemplate 可以用来复用 pod 的定义, 但外部往里面传参的方式很搓, 不是所有字段都能直接覆盖, 比如activeDeadlineSeconds, resources 等, 必须用 podSpecPatch 的方式, 文档里又不说清楚, 只能自己去翻issue.
- exit-handler 用来在 workflow 状态改变的时候触发一个 callback, 但没法获取触发它的那个 workflow 的详细信息, 比如一个 workflow 由很多个 step 构成, 失败时我希望能从 exit-handler 里拿到是哪个step 挂了, 现在做不到. 也没法往 exit-handler 里传参(只能取一些global paramaters).
- 如果一个 workflow 由多个 step 构成, 当其中一个 step 挂了, 可以设置 continueOn 控制是否继续后面的step, 但如果我选择了继续, 整个 workflow 的最终状态又是 success 的, 导致在 exit-handler 里没法区分, 最好有个 partial success 的状态.


### 日志

应用里的日志, 之前是打到部署在 vm 上的一台 rsyslog 上. 在 k8s 环境下我用 fluent-bit 做 daemonset 来收集 pod 日志. fluent-bit 在最近的版本里支持了转发到 loki, 我就部署了 loki 来收集pod 日志, 可以方便在 grafana 里进行查看. 问题也很多:

- loki 支持单机版部署, 和分布式部署(consul/etcd 做状态存储, 或者用 gossip 协议进行连接), 官方的 helm chart 只有单机版部署的. 自己从头写分离部署挺花时间的. 我最后只搞了个单机版的, 原因后详.
- fluent-bit 转发 loki 的实现用下来问题很多, 业务日志量比较多, fluent-bit 转发速度追不上日志生产速度. 我猜测是因为 fluent-bit 的 tail 插件监控 `/var/log/containers/*.log` 的时候, 所有文件的监控是跑在一个 coroutine 里的, 所以我观测即使分配了更多 cpu, 利用率也上不去. 再加上 loki 要求一个 stream 上发送的数据 timestamp 必须是按顺序的, 所以 loki 插件关闭了 multiplex, 效率进一步降低. 因为 fluent-bit 还需要接受一些fluentd 协议的东西，没法去掉, 就没测试官方的 promtail, 不想再加更多 daemonset 了.


最后我部署了一个 syslog-ng 来接受应用日志, 所有业务 pod 加上 annotation: `fluentbit.io/exclude: "true"`, 让 fluent-bit 忽略它的日志, fluent-bit 只转发其他的 pod 日志到 loki(主要是各种controller). rsyslog 的配置格式恶心了我好多年, syslog-ng 更清晰. 为了把日志 daily 得归档到 s3, 用 go 写了个 sidecar 程序和 syslog-ng 部署在一个 pod 里.

也想过用 fluent-bit 来当 中心的 syslog server, 但它局限性还挺大的:

- 如果日志打到本地, 无法模板化配置本地日志路径 [#2786](https://github.com/fluent/fluent-bit/issues/2786), 要用 lua 搞一堆 trick, 压缩上传 s3 还是要自己弄
- 上传到 s3 倒是支持模板化路径, 但目前又不支持上传前压缩, 有个 pr 进行中: https://github.com/fluent/fluent-bit/pull/2807

还曾经测试过一种方案是用 fluent-bit 的 syslog input 来接应用日志, 转发到 loki, 结果发现 loki 里的日志最后多了一个 \000(NULL), 导致 loki 的一些 parser 语法失败, 追了半天原因原来是 python 的 syslog handler 实现为了兼容一些老的 syslog server, 加上去的: https://bugs.python.org/issue12168 . 可以用 `SysLogHandler.append_nul = False` 禁掉. syslog-ng 抗得住直接接所有日志, 尽量减少中间层, 就不用 fluent-bit 转发了.

## 读书

看完了山田宗树的<百年法>, 讲了人类研究出了不老化病毒, 等于实现了永生, 为了缓解人口压力, 各国又实施了百年法限制接种人类不老化病毒后只能活100年, 到期要安乐死, 之后在社会层面引发的一系列问题, 最后结局挺讽刺的. 作者的脑洞还是偏向思维实验, 感觉不够科幻, 如果把故事的时间跨度拉到千年, 那能扯的故事就挺多了. 读下来这调调总让我想起<午夜凶铃>的原作<环界>, 打分7/10.


读了黄奇帆的<结构性改革>, 对2015年开始政府的一系列动作看得更明白了一点, 比起08年的4万亿, 现在手法精致了不少, 体制内有高人啊, 不管你怎么看未来, 这本书值得一看.

<剑桥美国史> 看了一半, 刚看完南北战争那段, 只能说中国人对他们那套虚无飘渺的 manifest destiny 真的挺难理解的, 毕竟我们信的是王侯将相宁有种乎. 古人还是高, 早早看穿宗教的本质才有三武灭佛. 不管信不信教, 宗教在美国人的精神体系里都留下了很多烙印, 拜登是从肯尼迪之后第二个信天主教的总统, hmmmm.... 不知道后面4年会发生什么. 另外, 前几天感恩节, 随手看了下英文版 wikipedia 的词条, 通篇不提和印地安人的关系, 中文版倒是有, 这春秋手法值得国内媒体学习.

<李宗仁回忆录> 刚看了个开头, 这种自述的东西, 还是要辨证得看, 李宗仁真是变着法得夸自己, 就差出生时天降祥瑞了, 什么驯服烈马, 什么两个军头为了抢他来自己部队炒排骨(当排长)在妓院打起来, 远古龙傲天脚本.
