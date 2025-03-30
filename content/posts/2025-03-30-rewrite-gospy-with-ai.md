---
title: "gospy dev note2 (rewrite with aider)"
date: 2025-03-30T19:05:11+08:00
categories:
- tech
tags:
- gospy
- golang
---

几年前写过个工具 [gospy](https://github.com/monsterxx03/gospy), 用于从旁路 dump 一个 golang 进程的 runtime 信息(包括 goroutine, memory 等), 大致原理见以前的[文章](/tags/gospy/).


基本功能能用, 但没继续做下去, 除了没时间外, 其他还有几个问题:
- 不支持 MacOS (主要是没搞懂 MacOS 下怎么读取进程内存).
- DWARF 解析写的过于繁琐, golang 版本更新时, 解析逻辑很难调整.
- 对写 UI (包括 terminal UI 和前端) 实在没兴趣, 不写又没法暴露功能, 也懒得去做通过 http 接口暴露数据.

前阵子试了下通过 [aider](https://aider.chat/) 来写代码, 效果非常惊艳, 对我来说, 比 curosr 还顺手. 于是花了两个周末的时间, 把 gospy 整个重写了.

## 第一个周末

完全是在研究怎么搞定 MacOS 上读取内存的功能. 试了 deepSeek r1 和 claude 3.7sonnet 都没法给出最小可运行的 demo, 也试了 gork , 完全在口胡.

不过 deepseek 和 claude 给的思路还是对的, 要用系统调用 `mach_vm_read`, 只是两边都没法给出能编译的 cgo 实现, 怎么计算 mach-o 文件在虚拟内存里的基地址也讲的不清不楚.

后来还是去看了 delve 的代码, 这次我们有 AI 了, 看代码可方便多了, 直接让 aider 帮我定位了 dlv 在 mac 上读取内存的[相关代码](https://github.com/go-delve/delve/blob/a80904ca1f5da4ee99053c46b35f4d6a2124f824/pkg/proc/native/threads_darwin.c#L32), 依葫芦画瓢用cgo写了一遍, 要能跑还需要知道 mach-o 文件在进程启动后, 怎么计算出进程在虚拟内存里的基地址.

两段关键代码:
- [EntryPoint](https://github.com/go-delve/delve/blob/a80904ca1f5da4ee99053c46b35f4d6a2124f824/pkg/proc/gdbserial/gdbserver.go#L671)
- [loadBinaryInfoMacho](https://github.com/go-delve/delve/blob/a80904ca1f5da4ee99053c46b35f4d6a2124f824/pkg/proc/bininfo.go#L2013)

用 EntryPoint - image.StaticBase  就得到进程的基地址, 这边 AI 没帮上太多忙, 最后考改dlv 代码加调试信息定位出来的.

不过 dlv 中 EntryPoint 的获取作弊了, 是通过 gdb 拿的, 怎么自己写程序拿?

研究了半天, 最后通过 `mach_vm_region` 读到了 EntryPoint(vm region 类似 linux 下的 `/proc/{pid}/maps`).

终于实现了 MacOS 上读取进程指定虚拟内存地址的功能!

## 第二个周末

基本所有代码都是第二个周末写的, 两天靠 aider 写了 2000 多行代码, 中间还重构了很多次, 最后出来的代码只能说...比我写的好:), 这块代码之前写了得3个月, 当然这回是开了上帝视角, 在完全知道每个功能怎么实现的情况下做的, 所以能给 AI 下非常精准的指令.

自己封装了linux darwin 上的 ReadAt 函数, 剩下靠 aider 帮我脑补出了从指定内存地址读取 int8, uint32...等基础类型的代码.
进阶让它基于基础代码写出了读取golang 中 string, slice, pointer 的代码, 都能一次写对, 当初我可是用  readelf dump 了二进制中 dwarf 一点点看才明白怎么搞定这些功能的.

比如解析出目标进程中所有的 goroutine, 这个功能, 大致步骤:
- 从 DWARF 中获取  runtime.allgs 这个 全局变量的 offset, 用基地址 + offset 读出 allgs 的值, 是一个指向slice的指针
- golang 中slice 头是 3 个 byte(pointer to data, length, capacity), 用 data pointer 和 length 就能遍历出每个元素的内存地址, 这个就是每个goroutine 的内存地址
- 然后从 DWARF 中解析出 runtime.g 结构体中相关字段的offset, 中 goroutine  地址 + offset  就能读出字段值.

这块功能之前做的很折腾, 是定义了一个自己的struct, 然后通过 tag 和runtime包中相关结构体的字段做映射, 然后遍历 DWARF信息通过反射算出需要的所有字段的 offset: [reflect.go](https://github.com/monsterxx03/gospy/blob/1b35810157e26694082bf9556549431be40e6033/pkg/proc/reflect.go#L48)

反射实在恶心, 快把我写吐了, 这次重写, aider 给的方法很简单粗暴, 重复代码会多一点, 但可读性比我之前的方案更好, 反正是 AI 写, 我也不介意多花点 token.

很快就把基本功能写好了, 顺便用 tview 做了个terminal UI, 实在是快, 都不用看劳什子文档了, 生成的 UI 代码微调几下就能用.

不过写完后发现速度很慢, 在 mac 上读一个 prometheus 进程的信息,一次要 1s 多, 到这就凸显出人的用处了, 直接让 AI 分析性能瓶颈, 完全没给出任何有用的信息, 自己debug 了下,发现主要问题是读内存信息(memstats) 的 DWARF offset时, 传入的结构体名写错了, 导致多次遍历了全部 DWARF 结构, 返回的error 又被 AI 生成的代码忽略了(它本意是为了容错, 在读不到的情况下取上次的结果, 结果造成了很大的困扰).

修正后读取速度降到了 10ms, 继续分析代码, 觉得下面的瓶颈是每次读取内存必须调用 cgo, 能不能实现批量读取?

然后厉害的事情发生了, 我写了下面的注释:

```
// ai! 下面这段代码多次读取了内存, 能不能改成批量读取, 减少 cgo 调用? 不行的话帮我加个 TODO 的注释.
```

它就帮我实现了一个可用的  `readGoroutineBatch` 函数, 原理是从 DWARF 中直接找到 runtime.g 的结构体大小, 然后一次读出整个goroutine 的内存 buf, 还顺手把外层调用里去读goroutine 字段的代码也改了, 变成从读出来的内存buf 里直接拆数据, 虽然逻辑不算复杂, 但很繁琐, 手动写估计能倒腾一下午, 这个功能花了我大概5分钱完成了.

然后!!! 执行耗时就从 10ms 涨到了 20ms... 原因是它新实现了一个 `GetStructSize` 函数, 用于从 DWARF 中获取指定结构体的大小, 这个函数会在解析每个 goroutine的代码中被循环调用, 但值其实都是一样的, 分析出问题后, 让 AI 给这个函数加了个缓存逻辑, 执行耗时终于到了 3ms.

## Aider

[aider](https://aider.chat/) 是一个结对编程工具, 不依赖IDE/编辑器, 启动后以 shell 的形式运行, 可以自己配置使用的模型.

一开始我用的是它的 architect 模式, 用 r1 作为主模型, v3 作为编辑器模型, r1 用来思考问题, 生成代码 patch, v3 来合并代码(cursor 也是这么个原理), 效果比单独用 r1, v3 要好, 问题是 r1 的思考链太长了, 整个速度就会被拖慢.

后来 v3 发布了0324 版本, 试了下只用 v3, 发现效果就和之前的组合方式差不多了, 速度还快了很多倍, token 消耗也少了.

整个项目完成后, 算了下大概就花了15 块钱, 实在是划算.

## End

整个重写过程, AI 写了得有 90% 以上的代码, 自己只是调整下代码结构, 应该说相当愉快, 在知道怎么实现的情况下, 让现在的 AI 来做, 可以说是无敌的(v3-0324 完全够用).

如果自己不是特别清楚实现路径, 让 AI 自由发挥, 大概率会掉进坑里, 而大多数项目的障碍, 其实是需求分析这块就跑偏了.

对于 AI 编程工具的选择, cursor 很好用, tab flow 用起来相当舒服, 但经过这次对 aider 的深度使用, 感觉寄生于 IDE/Editor 的编程工具恐怕不是未来, 可能是类似 aider 这种更通用的形态,毕竟整个过程里我一次都没怀念过 curosr 的 tab, 模型再加把劲, 真的不需要手动编辑代码了, 或许要的只是, 动动你的小脑瓜?
