---
title: "snet dev note: support MacOS"
date: 2019-06-20T16:03:56+08:00
categories:
  - tech
tags:
  - network
  - golang
  - snet
---

这两天得了空, 让 snet 支持了下 MacOS.

snet 的大致原理是通过系统防火墙的流量重定向功能,将所有去往国外的流量导到 snet 监听的端口, 在程序内部
将流量传递给上游的 proxy server(ss, http), 拿到响应后再回给客户端.

实现关键是要在 snet 内部获取到流量的原目标地址, 因为重定向之后 tcp connnection 的目标地址变成了 snet 监听
的地址.

Linux 上的实现，以前讲过: https://blog.monsterxx03.com/2019/03/31/snet-transparent-ss-proxy-on-linux/
是通过 `SO_ORIGINAL_DST` 这个 socket option 实现的.

MacOS 上没有 iptables, 类似的工具是系统自带的 pfctl, 捣鼓了一下也能实现一样的功能.

## 用 pfctl 做流量重定向

pfctl 的文档可以通过 `man pfctl`, `man pf.conf` 查看. 我也只是看了个大概, 细节并不清楚.

流量重定向需要用到 pf 的 `rdr` 规则, 但是 `rdr` 只能处理 incoming 的流量，对 outgoing 的流量无效. 

tricky 的地方是要把流量先重路由到 lo0, 再对 lo0 上的流量实行 rdr, 例子(顺序很重要):

    dev="en0"
    lo="lo0"
    rdr on $lo proto tcp from $dev to any port 1:65535 -> 127.0.0.1 port 1100  # let proxy handle tcp 
    pass out on $dev route-to $lo proto tcp from $dev to any port 1:65535  # re-route outgoing tcp

## 获取重定向后流量的原始目标地址

darwin 上没有 `SO_ORIGINAL_DST`, 翻了点 OpenBSD 的资料, 找到了几段通过 C 获取原地址的代码, 尝试用 CGO
做 wrapper, 最后试成功了.

Mac 上坑爹的地方在于, 虽然 darwin 内核里有 pf 模块, 但 MacOS 系统里没有对应的头文件，需要自己下载:

- https://github.com/opensource-apple/xnu/blob/master/bsd/net/pfvar.h
- https://github.com/opensource-apple/xnu/blob/master/bsd/net/radix.h
- https://github.com/opensource-apple/xnu/blob/master/libkern/libkern/tree.h

做了个 poc 的例子, 可以直接跑起来: https://github.com/monsterxx03/pf_poc

大致流程是:

- 初始化 struct `pfioc_natlook` 
- 填充 client socket 的 source ip, source port
- 填充 proxy bind socket 的 ip, port
- 打开 /dev/pf，执行 `ioctl(fd, DIOCNATLOOK, <pointer to pfioc_natlook>)`
- 然后　struct 的 rdaddr, rdxport 就被填充了，这就是我们需要的原始地址
