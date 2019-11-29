---
title: "snet dev note"
date: 2019-11-29T12:00:01+08:00
categories:
  - tech
tags:
  - network
  - golang
  - snet
draft: true
---


[snet](https://github.com/monsterxx03/snet): 0.5 ~ 0.6.1, 整理从[上一篇](/2019/06/20/snet-dev-note-support-macos/)以来的一些更新.

## 新增选项

`proxy-scope`, 默认 `bypassCN`, 可选 `global`. `bypassCN` 会做国内外分流, `global` 直接让所有流量去往国外.


`host-map`, 为域名指定 ip. 之前在测试一个功能的时候需要在内网让手机对某个域名的解析切换到我的测试 ip 上, 坑爹的是公司的路由器竟然没这功能, 索性在 snet 里写了这个功能, 让我的台式机发射 wifi, 手机连上来, snet 的 `mode` 切换成 `router`, `listen-host` 改为 `0.0.0.0` 就好了.


`block-hosts`, 因为 `block-host-file` 里的域名是从一个现成的列表生成出来的, 不好支持通配符, 所以加了这个, 比如 `["*.hpplay.cn"]`, 就能把电视上所有乐播投屏的广告干掉.


`proxy-type` 新增 `tls`, 一个基于 tls 的简单自定义协议, 使用这个需要部署一个 snet 的 server 端, 详细看 README 吧.


## Bug

之前对 ssh 长链接的支持有问题, 很容易断, 因为只在连接建立的时候设置了 timeout, 修改了下 piping 的逻辑, 在每次读写新数据后都刷新一下 timeout 时间就好了.

## CI/CD

之前用的 Travis CI, 迁移到了 github actions, 尝试在打 tag 的时候自动发 release. 因为在 mac 上用到了 cgo, 没法从 linux 交叉编译过去. 开始想在两个
系统上分别编译 binary 再上传到一个 release 里去, 但不知道怎么只执行一次 `create_release` 并获得它的上传地址.


提了个 post: https://github.community/t5/GitHub-Actions/Upload-multi-assets-from-different-os-to-a-release/m-p/39725, 里面有人提到了一些解决方法, 但好像很烦, 我没有尝试.

因为在 linux 没有用到 cgo, 所以直接在 mac 上交叉编译 linux 版本得了.
