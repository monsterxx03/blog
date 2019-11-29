---
title: "gospy dev note"
date: 2019-09-29T15:07:11+08:00
categories:
- tech
tags:
- gospy
- golang
---

[前文](/2019/09/20/gospy-non-invasive-goroutine-inspector/)讲了下 gospy 的大致用法, 这篇记录具体实现和过程中碰到的一些问题.


## 原理

要从外部获取 golang 进程的 runtime 信息, 需要做得是从进程的 binary 中的 debug 信息里 parse 出需要的一些变量的虚拟内存地址, 读取目标进程的内存, 得到相应的数据,
将两者映射起来就好了.只支持了 linux 上的 ELF 格式 binary, debug 信息是 go 在编译时候弄进去的, 格式是通用的 DWARF.


ELF 和 DWARF 格式本身不细究(汗, 文档几百页也实在看不动), go 标准库里自带相应的 parser: `debug/elf, debug/dwarf, debug/gosym`. 一个基本的读取例子:

	f, _ := os.Open(path)
	b, _ := elf.NewFile(f)
	lndata, _ := b.Section(".gopclntab").Data()
	ln := gosym.NewLineTable(lndata, b.Section(".text").Addr)
	symtab, _ := gosym.NewTable([]byte{}, ln)

在 go 1.3 之前, elf binary 里有一节 session: `.gosymtab`, 1.3 开始不需要了,　为了 api 兼容, `gosym.NewTable` 还需要这个参数,传个空　byte slice 进去就行.

得到的 `symtab` 主要作用是根据一个 PC(program counter) 值, 得到它对应的函数名，文件，行号信息. 这里和一般的 debugger(gdb, lldb) 实现原理类似, 不过 debugger 为了中断程序
需要向目标进程的寄存器写入数据, 而 gospy 只是读取, 要简单很多. 一般的 debugger 下断点的过程是获取目标进程的所有线程, 用 `PTRACE_ATTACH` 暂停所有线程, 从 RIP 寄存器里读出该
线程的当前 pc 值, 再查符号表得到函数文本信息. 

