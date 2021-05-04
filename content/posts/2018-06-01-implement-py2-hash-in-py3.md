---
title: "在python3.7 中实现python2.7 的内置 hash 函数"
date: 2018-06-01T17:03:24+08:00
categories:
    - tech
tags:
    - python
    - code-infra
---

最近着手准备从 python2.7 迁移到 python3.7, 还没开始就碰到一个问题. 老系统里有一部分竟然是将 python 内置 hash 函数的结果存进了数据库, 这个做法绝对是错的,
hash 的结果本来就没有保证过在各个版本的 python 中保证一致. 而且 python3 中算法完全变了, 默认在进程初始化的时候会用随机种子加进 hash 过程, 所以python 进程
一重启结果就不一样了. 木已成舟， 目前看将数据库里的值全部改掉是不可能了, 只能在 python3 中重新实现一下这个算法.

python2.7 中的hash 算法是 fnv (有修改), python3 中变成了 sip, [pep-456](https://www.python.org/dev/peps/pep-0456).

fnv 的实现很简单， 引用 wikipedia 上的伪代码:

    hash = FNV_offset_basis
    for each byte_of_data to be hashed
            hash = hash × FNV_prime
            hash = hash XOR byte_of_data
    return hash

FNV_prime 是 1000003, python 实现和标准 fnv 的不同在于, 在进入循环 hash 之前将数据左移了7位, 在最后又和长度做了次XOR, 生成的数据随机性更大一点. 

fnv 的代码在 python3 中还是有的 https://github.com/python/cpython/blob/3.7/Python/pyhash.c#L242,
自己编译 python， 可以通过选项控制使用 fnv 算法, 虽然没打算这么实施, 姑且试了一下, `./configure --with-hash-algorithm=fnv`, 选择 fnv hash. 并且要在启动 python 的时候禁掉随机种子: `PYTHONHASHSEED=0  python3.7`. 注意, 这里只是测试，强烈不建议这么做, fnv 面对 hash 碰撞攻击会挂的很惨.

但试过之后, 只有对少于 8字节的 byte string  的 hash 结果是一致的, 超过8字节 或 unicode 完全不一样, 原因是在 python3 里的 unicode 实现和 python2 里的也是不一样的(python3.3 和 python3.2 又是不一样的 https://www.python.org/dev/peps/pep-0393/).
主要看下 python2.7 和 3.7 对 unicode hash 的实现有什么不一样. 

python2.7 中的unicode: https://github.com/python/cpython/blob/2.7/Include/unicodeobject.h#L415

    typedef struct {
        PyObject_HEAD
        Py_ssize_t length;          /* Length of raw Unicode data in buffer */
        Py_UNICODE *str;            /* Raw Unicode buffer */
        long hash;                  /* Hash value; -1 if not set */
        PyObject *defenc;           /* (Default) Encoded version as Python
                                    string, or NULL; this is used for
                                    implementing the buffer protocol */
    } PyUnicodeObject;

Py_UNICODE 其实就是 typedef wchar_t.

从 python 3.3 开始 unicode 是一种变长实现: https://github.com/python/cpython/blob/3.7/Include/unicodeobject.h#L348 

    typedef struct {
        PyCompactUnicodeObject _base;
        union {
            void *any;
            Py_UCS1 *latin1;
            Py_UCS2 *ucs2;
            Py_UCS4 *ucs4;
        } data;                     /* Canonical, smallest-form Unicode buffer */
    } PyUnicodeObject;

unicode 的存储取决于字符中处于 unicode 表中最高位的 code point, 是ASCII 或 latin1 字符(U+0000-U+00FF) 每个 code point 会占 1 字节, BMP 字符(U+0000-U+FFFF) 用两字节, 剩下的 (U+10000-U+10FFFF) 用 4 字节.


python2.7 中的 unicode hash: https://github.com/python/cpython/blob/2.7/Objects/unicodeobject.c#L6637, 每次取一个 Py_UNICODE 作为data block 进入hash 过程.

python3.7 中的 hash https://github.com/python/cpython/blob/3.7/Python/pyhash.c#L242, 更复杂一点, data block 8 字节, 至少取一个 block 来做单字节 hash, 剩下的一个 block 一个 block 处理, 所以少于 8 字节的 byte string 结果和 python2.7 是一样的. 


知道了差异, 写了个 c extension, 把 python2.7 的 hash 在 python3 上重新实现了一下: https://github.com/monsterxx03/legacyhash , 只能在 python3 上编译, 只支持 byte string 和 unicode.
