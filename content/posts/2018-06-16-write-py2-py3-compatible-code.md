---
title: "编写 python 2/3 兼容代码"
date: 2018-06-16T14:38:26+08:00
categories:
    - tech
tags:
    - python
---


[上一篇](https://blog.monsterxx03.com/2018/06/07/from-python2-to-python3/) 里简单得提了一点开始做 python 2 到 python3 迁移时候碰到的问题, 和工具的选择(推荐用 six).这篇讲下编写 python 2 / 3 兼容代码要注意的事情.

## \__future__

python2 里自带的向后兼容模块，将 python3 的一些语法行为 backport 到 python2 里, 使用的时候需要在文件头部声明, 作用域只在当前文件.

首先是几个在 python 2.7 里不用特意写，已经默认开启的特性:

- `from __future__ import nested_scopes` 2.2 开始就默认开启了，用于修改嵌套函数内的变量搜索作用域, 在此之前, 全局模块的优先级比被嵌套函数的父函数要高, 现在都没这个问题了.
- `from __future__ import generators`, yield 关键词, 2.3 默认支持.
- `from __future__ import with_statement`, with 关键词, 2.6 默认支持.

我显示开启的两个特性:

- `from __future__ import print_function`, 就是将 `print` 关键词变成函数啦, 导入后支持 python3 中 print 的完整参数，再用不带括号的 print 就会在文件被导入的时候报语法错误啦.
- `from __future__ import absolute_import`, python2 导入模块的时候默认是相对路径优先, 比如:在同级目录下有个 `email.py`, 你在代码中写 `import email` 的时候就会优先导入同目录的文件, 而不会去找标准库的 email 模块. 开启这个之后, 就会从全局的 PYTHONPATH 里找, 相对导入只能用 `from . import xx` 这种语法.

还有两个比较有用但因为影响比较大我没用的功能:

- `from __future__ import division`, py2 中除法是向下取整的 floor division, 开启后就像 py3 一样 `1/2` 就返回浮点数了. 因为代码量比较大， 我并没有在每个文件头部加上这句, 实际使用除法的时候, 需要 floor division 就用 `//`, 否则和以前一样 `1 * 1.0/2`, 保证代码正确.
- `from __future__ import unicode_literals`, 开启后默认所有的字符串定义都会变成 unicode, 不过 `str` 还是 byte string 啦, 这个并没法解决 unicode 的问题, 所以也没用.

## list comprehension 中变量作用域的变化

``` python
    i=0
    [i for i in range(10)]
    print(i)
```

这段代码在 py2 中会打印9, 在 py3 中打印0, list comprehension 中的变量不再污染外部作用域.

不过普通 for 循环中的变量作用域并没什么变化:

``` python
    i =0
    for i in range(10):
        pass
    print(i)
```

py2 / 3 中都打印 9, 原因是 python 中并没有 block 的概念.


list comprehension 如果作用在 class define 中行为也不太一样, 下面代码在 py2 里是队的， 但在 py3 里就会报错:

```python
    class A(object):
        name = 'test'
        d = {'a': '1', 'b': '2'}
        [d.update({_k: name}) for _k in d]

    >>> NameError: name 'd' is not defined
```

解决方法1, 在 class 的定义之外做更改操作:

``` python
    class A(object):
        name = 'test'
        d = {'a': '1', 'b': '2'}

    [A.d.update({_k: A.name}) for _k in A.d]
```

方法2, 用一个匿名lambda 函数将 class variable 传进去:

```python
    class A(object):
        name = 'test'
        (lambda name=name, d=d: [d.update({_k: name}) for _k in d])()
```

方法2 看上去太 trick, 我用 1.

## 长整型

py2 中如果 int 数值大于 `sys.maxint` (2**63-1), 会被隐式得转换成 long, 定义数值变量的时候可以在最后加上 `l` 或 `L` 声明这是个 long, py3 中去把 int 和long 合并了, 只用 int 就行, `long` 不存在了, 使用会报错, `l` 和 `L` 也不能再加了,对代码的影响主要是类型判断.

`isinstance(v, (int, long))`, 需要改成 `isinstance(v, six.integer_types)`, 

`isinstance(v, (int, long, float))` --> `isinstance(v,(six.integer_types, float))`

## reload 


不再是 builtin 函数, 用 `from imp import reload` 可以兼容 2,3.

## zip, map, filter

py2 里会返回 list, py3 会返回 iterable 对象, 这样在取 slice, 或多次遍历的时候就会出问题, 需要用 `list()` 手动转成 list, 我的建议是不使用这些函数, 完全用 list comprehension 改写, 就不会出问题了.

## reduce

也不再是 built 函数, 需要 `from functools import reduce`.

## xrange 

py3 中没有啦, 只能用 range, 而且返回结果也是 iterable, 用 `six.moves import range`, 这个在 py2 下就是 `xrange`.

## dict

`iterkeys(), itervalues(), iteritems()` 在 py3 中也没有啦, `keys(), values(), items()` 会返回 iterable, 要 
list也要自己转换, 可以用 `six.iteritems(d), six.iterkeys(d), six.itervalues(d)`.

## 异常处理

一些特别老的语法不再支持, 下面的写法在 py3 下都会报错:

    try:
        1/0
    except Exception, e:
        pass

    raise ValueError, "yahaha"

用正常的写法就行:

    try:
        1/0
    except Exception as e:
        pass

    raise ValueError("yahaha")

## StringIO 

StringIO (cStringIO) 在 py3 里都没了, 用 `io.BytesIO` 和 `io.StringIO`, 注意 `BytesIO` 只接受bytes, `StringIO` 只接受 unicode.

## 其他标准库的变化

只列一些用的比较多的,不常用的就不列了, `six` 代码里看看就知道了.

`ConfigParser` 变成了 `configparser`, 在 py2 下有 backport的版本,装一个就行.

`from HTMLParser import HTMLParser` --> `from html.parser import HTMLParser`, 如果你装过 `future` 的话,后一种导入在 py2 里是能用的, future 在安装时候会安装一个假的 html 包, 里面判断了python 版本决定真正的导入方式.

`Queue` --> `queue`, 装过 `future`, 后一种也能用.

`urlparse` --> `from six.moves.urllib import parse as urlparse`,  future 有个 backport的版本, 但需要调用它的 `install_alias` 来修改全局 sys.modules, 这个有坑,前篇讲过,所以我用 six.

一些 urllib 里的 url encode 函数也移到了 `urllib.parse` 里去, 所以需要 `from six.moves.urllib.parse import quote, unquote, unquote_plus, quote_plus, urlencode`.

urllib2 也没有了,变成了 `urllib.request`, 需要用 six: `from six.moves.urllib import request`.

## unicode 和 str

这个坑是最大的, py2 里 `str` 表示 byte string, 实际上是是带有些字符串行为的 binary data, 它和 `unicode` 的边界非常模糊,为了使用方便做了很多 hack, 所以 py2 里 `str` 和 `unicode` 经常混用, 而 `bytes` 和 `str` 完全等价,只是个 alias. py3 里 `str` 默认就是 unicode, `bytes` 就是二进制字节, `unicode` 关键词没了, 而且 `str` 和 `bytes` 分界明显,不能混在一起.

### in py2

你明白下面的语句,哪些会失败,哪些成功, 失败是报 `UnicodeDecodeError` 还是 `UnicodeEncodeError`, 成功的话返回值是 `str` 还是 `unicode` 呢? 准确说出来你就明白啦.

``` python

'啊啊' + 'yaha'

u'啊' + 'yaha'

u'啊' + u'yaha'

'.'.join(['啊',  'yaha'])

'.'.join([u'啊', '啊'])

u'.'.join(['啊', 'abc'])

u'.'.join([u'啊', 'abc'])


u'test {}'.format('啊')

'test {}'.format(u'啊')

'test {}'.format('啊')

u'test {}'.format(u'啊')

'test %s' % '啊'

u'test %s' % '啊'

'test %s' % u'啊'

'黑test %s' % u'啊'
```

我把规则总结下, `%s` 格式化, 字符串内置的 `join, replace`,其实行为和 `+` 是一样的:

- 所有部分都是 `str`, 结果就是 `str`, 不会发生 decode, encode
- 所有部分都是 `unicode`, 结果就是 `unicode`, 不会发生 decode, encode
- `str` 和 `unicode` 混合, 会将所有 `str` 用 ascii 解码,  所以可能会报 `UnicodeDecodeError`, 结果是 unicode.

string format 比较特别:

- format的结果和 format string 的类型保持相同
- 如果 format string 是 `str`, 会将所有 format 参数中的 `unicode` 用 ascii 编码, 可能会有 `UnicodeEncodeError`, 如果 format string, 会将所有 format 参数中的 `str` 用 ascii 解码,可能报 `UnicodeDecodeError`.

### in py3

`bytes` 和  `str` 无法相加, `bytes` 也无法作为 format string, 也不能用 `str` 将 bytes 换为字符(会的到 `"b'a'"` 之类的结果), 原因是 str 会调用传入参数的 `__str__` 函数,并不会去解码, 你需要指定编码: `str(b'abc', encoding='utf-8')`  或 `b'abc'.decode('utf-8')`

### 兼容 py2/3

简单讲, 如果你代码中出现 `unicode` 关键词, 在py3一定是错的, 因为 `unicode` 关键词被移除了, 出现 `str` 大概率在 py3 下是错的, 因为语义变成了 unicode.

永远不要使用 `str(v)`, 需要字符串,用 `six.text_type(v)`.

不要用 `isinstance(v, str)` 做类型检查, 检查bytes, 用 `isinstance(v, bytes)`, 检查text, 用 `isinstance(v, six.text_type)`.

basestring 也没了, 用 `isinstance(v, six.string_types)`, 注意 py3 下这个就是 `str`, 所以正确与否要看上下文语义.

在函数之间传递的时候, 统一使用 unicode 类型, 只在程序的边界 encode 成 bytes,比如写入文件, 通过网络发送, base64 编码, 丢给散列函数等.