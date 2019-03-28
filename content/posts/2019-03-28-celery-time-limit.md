---
title: "Celery Time Limit 的坑"
date: 2019-03-28T17:37:29+08:00
categories:
    - tech
tags:
    - python
    - celery
---

之前用 celery 做的 task 都是一些很简单轻量级的 task, 从来没触发过 timeout, 最近加入了一些复杂很耗时的 task, 碰到一些 time limit 的坑.


celery 中 time limit 有两种, soft_time_limit 和 hard_time_limit, 区别是 soft_time_limit 会在内部抛一个 Exception, task 可以 catch 自行处理.
hard time limit 没法被 catch.

使用如下:

{{<highlight python>}}
from myapp import app
from celery.exceptions import SoftTimeLimitExceeded

@app.task
def mytask():
    try:
        do_work()
    except SoftTimeLimitExceeded:
        clean_up_in_a_hurry()
{{</highlight>}}


我 celery pool 用的是 gevent, 实际上在现在的实现里 gevent 做 worker pool 的时候会忽略 soft_time_limit, 只有 hard_time_limit 会被触发(通过 gevent.Timeout 实现).

坑爹的是文档里写的是错的: http://docs.celeryproject.org/en/latest/userguide/workers.html#time-limit

soft_time_limit 只在 prefork pool 里支持.

我现在想让 celery 把这个 hard timeout 的情况 report 到 sentry, 看了圈代码并没法从外面 override timeout 的 callback. 只能很丑得做了个 monkey patch, 在初始化 celeryapp
的代码里:

{{<highlight python>}}
from gevent import Timeout
from celery.concurrency.base import apply_target as celery_apply_target
from celery.concurrency import gevent as  celery_gevent_pool


_original_apply_timeout = celery_gevent_pool.apply_timeout


# monkey patch original timeout handler, to report error to sentry
# when using gevent pool, celery will ignore soft_time_limit,
# only hard_time_limit works.
def apply_timeout_with_report(target, args=(), kwargs={}, callback=None,
                  accept_callback=None, pid=None, timeout=None,
                  timeout_callback=None, Timeout=Timeout,
                  apply_target=celery_apply_target, **rest):
    try:
        with Timeout(timeout):
            return apply_target(target, args, kwargs, callback,
                                accept_callback, pid, propagate=(Timeout,), **rest)
    except Timeout as e:
        # report to sentry


celery_gevent_pool.apply_timeout = apply_timeout_with_report
{{</highlight>}}

其实稍微修改下 https://github.com/celery/celery/blob/v4.3.0rc3/celery/concurrency/gevent.py#L21  这个函数就能让celery 支持 soft time limit:


{{<highlight python>}}
def apply_timeout(target, args=(), kwargs={}, callback=None,
                  accept_callback=None, pid=None, timeout=None,
                  soft_timeout=None, timeout_callback=None,
                  Timeout=Timeout, apply_target=base.apply_target, **rest):
    hard_timeout = Timeout(timeout)
    soft_timeout = Timeout(soft_timeout, SoftTimeLimitExceeded)
    hard_timeout.start()
    soft_timeout.start()
    try:
        return apply_target(target, args, kwargs, callback,
                            accept_callback, pid,
                            propagate=(Timeout,), **rest)
    except Timeout as t:
        if isinstance(t, hard_timeout):
            # only catch gevent.Timeout set by us
            return timeout_callback(False, timeout)
        else:
            raise
    finally:
        hard_timeout.close()
        soft_timeout.close()
{{</highlight>}}


但这块单元测试很难写啊...先凑合着用了.