但 golang 运行代码是以 goroutine 为单位, 要获取用户空间的代码信息, 我必须能定位到单独的 goroutine, 而不是系统线程, 怎么做? 开始也迷惑了很久, 后来看到篇文章 [building a go debugger](https://backtrace.io/blog/backtrace/building-a-go-debugger/), 才找到思路.


需要从 elf binary 里 parse 出 `runtime.allgs` 和 `runtime.allglen`, 前者是 hold 所有 goroutine 的 slice, 后者是 slice 的当前长度, 有了这两个值就能算出
每个 goroutine 的当前内存地址, `runtime.g` 是代表 goroutine 的结构体，把它的每个字段 offset 往 goroutine 的内存地址上一映射，就能得到每个字段的值.

`runtime.g` 里有两字段 `gopc` 和 `startpc`, 前者是 goroutine 当前运行函数的 PC 值, 后者是触发这个 goroutine 的函数的 PC 值, 相当于提供了两种视角来看 goroutine,
调用 `symtab.PCToLine(pc)`, 就能得到对应函数信息. golang 的 GMP 调度模型里的其他 P, M, sched, 信息也能用相同的方式 parse 出来.

## 从 DWARF 中 parse 出 struct

DWARF 里 parse 变量地址比较简单，不提了, struct 比较搞. 开始看了下 delve 的实现，它实现了一套比较完整的 DWARF 地址转换代码, 对我的需求来说过于复杂. 先手工 dump 了下 ELF 中的 DWARF 信息, 看看有什么规律:

    readelf -W --debug-dump=info <elf file>

搜索了一下 `runtime.g` 这个结构体, 大致结构如下:

    <1><3ac4b>: Abbrev Number: 37 (DW_TAG_structure_type)
        <3ac4c>   DW_AT_name        : runtime.g
        <3ac56>   DW_AT_byte_size   : 376
        <3ac58>   Unknown AT value: 2900: 25
        <3ac59>   Unknown AT value: 2904: 0x68660
    <2><3ac61>: Abbrev Number: 22 (DW_TAG_member)
        <3ac62>   DW_AT_name        : stack
        <3ac68>   DW_AT_data_member_location: 0
        <3ac69>   DW_AT_type        : <0x3af88>
        <3ac6d>   Unknown AT value: 2903: 0
    <2><3ac6e>: Abbrev Number: 22 (DW_TAG_member)
        <3ac6f>   DW_AT_name        : stackguard0
        <3ac7b>   DW_AT_data_member_location: 16
        <3ac7c>   DW_AT_type        : <0x395ed>
        <3ac80>   Unknown AT value: 2903: 0
    <2><3ac81>: Abbrev Number: 22 (DW_TAG_member)
        <3ac82>   DW_AT_name        : stackguard1
        <3ac8e>   DW_AT_data_member_location: 24
        <3ac8f>   DW_AT_type        : <0x395ed>
        <3ac93>   Unknown AT value: 2903: 0

    ...
    <1><3af4b>: Abbrev Number: 38 (DW_TAG_typedef)
        <3af4c>   DW_AT_name        : runtime.g
        <3af56>   DW_AT_type        : <0x3ac4b>


最左边的 `<1>, <2>`表示层级，所有 struct field 都在第二级. `<3ac4b>` 这样的是该字段在 binary 中的 offset(其实就能当虚拟内存地址, 进程启动的时候直接把 binary mmap 进内存了嘛).

`DW_TAG_structure_type` 这一节里的 `DW_AT_byte_size` 是整个 struct 的大小, 每个字段都是　`DW_TAG_member`, 里面的 `DW_AT_DATA_member_location` 是该字段在 struct 内的 offset, 减去上一个字段的 offset 值，就能得到该字段的 size, 遍历一次就能得到整个 struct 结构.

## 读取进程内存

从 DWARF 里能得到需要的变量的虚拟内存地址，拿着这个地址去读内存就能得到需要的数据. 怎么读呢? 有三种方法.

一般做法是先 `PTRACE_ATTACH`, 暂停进程, 然后 `PTRACE_PEEKDATA`, 读取对应内存. 限制是 `PTRACE_PEEKDATA` 每次只能读取一个 long, 8 字节, 要读取一块连续内存就要切片多次调用.

[py-spy](https://github.com/benfred/py-spy) 用的是 `process_vm_readv`, 从 linux 3.2 开始支持, 可以读取连续内存.

而我找了个更偷懒的方法, 直接读 `/proc/<pid>/mem`, 这个虚拟文件就是进程的虚拟内存, 可以当普通文件一样读, 效率比调用 syscall 高很多, golang 里一次 syscall 差不多都要几十 us.

为了得到一致的内存视图, 还是要用 `PTRACE_ATTACH` 先暂停线程, 用着就发现了 golang 的一个坑, ptrace 的所有调用必须和 `PTRACE_ATTACH` 来自同一个线程, golang 没法控制运行线程, 后续 deatch 会报找不到线程的错. 看了 delve 的源码找了个 workaround. spawn 一个 goroutine 专门处理 ptrace 调用, 在这个 goroutine 里用 `runtime.LockOSThread()`, 锁定该 goroutine 的运行线程, 把要执行的 ptrace call 通过 chan 传进去: https://github.com/go-delve/delve/blob/v1.3.1/pkg/proc/native/proc.go#L391

我的做法其实对跑在容器里的进程也管用, 但一开始没试, 直接照着 py-spy 的方法尝试用 `setns`, switch 进目标进程的 namespace 再搞事, 进 mnt namespace 的时候就老是报参数不对的错, 发现还是 goroutine 在系统线程上调度带来的坑: https://stackoverflow.com/questions/25704661/calling-setns-from-go-returns-einval-for-mnt-namespace.

不太明白的是为什么像上面 ptrace 那样锁定线程也不行, workaround 是要在 go 的 runtime 初始化前通过 cgo 来执行 setns, 但这样没法动态得传递目标 pid 信息. docker 里的单独 build 了一个 lib [nsenter](https://github.com/opencontainers/runc/tree/master/libcontainer/nsenter), 把 setns 相关的工作放在了一个子进程里用 C 去做, 但交互比较麻烦, 我就没尝试了.
