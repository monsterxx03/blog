---
title: "Python Web 应用性能调优"
date: 2017-07-01T23:38:24+08:00
categories:
    - tech
tags:
    - glow
    - server-infra
---

# Python web 应用性能调优

为了快速上线，早期很多代码基本是怎么方便怎么来，这样就留下了很多隐患，性能也不是很理想，python 因为 GIL 的原因，在性能上有天然劣势，即使用了 gevent/eventlet 这种协程方案，也很容易因为耗时的 CPU 操作阻塞住整个进程。前阵子对基础代码做了些重构，效果显著，记录一些。

设定目标:

- 性能提高了，最直接的效果当然是能用更少的机器处理相同流量，目标是关闭 20% 的 stateless webserver.
- 尽量在框架代码上做改动，不动业务逻辑代码。
- 低风险 (历史经验告诉我们，动态一时爽，重构火葬场....)

## 治标

常见场景是大家开开心心做完一个 feature， sandbox 测试也没啥问题，上线了，结果 server load 飙升，各种 timeout 都来了，要么 rollback 代码，要么加机器。问题代码在哪?

我们监控用的是 datadog (statsd协议)，对这种问题最有效的指标是看每个接口的 `avg_latency * req_count` 得到每个接口在一段时间内的总耗时，在柱状图上最长的那块就是对性能影响最大的接口。进一步的调试就靠 cProfile 和读代码了。

但很多时候出问题的代码逻辑巨复杂，还很多人改动过，开发和 sandbox 环境数据的量和线上差距太大，无法复现问题，在线上用 cProfile 只能测只读接口(为了不写坏用户数据)。

而且这种方式只能治标，调试个别慢的业务接口，目标里说了只想改框架，提高整体性能，怎么整?

## 治本

我希望能对运行时进程状态打 snapshot，每次快照记录下当前的函数调用栈，叠合多次采样，出现次数多的函数必然就是瓶颈所在. 这思想在其他语言里用的也很多，其实就是 Brendan Gregg 的 flamegraph.

以前内部做过类似的事情，不过代码是侵入式的，在运行时通过 signal, inspect, traceback 等模块，定期打调用栈的 snapshot, 输出到文件，转成 svg 的 flamegraph 来看，但是 overhead 太高，后来弃用了。

后来利用了 uber 开源的一个工具: https://github.com/uber/pyflame, 可以非侵入式得对运行中的 python 进程做 snapshot, 输出成 svg.

效果如图:

![flamegraph](https://cloud.githubusercontent.com/assets/2734/17949703/8ef7d08c-6a0b-11e6-8bbd-41f82086d862.png)

横条越长的部分，表示被采样到的次数越多，从下往上可以看到在每一层上的函数耗时分布。

使用非常简单:

`pyflame -s 60 -r 0.01 ${pid} | flamegraph.pl > myprofile.svg`

- -s 60， 总采样时间为 60s
- -r 0.01， 以0.01s 的频率做采样

在最终的输出图上可能有比较长的 IDLE 时间, pyflame 只能捕获到当前获取了 GIL 的代码的调用栈, 其他的部分就会是 IDLE, 包括几种情况:

- IO wait, 比如 call 一个很慢的 rpc server， client 等待过程中，采集到的时间就是 IDLE
- C 编写的部分
- 进程处于空闲时间。

大体可以认为 pyflame 上采样到的部分是 CPU heavy的代码。

通过 pyflame, 可以很快得对进程运行时耗时分布有个大概的感觉，即使你完全不了解业务逻辑.

## 重构

线上 web 应用，前面是基于 flask的 web 端和api server, 后面是几组业务不同的 RPC server，两者之间通过 msgpack 通信. 为了方便， RPC server 也是基于 flask 的，通过 pyflame 调试，发现 flask 的 overhead 还是很高的，在 RPC 那层， 一些接口实际业务代码的采样次数，只有总采样的1/6左右 (并不能反应实际耗时分布)，其余都耗在了 flask 层。

### RPC server

RPC 层不处理web逻辑， flask完全用不到，可以干掉。有想过替换成 thrift/protobuf 这种二进制通行协议，传输的数据不带 schema 信息，效率能高不少，但这样势必要大改接口，还要考虑之后schema改动，升级时候server 和 client 端的兼容性问题。本着不动业务代码和低风险的原则，还是保守的 http + msgpack.

对于 RPC server, 索性跳过 web 框架，直接实现 WSGI，参考 [pep333](https://www.python.org/dev/peps/pep-0333/) , 非常简单，改完 rpc server入口代码不到200行，用 wrk 做下 helloworld 的 benchmark, 并发轻松变3倍.


### RPC client

改完 rpc server 层，负载已经有了显著降低(20% 左右)，还有个性价比很高的优化是替换 rpc client. 之前用的是 requests, 说实话，个人对这种接口漂亮，使用方便的库一直是持保留态度的，尤其是在这种性能敏感的场景，在 pyflame 的采样图上也能看到 requests 代码里的耗时很长.

尝试用 https://github.com/gwik/geventhttpclient 替换掉 requests. 简单的 benchmark 脚本测试下来，完成相同的请求数， geventhttpclient 只用了 requests 1/4 的时间 (gevent patch 过的情况下).

修改完 RPC client 的代码，上线后却傻眼了, server load 降得很明显，可是latency 却直接上升了 30% 多???

经过排查，发现替换 client 过后，内网流量莫名增加了，拿两台机器做 A/B testing, 效果很明显。开始怀疑是 geventhttpclient 的 connection pool 实现有问题，导致 tcp 连接没有复用。

尝试用 tcpdump 抓 sync 包: `tcpdump "tcp[tcpflags] & (tcp-syn) != 0"`

对比了 requests 和 geventhttpclient 的两台机器，syn  包的数目并没有太大差别。

但抓包过程中偶然发现，geventhttpclient 在发送 http 请求的时候，header 和 body 竟然是用两个 packet 发送的, requests 底层是用的标准库的 httplib, 会将 header buffer 起来和 body 通过一个packet 发出去，所以每发一次请求，geventhttpclient 会多发一个 ip + tcp header(40字节)，怪不得流量变多了。

把这个问题修了下, 上线后 latency 立刻回复了正常。顺手把改动推到了官方: https://github.com/gwik/geventhttpclient/pull/85

## 总结

经过一轮修改，最后关闭了30% 的 stateless server. 总共动到的代码也就几百行，业务开发无感知。应该说性价比很高。

在复杂业务逻辑下，调试性能问题总是特别头疼，单机的 benchmark QPS 数据也就估个天花板，意义不大，关键还是要完善监控和工具链，帮助快速定位问题。下一步打算上 opentracing, 完善分布式环境下的性能追踪.
