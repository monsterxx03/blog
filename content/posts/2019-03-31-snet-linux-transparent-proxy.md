---
title: "SNET: Transparent SS proxy on Linux"
date: 2019-03-31T22:19:46+08:00
categories:
  - tech
tags:
  - network
---

日常使用 Linux 工作, Linux 下实现全局透明代理可以用 iptables + ss-redir, 要有比较好的上网体验还需要 ChinaDNS 配合 dnsmasq,
这一整套在路由器上搞一遍就算了, 在本地太麻烦了. 仔细想想这几个加起来的功能实现起来也并不复杂, 前阵子就写了个小东西,
用一个进程完成全局透明代理 + ChinaDNS + 国内外分流: https://github.com/monsterxx03/snet  

目前的限制:

- 不支持 ipv6
- 只支持 tcp (因为我的测试服务器不支持 udp, 以后再加上吧)
- 上游 server 只支持 ss

目的是一个进程 + 一个配置文件完成所有事情. 需要的 iptable 规则也全部内置了(包括 CN ip 段), 缺少灵活但对我够用了, 以后有需要再加上选项不自动配吧.

需要手工装下 ipset.

配置文件示例:

    {
        "listen-host": "127.0.0.1",
        "listen-port": 1111,
        "ss-host": "ss.example.com",
        "ss-port": 8080,
        "ss-chpier-method": "aes-256-cfb",
        "ss-passwd": "passwd",
        "cn-dns": "114.114.114.114",
        "fq-dns": "8.8.8.8",
        "enable-dns-cache": true,
        "mode": "local" 
    }

ss 协议实现用的是 shadowsocks-go, 所以 cipher 就是 go 版支持的那些: https://github.com/shadowsocks/shadowsocks-go/blob/1.2.1/shadowsocks/encrypt.go#L159

cn-dns: 选择一个国内的 dns server.

fq-dns: 选择一个国外的干净的 dns server.

enable-dns-cache: 默认会按 A 记录的 TTL 缓存查询结果, 不需要可以关闭.

mode: 在桌面版 linux 上用选 local, 在 openwrt 路由器上选 router, 区别只是自动设置的 iptables 规则不一样. 

`sudo ./snet -config config.json` 就能跑起来啦, 并接管了全局的 TCP 流量.

## 实现



看一下用 local 模式启动时候到底干了什么, `iptables -t nat -L`:

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination
    SNET       tcp  --  0.0.0.0/0            0.0.0.0/0
    SNET       udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53

    Chain SNET (2 references)
    target     prot opt source               destination
    RETURN     tcp  --  0.0.0.0/0            0.0.0.0/0            match-set BYPASS_SNET dst
    REDIRECT   tcp  --  0.0.0.0/0            0.0.0.0/0            redir ports 1111
    RETURN     all  --  0.0.0.0/0            114.114.114.114
    DNAT       udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 to:127.0.0.1:1111


我建了一张叫 SNET 的 chain, 并把所有出去的 tcp 流量和查询 53 端口的 udp 流量(dns) 转到这个 chain 里.

解释下 SNET chain 里的内容: 

- 在 `BYASS_SNET` 这个 ipset 中的 ip 全部跳过, 里面是保留 ip 段 + cn ip 段 + ss server ip(不然就死循环啦).
- 剩下的所有 tcp 流量全部重定向到本地的 1111 端口(snet 的监听端口)
- 所有 114 dns 的流量不处理(配置文件里 设置的 cn-dns).
- 剩余的发往 53 端口 udp 全部转发到 snet 监听的 udp 端口.

`BYPASS_SNET` 的内容可以用 `ipset list BYPASS_SNET` 查看.

除了设置 iptables, 这个程序具体只干了两件事:

- 将转发过来的 tcp 流量用 ss 协议转发到 ss server.
- 绕过 dns 污染, 并实现 ChinaDNS 的功能. 

流量转发部分没什么好说的, 关键是要获取 tcp connection 的原始目标 ip + 端口, 因为被 iptables 重定向后目标就变成了 snet 的监听地址, 这里看了下 redsocks 的代码, 是通过 `getsockopt(fd, SOL_IP, SO_ORIGINAL_DST, dstaddr, &socklen)` 实现的: https://github.com/darkk/redsocks/blob/release-0.5/base.c#L223

DNS 部分讨了个巧, 一般处理 DNS 污染两种方式: 查询支持 EDNS 之类的加密协议的 DNS server, 或隧道到国外查询.

我目前的实现不支持 UDP, 就算支持了, 我也不想让 DNS 查询依赖上游 ss server 必须开启 UDP. 但现在既然已经有了 tcp 的加密隧道, 直接用呗, 没必要必须用支持 EDNS 的 server 呀.

 DNS 协议其实本来就是支持 TCP 的, 只不过一般的流程是 client 发送 udp 的 DNS 查询, 如果 response 超过 512 字节, DNS header 中有个 TC 比特位会被置为1, client 会用 tcp 再发起一次 DNS 查询, server 用 tcp 返回完成的 response.

直接用 tcp 向 server 发送 DNS 查询 也行: `dig baidu.com @1114.114.114.114 +tcp`.

所以现在 DNS 部分的实现是: 收到 iptables 转发过来的 DNS 查询后, 会同时向 cn-dns 和 fq-dns 发送 DNS 查询, cn-dns 直接走 udp(所以在 SNET 中跳过了 cn-dns), fq-dns 通过 ss 走tcp.

如果 cn-dns 的返回结果是国内 ip 就用 cn-dns 的结果, 是国外 ip 就用 fq-dns 的结果. 这是 ChinaDNS 的逻辑, 这么做的前提是目前墙对被污染的域名只会返回国外 ip, (不然国内就乱掉了). 如果该网站在国内外都有 CDN, 也不会出现被跳转到国外站点去的情况(淘宝, 微博...). 网易云之类的也不会因为用国外 ip 而无法使用.

顺便根据 A 记录的 TTL 对查询结果做了个缓存, 目前工作良好.

## 一些小坑

开始时候没看 RFC, 处理 tcp DNS 的时候直接把 UDP payload 通过 tcp 灌过去了, 以为 DNS 协议里是有 length, 结果怎么都不对, 后来才发现要在 UDP 的 DNS 数据包前面加两字节标记长度才行(DNS 协议的 header 里并没有包长度...)

测试 openwrt 的时候, 手头只有一台几年前的极路由2, 这东西架构是 32 位的 mips, 用 go 1.12 交叉编译后却跑不起来, 原来极路由的 cpu 不支持硬件浮点数运算指令, 需要在 openwrt 编译固件的时候开启 FPU 模拟, 重编译固件太麻烦啦, 后来查到 go 1.10 开始支持软件模拟浮点数,编译 mips 时加上环境变量 `GOMIPS=softfloat` 就行了.

有的发行版开了 ipv6 后会在 /etc/resolv.conf 里加入一个本地 ipv6 的 dns server 地址, 如果它在第一行, snet 的 dns udp 重定向就没用了，需要关闭 ipv6 或把这条 nameserver 删掉.

## 后续

目前个人的需求基本已经满足啦, 可能还需要的是:

- 按域名指定使用 cn-dns 或 fq-dns 的结果(有时还真是需要访问海外站)
- 加点统计接口:域名查询, 流量分析...
