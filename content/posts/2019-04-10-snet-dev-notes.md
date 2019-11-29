---
title: "snet dev note"
date: 2019-04-10T22:33:49+08:00
categories:
  - tech
tags:
  - network
  - golang
  - snet
---

完成 [SNET](https://github.com/monsterxx03/snet) 初版后又做了些后续更新,　记录一点.

## 支持 http tunnel

配置文件里增加一个 `proxy-type` 选项, 默认为 `ss`, 可改成 `http`, 这样可以将
支持 http tunnel 的代理服务器作为 upstream(例如 squid). 填上 `http-proxy-` 开头
的选项就行.

实现上 client 端要对接 http tunnel 非常简单:

- client 发送请求: `Connect tgt-host:tgt-port HTTP/1.1`
- server response: `HTTP/1.1 200`, 即表示 server 端支持 http tunnel
- client 后续向该 tcp connection 写入的数据都会被 server 转发到 tgt-host:tgt-port


改动的时候把 upstream proxy 的部分重构了一下, 抽了个 `Proxy` interface 出来, 后续想对接其他协议方便扩展.


## 对 udp 支持的尝试


对 tcp 流量的转发能通过 iptables REDIRECT 实现的, 通过 getsockoption 可以知道 tcp connection
的原目标, 但这对 udp 行不通, REDIRECT 之后拿不到原 target. 

shadowsocks-libev 对 udp 转发的支持是通过 iptables 里的 tproxy target 实现的, 尝试了一下 tproxy, 但是 tproxy 只能
用在 prerouting chain 里. 那意味着只能当该程序所在机器作为网关的时候才能接收到流量, 我写这个主要不是为了在路由器上
而是本机用, tproxy 就 pass 了.

还有一种方式是通过 tun 设备来实现流量接管, tun 工作在 3 层上, 从上面读出来的流量是裸的 ip 数据包. badvpn 之类的工具
实现了 tun2socks 的方式, 用 lwip 来在用户态实现 ip/tcp reassembly, 得到 tcp 流之后再转发给 socks5 程序. 好处是可以实现跨平台的流量接管.

还看到了一种非常扭曲的实现方式, 通过 raw socket 得到出去的 udp, 再用 普通的 socket 连回 udp client socket, 将数据写回:
https://github.com/EtiennePerot/sshuttle/blob/master/firewall.py#L201
脑洞很大, 看上去很麻烦...没尝试.

目前我对 udp 实在没什么需求, 暂时没什么动力弄了, 要做的话估计只能通过 tun 来搞(引入 lwip 总有点杀鸡用牛刀的感觉).

## dns 处理的一个 bug

用了一整子一直都很稳定, 但今天在用 kubectl 倒腾 k8s 的时候发现了问题, k8s cluster 在国外, api server 也是国外 ip.
开了 snet 后用 kubectl 巨慢无比, 而且经常 timeout. 关了 snet 直连却很快.

在 upsteam server 上试了下, 到 k8s api server 速度没问题.

`kubectl version -v=10` 能打印出对应 http request 的 curl 命令, 拷贝出来用 curl 直接试了下, 也没问题, 只有用 kubectl 的时候才会 timeout.

抓包看了下 kubectl 发起 api 请求时候, 先尝试解析了 AAAA 记录, 如果 AAAA 记录不存在, 再尝试 A 记录, 貌似调用了 glibc 里 gethostname 函数的默认行为
都是这样的.

问题出在我实现的 DNS 缓存上, 简单的根据查询域名缓存了查询结果, 实际上我只解析 A 记录, 所以从缓存里取出来的都是 A 记录的结果. 

重现方式:

- dig www.baidu.com
- dig -t AAAA www.baidu.com

第二次 dig 就会报 query result mismatch, 因为返回的是第一次的 A 记录. 比较坑的是 kubectl 报的是 dail timeout, 看上去像连不上 tcp 端口, 比较难看原因.

修复很简单, cache key 把 query type 也加上就行啦.
