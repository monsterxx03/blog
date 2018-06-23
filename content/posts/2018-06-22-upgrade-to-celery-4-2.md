---
title: "升级celery 到 4.2.0 碰到的坑"
date: 2018-06-22T16:10:41+08:00
categories:
    - tech
tags:
    - python
    - celery
---

在把代码往 python3 迁移的过程中需要升级一些第三方库, 升级了 gevent 后发现 celery 有问题, 于是尝试把 celery 从3.1.25 升级到 4.2.0, 中间碰到了很多问题, 记录一点. 

## 配置的变化

`CELERY_ACCEPT_CONENT` 之前默认是都允许的,  4.0 开始默认值只允许 json, 因为我用的是msgpack, 所以需要修改这个配置让它接受 msgpack.

`CELERY_RESULT_SERIALIZER` 之前默认是pickle, 现在默认也变成了json, 如果task 的返回结果是 binary 的话, json 无法处理,要么把结果 base64 编码, 要么把`CELERY_RESULT_SERIALIZER` 配置成 msgpack,  pickle 明显 py2 / 3 不兼容, 没用.

## CELERY_RESULT_BACKEND 使用 redis 的坑

配置了 `CELERY_RESULT_BACKEND` 后, 会把 task 执行结果存起来, 用redis 做backend 支持 expire, 默认 1 天.

我的 worker pool 是 gevent, 升级 4.2.0 上线之后, 报了很多奇怪的错, 全是把 task 插入 redis 时候报的错, 错误原因大致是因为redis client 
的 socket 在不同的 greenlet 中被使用造成的, 所以有时候会尝试使用一个已经被关闭的 socket,  有时有 socket 还没有被建立, 而且全是在调用
redis subscribe channel 的时候出的错, channel list 还很长.

在 celery 4.0 中, 如果设置了 `result_backend`, 并且 task 没设置 `ignore_result=True` 的话, 在将 task 加入队列的时候, 会用 redis 的pubsub
来监听 `celery-meta-{task_uuid}` 这个channel, 但 redis client 的 pubsub 不是线程安全的, 也不能跨 greenlet 使用.

redis client 的 pubsub 在建立socket 后会执行 `self.on_connect` callback: https://github.com/andymccurdy/redis-py/blob/master/redis/client.py#L2369, 目的是在重连的时候重新 sub 之前的 channnel. 但是 gevent patch 之后, socket 的建立会变成异步的, 并行建立执行redis pubsub 的`execute_command` 的时候会重复subscribe channel.

原因是猜测, 我也没在开发环境完全复现线上的错误, 但写了个脚本证实 subscribe 的问题:

``` python
import gevent
import gevent.monkey
gevent.monkey.patch_all()

from celery import Celery

app = Celery()
app.conf.update(
    BROKER_URL='redis://localhost:6379/1',
    CELERY_RESULT_BACKEND='redis://localhost:6379/1'
)

@app.task
def wrap():
    pass

def run():
    gs = []
    for i in range(1000):
        gs.append(gevent.spawn(wrap.delay))

    gevent.joinall(gs)

run()
```

脚本并不会报错, 但如果在执行的时候另一边用 redis-cli 连上 redis, 执行 `monitor` 命令, 能看到在开始建立链接的时候, 的确有重复 subscribe channel 的调用. 没想到简单的修复方法, workaround 有两个:

1. 禁掉 `result_backend`, 所有的执行结果都不会存储, 调用`AsyncResult` 的 `get()`, 会报错.
2. 在task 上加上 `ignore_result=True` 的 option, 这样单个 task 的结果不会存储, 也不会调用 redis 的 subscribe 了.

我大部分的 task 都不需要存结果, 少数几个需要获取异步执行结果, 这几个task 执行很少, 基本没啥并发, 所以我还是定义了 `result_backend`, 但把大部分 task 都设置了 `ignore_result=True`. 

详情可以看这个 issue: https://github.com/celery/celery/issues/4670

## AsyncTask 的内存泄漏

