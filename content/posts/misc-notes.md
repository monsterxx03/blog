---
title: "misc notes"
date: 2019-01-24T10:58:10+08:00
categories:
    - tech
tags:
    - python
draft: true
---

#  Refactor

年前在对公司的代码做一次大的重构, 主要目的是把 repo 拆开, 以前虽然每个项目一个 repo, 但是因为很早期的原因， 所有 repo 之间都有很混乱的 import 关系.这次为了拆开 import 关系，　必须对所有代码做一次大重构, 过程中记录一些比较差的实现, 和怎么修改的.

## 滥用 *args, **kwargs

在内部代码中经常能在一些接口函数中看到 `*args, **kwargs`, 这样子的接口暴露给别人用是极其没意义的, 某个函数我一路往下看了好几层代码，到最底层的
db 处理处才弄明白要往里面传什么参数.

什么时候用 `*args, **kwargs`, 当该函数只作为一个 delegate 层时可以用, 此时该函数不应该尝试获取或修改 `*args, **kwargs` 的内容, 而仅仅将参数往下传递,
最常用的自然是在 `decorator` 中传参了.


    {{<highlight python>}}
    def demo_dec(func):
        def wrapper(*args, **kwargs):
            return func(*args, **kwargs)
        return wrapper
    {{</highlight>}}

接口函数务必把参数名定义清楚.

## keyword 参数不写参数名

还有一种情况是函数定义了可选参数, 但调用方传参时候不写参数名.

    {{<highlight python>}}
    def test(a, b=None):
        pass

    test(1, 2)
    {{</highlight>}}

我一直就觉得 python 里这个设计很失败, 给需要修改原始函数的人带来极大麻烦, test 函数如果添加一个可选参数在 a 和 b 之间, 调用方参数就传错了.

还好 python3　里引入了 keyword only arguments, 可以把函数定义成:

    {{<highlight python>}}
    def test(a, *, b=None):
        pass

    test(1, b=2)
    {{</highlight>}}

调用方必须显示写参数名，否则直接报错.

# Reading notes of Site Reliability Workbook

## Simplicity

- training time
- explanation time
- Administrative diversity
- Diversity of deployed configurations
- age


 Don’t compare the expected result to your current system. Instead, compare the expected result to what your current system would look like if you invested the same effort in improving it.

 ## Postmortem

Bad postmortem

 - missing context: should explain internal terminology, since postmortem audience may have different background.
 - key details omitted:
    - problem summary should have numbers about impact(revenue, number of users), If you don't know how to measure it, then you can't know it's fixed.
    - root causes and trigger
    - recovery efforts

A postmortem is a factual artifact that should be free from personal judgments and subjective language.