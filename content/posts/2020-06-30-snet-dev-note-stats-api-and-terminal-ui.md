---
title: "snet dev note: stats api and terminal UI"
date: 2020-06-30T14:40:01+08:00
categories:
- tech
tags:
- snet
- network
- golang
---

从 [0.10.0](https://github.com/monsterxx03/snet/releases) 版本开始给 snet 加了 stats api 来暴露内部的一些统计数据.


设置 `"enable-stats": true` 开启, 默认监听 8810 端口, `curl http://localhost:8810/stats`

        {
            "Uptime": "26m42s",
            "Total": {
                "RxSize": 161539743,
                "TxSize": 1960171
            },
            "Hosts": [
                {
                    "Host": "112.113.115.113",
                    "Port": 443,
                    "RxRate": 0,
                    "TxRate": 0,
                    "RxSize": 840413,
                    "TxSize": 172528
                },
                ...
            ]
        }


分 host, 统计从该地址接收的字节数(RxSize), 发送字节数(TxSize), 和相应的 rate/s (RxRate, TxRate).

默认只记录 ip, 可以设置 `"stats-enable-tls-sni-sniffer": true`, 开启对去往 443 端口的流量进行 sni sniffer, 尝试解析出域名.
`"stats-enable-http-host-sniffer": true`, 对发往 80 端口流量尝试解析 http host 字段, 两者都会给连接建立过程增加一些 overhead.

server 开启 stats api 后, 用 `./snet -top` 可以显示一个类似 top 的 terminal UI 作流量监控.

![](https://raw.githubusercontent.com/monsterxx03/snet/master/images/top.gif)


下载上传速率的计算, 逻辑上很简单,用一段时间内的流量除以时间就可以了, 具体实现的时候讨了个巧, 用了 `container/ring` 里的环形队列结构, 如果统计速度的范围是N秒,
就建立一个 size 为 N+1 的 ring, 每秒 tick 一下, 把当前的总流量更新到 ring 上当前节点, 并下移一个节点, 计算 N 秒平均速率即 `(node(-1).Val - node(-N-1).Val) / N`.
任何区间小于 N 的范围速率都能用这个结构计算. 用环形队列省了清除 N 秒前数据的逻辑.