改动完上线跑了一天后,  发现好几个 service 内存占用异常高, 频频触发监控的内存报警.

因为升级了 gevent 和 celery, 开始不太确定问题出在哪. 但是内存异常的 service 并不是 celery 的worker 进程, 但会执行大量的 `apply_async` 的插入task 的操作.

所以我猜测问题只出在 `apply_async` 这个函数上. 开始尝试在本地复现问题.

写了一个简单脚本测试 `apply_async`, 因为不涉及执行 task, 所以直接不引入 gevent:

``` python
import resource

from celery import Celery

app = Celery()
app.conf.update(BROKER_URL='redis://localhost:6379/1')

@app.task
def dummy():
    return '1'


def print_mem():
    print 'Memory usage: %s (kb)' % resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

def run():
    for i in range(10000):
        dummy.delay()
        if i % 1000 == 0:
            print_mem()

run()
```

每执行 1000 次 task 打印下内存占用, 这个脚本在 celery 3.1.25 下执行内存占用非常平稳, 在 4.2.0 里内存稳步增长.

```
Memory usage: 31736 (kb)
Memory usage: 32916 (kb)
Memory usage: 34628 (kb)
Memory usage: 36432 (kb)
Memory usage: 38292 (kb)
Memory usage: 39812 (kb)
Memory usage: 41668 (kb)
Memory usage: 42608 (kb)
Memory usage: 43576 (kb)
Memory usage: 44592 (kb)
```

好吧, 确定问题出在 celery 4.2.0 的 `apply_async` 里.

关于 python 的memory leak perf, 这篇文章 https://chase-seibert.github.io/blog/2013/08/03/diagnosing-memory-leaks-python.html 里提到的 `objgraph`, `guppy` 还不错.

我用 `objgraph` 看到 perf 函数跑完后, 内存里驻留了很多 celery 的`AsyncResult` 对象, 用 3.1.25 跑是没有的. 

继续debug下去发现问题在这行: https://github.com/celery/celery/blob/4.2/celery/result.py#L102, 去掉 `self.on_ready = promise(self._on_fulfilled)` 后内存就能被回收了. 而这个promise 有个 `weak` 参数, 设置成 True 之后就会创建 `self._on_fulfilled` 的弱引用, AsyncResult 也能被回收, 那 weak 改成 True 就对了吗? 

并不是, 找到以前的一个 pr: https://github.com/celery/celery/pull/4131, 这段代码本来 weak 就是 True 的, 当时为什么要把弱引用去掉呢? 因为传入的 `self._on_fulfilled` 是一个对象的 bound method, `weakref.ref` 无法处理, 后续尝试获取引用的时候总会得到 None, 当时的 pr 就是为了解决 `self._on_fulfilled` 不会被执行的问题. 要对 bound method 做弱引用需要使用 [WeakMethod](https://docs.python.org/3/library/weakref.html#weakref.WeakMethod), 但这个只在 python3.4 才开始有.

那内存泄露哪里来的呢，`AsyncResult` 里的确有循环引用 `AsyncResult -> self.on_ready -> promise -> self._on_fulfilled -> self`, python 的 内存管理同时使用 reference count 和 mark and sweep, reference count 碰到循环引用的对象时候无法回收对象, mark and sweep 可以, 可是 `AsyncResult` 这个 class 还定义了 `__del__` 方法, 这会让 python 的 gc 在处理循环引用的对象时不知道该以什么顺序去运行他们的 `__del__` 方法, 这些对象就会一直驻留在内存里: [gc.garbage](https://docs.python.org/2/library/gc.html#gc.garbage)

我提了一个临时的 pr: https://github.com/celery/celery/pull/4839  只是删除了 `__del__` 方法, 这样 python 的 gc 就能正确处理循环引用对象了. 官方打算把 WeakMethod backport 到 celery 用到的异步库 vine 里去: https://github.com/celery/vine/issues/21 这样在 celery 那头设置 `weak=True` 就能正确处理了.
