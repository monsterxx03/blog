---
title: "Tachi: agent 组网(mDNS,Gossip,TailScale) 和 A2A"
date: 2026-10-30T22:40:00+08:00
categories:
  - tech
tags:
  - tachi
  - ai
  - network
  - a2a
---

我一直觉得用 agent 干几种事情是没意义的:
- cosplay 人类工种(什么架构师，工程师，reviewer, designer....), 把这种人类社会的低效分工强行塞给 AI 不觉得很搞笑么.
- 单机上的并行 agent team, 不能说没用，在 tachi 里我也并行开 subagent，但这个本质上是把一个大任务拆解成可并行化的小任务去做，最后主 agent 还要汇总继续，目的主要在隔离上下文，但随着模型能力的增强，个人感觉这个的价值在渐渐降低，子 agent 返回的总归只能是摘要，存在信息损失，主 agent 还要再去做一遍 fact check, 到最后到底能省多少上下文，真难说.

个人认为： 多 agent 是有价值的，但价值在每个 Agent 都有自己的独立执行环境, 这个环境包括:不同的网络环境，能访问到不同的数据, 能有独立的 memory.

以我日常为例, 有这么几台机器在同时用:
- 家里一台 nas.
- 家里一台mini itx 小主机, 上面跑了 omarchy linux.
- 一台境外 vps.
- 自己的 MBP, 有时在家里，有时在外面.

只是想打通网络环境很简单，tailscale 之类的工具够多了，都行.

但每台机器上都跑一个 agent，怎么让 agent 之间互相感知呢？
- tachi@nas
- tachi@itx
- tachi@vps
- tachi@mbp

每个都在配置里写另外三个节点的 ip 地址? 当然可以，可哪天我要加个设备，或者那台 vps 挂了要换，还要去找到所有跑着 tachi 的设备换地址？

所以第一个问题是我要解决异构受信网络环境中的 agent 动态组网问题(mbp 不是常开，一会在线，一会不在，有时在家，有时在外), 然后再通过 a2a 来做任务委托, 每个节点都知道其他节点， a2a 就可以做成任意点对点.

## 什么是 mDNS

简单讲，就是在局域网里不依赖 DNS 服务器, 通过组播来实现服务发现，比如在局域网里连打印机，AirPlay/DLNA 投屏怎么找到电视, 都走的 mDNS.

机器启动时往 224.0.0.251:5353 宣告自己的存在, 发 udp DNS 包：

| 记录 | 类型 | 值 |
|-|-|-|
| _tachi._tcp.local. | PTR | Tachi@nas-9e47c7a3._tachi._tcp.local. |
| _services._dns-sd._udp.local. | PTR | _tachi._tcp.local. |
| Tachi@nas-9e47c7a3._tachi._tcp.local. | SRV | 0 0 9990 nas.local. |
| Tachi@nas-9e47c7a3._tachi._tcp.local. |  TXT |"id=9e47c7a3c6101f0b5cac9ce22c4d2b00" "v=1" "scheme=https" |
| nas.local.  | A | 192.168.11.166 |

几种记录的作用:
- PTR 记录用于宣告： 我是 `_tachi._tcp` 服务的一个实例
- SRV 记录用于宣告:  这个服务实例指向 nas.local:9990 这个地址
- TXT 记录，理解成服务自己写的 metadata， id 在我的系统内就是一个tachi 节点的唯一 id，用来固定身份.
- A 记录： 把nas.local 这个主机名翻译成 ip 地址.

宣告的包会被其他机器上的订阅组播地址(224.0.0.251)的进程(即其他 tachi 实例)收到， 从 dns 包里拼出连接地址 https://192.168.11.166:9990, 并且知道它的节点 id 是 9e47c7a3c6101f0b5cac9ce22c4d2b00(后续会和 mTLS 里的证书公钥算出的 node id 做比对).

到此， mDNS 就结束了，目的是在 0 配置的情况下，让局域网里的 tachi 实例互相发现.

## 什么是 Gossip

mDNS 只能在局域网里生效，对于在公网的 vps 怎么办？

Gossip 是传播“我认识的 tachi” 的方式.

具体到配置，局域网里只要有一台机器配置那台公网 vps 地址即可，比如在我 mbp 上的 tachi 配置:

```yaml
peers:
    - url: https://49.199.10.22:9990
```

tachi@mpb 启动后就会连上去，把这台 tachi@vps 加进自己的已知列表里，此时我的 tachi@mpb 就同时知道局域网里的所有其他 tachi + tachi@vps.

局域网里的 tachi@nas, tachi@itx, 会定时向自己认识的邻居(通过 mDNS 发现的那些)，询问:"你还认识谁"， 问到 tachi@mbp 时，就发现了那台公网的 tachi@vps, 一定时间后，局域网里的所有机器就都认识了 tachi@vps.

后续局域网里来了新机器，先通过 mDNS 找到其他局域网邻居，再通过 gossip，问邻居得知这台 tachi@vps 的存在，整个过程 0 配置。

如果后面我又买了台国内的 vps 呢(tachi@cn-vps)？ 

局域网里任意一台机器的 peers 配置里只要加上它，最终通过 gossip，所有其他机器全都会认识它，这就是“流言蜚语”的好处.

## 用 mDNS + Gossip + TailScale 来组网

上一节的实例里，那台作为种子连接目标的 gossip 对象用的是公网 ip，但如果这么配的结果是，所有局域网的机器都能连上它，但它没法连上局域网里的设备.

最简单的解决方法就是把公网 vps 和家里设备通过 tailscale 来组网， 大家一起进到 100.x.x.x 的 tailscale 网段来， 这样公网 vps 也连上了局域网 tachi 节点.

## A2A

a2a 主要解决 agent 之间的任务委托， 每个agent 节点用一个 card 来表明自己的身份, tachi 中返回的实际 card 格式如下:

```json
{
  "name": "Tachi@nas",
  "description": "跑在家里 nas 上(fnos), 可给用户发送微信消息",
  "version": "dev",
  "supportedInterfaces": [
    { "url": "https://100.12.47.91:9990", "protocolBinding": "HTTP+JSON", "protocolVersion": "1.0" }
  ],
  "capabilities": { "pushNotifications": false, "streaming": false },
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain"],
  "securityRequirements": [{ "schemes": { "mtls": {} } }],
  "securitySchemes": {
    "mtls": { "mtlsSecurityScheme": {
      "description": "A Tachi node's own self-signed certificate. Any such certificate is accepted: this node identifies callers rather than authorising them."
    }}
  },
  "skills": [
    { "id": "fnos", "name": "fnos", "tags": null,
      "description": "Use trim-cli to sign in to and manage fnOS devices, including files, photos, applications, Docker, downloads, logs, monitoring, networking, storage, users, and Baidu Netdisk. …" }
  ]
}
```

agent 之间通过前几节的 mDNS + gossip 得知节点地址 100.x.x.x:9990.

然后通过 GET https://100.x.x.x:9990/.well-known/agent-card.json 得到每个节点的身份, 内存里就有了所有 tachi 节点的身份信息(地址是什么，叫什么，能干什么).

在 agent 层面，我做了几个事:
- 每轮 user prompt 里通过 system reminder 的形式注入当前可达的 agent 列表<name: tachi@xxx, node id: xxxxx>
- 提供两个工具: DelegateTask(按node id 来选 tachi 节点委托任务，返回 task_id), RemoteTask(根据 task_id 查询任务状态).

![a2a](/posts/images/a2a.png)

task_id 可以理解成常规agent loop 里的一个 turn, 每次任务委托会有个contextId, 可以理解成 agent loop 的 session, DelegateTask 时支持传continue_from 参数，传入对应的contextId, 就续上了远端之前的session，达成了多轮会话的效果.

在我的实现里， contextId 并不完全等于 session id, 原因在于我的 session 压缩后是创建一个新 session，新 session 的 parent_id 指向之前被压缩的 session，所以是个链式的，contextId 总是指向同链上最新的 session.

## 实际使用场景

nas 上跑的 tachi 接了 weixin 的 ilink 协议，国内网络稳定，可以给我发微信消息, vps 上跑的 tachi 接了 discord，我在上面分了一堆 channel，订阅我感兴趣的 newsletter，blog 等信息, 境外网络稳定，不依赖代理，但访问 discord 本身要代理.

我在本地 mpb 的 tachi 上跑着长任务，然后打算出门了，就补上一句，跑完了把结果发给 tachi@nas，让他用微信通知我， 于是过了半小时，我收到了任务结果.

在 mpb 上干着活，刷着 b 站，看到某部电影的推荐，在本地 tachi 里说：“让tachi@nas 去下 xxx”, 于是 nas 上的 tachi 找到磁力链接，加进了 fnos 的下载队列, 不需要切到微信里去说.

目前在个人使用场景里 a2a 不能算非常有用，主要意义是本地上下文接上远端环境，但远端收到消息的那个会话里回话没法接上本地的会话，比如上面那个出门后收到微信任务结果的场景里，我没法直接回话说：“继续干 xxx”, 可以让它开个新委托给 mpb 干活，但对 mpb 来说，是个新 session 了，接不上之前的 session.
