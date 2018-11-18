---
title: "用 Bloom filter 给推荐列表去重"
date: 2018-11-17T13:58:27+08:00
categories:
    - tech
tags:
    - algorithm
    - bloomfilter
draft: true
---


之前产品里有一个功能是每天给用户推荐一批文章,要保证最后推给用户的文章每天不重复. 原先的实现很直接, 每次推送时候记录下用户 id 和 topic id 的键值对, 拿到新 topic 列表后,取出曾经给该用户推送过的文章列表, 两个 set 去重. 

这个实现的问题很明显, 存储空间量太大(M * N), user id (int64) + topic id (int64) = 16 bytes, 1 million 的用户, 每天给用户推送10篇文章, 一年要存储: 16 * 10 * 365 * 1M = 54.4GB. 查询效率也很低,要么一次取所有已读 topic id, 要么把要推送的 topic id 都丢进数据库去重.

Bloom filter 比较合适解决大集合去重的问题, 给定一个 key, bloom filter 返回不存在,则该 key 在集合中**一定**不存在, bloom filter 返回存在, 该元素仍旧**可能**存在集合中.

可以看出当元素存在时, bloom filter 是有一定误判率的, 所以说 bloom filter 是一种 probabilistic 数据结构, 存在 false positive 的情况, 但当元素不存在时结果正确, 很适合上面那个去重的例子, 只要 error rate 足够低, 就会有小概率漏推, 但不会重推.

关于 bloom filter 的实现原理网上有很多图文并茂的解说, 这里只做个简述:

bloom filter 支持两个基本的操作 add(添加元素), test(测试元素是否在集合中).预先划定一个 bit arrary, 大小为 m.

- add:选择 k 个独立的 hash 算法, 依次计算, 将结果映射到 bit array 的 m 个 bit 上, 将对应的 bit 置 1.
- test: 将 key 和 add 时一样用 k 个 hash 依次计算, 如果结果在对应的 bit 上有一个为0, 则元素一定不存在集合中, 如果所有对应 bit 上都是 1, 则元素可能存在集合中(因为这些 bit 可能是其他元素写上去的, 这就是误判率的由来).

给段代码模拟下 bloom filter 的实现:


    class BloomFilter(object):

        def __init__(self, size, k):
            self.array = bitarray(size, endian='little')
            self.array.setall(False)
            self.k = k

        def add(self, key):
            for i in range(self.k):
                self.array[_hash("{}_{}".format(key, i))] = True

        def test(self, key):
            for i in range(self.k):
                if not self.array[_hash("{}_{}".format(key, i))]:
                    return False
            return True

    if __name__ == '__main__':
        bf = BloomFilter(10000, 3)
        bf.add('a')
        print(bf.test('a'))
        print(bf.test('b'))

上文说 bloom filter 需要 k 个独立的 hash 算法, 这里只用了一个 _hash, 但在每次循环中把 index 放进去做了个 salt, 好的 hash 算法只要原始数据有一个 bit 的变化,生成的 digest 都会有雪崩效应, 所以用一个 hash 实现就够. hash 算法的选择要求数据均匀分布, 运算快, 没必要用 sha1, md5 这种比较重的 cryptographic hash, 选择 murmur, fnv 就够了. 

但这个最简实现看上去不太实用, size 为 m 的 bit array, 随着插入元素的变多, 空比特位会变少, error rate 明显会上升, 我比较想知道 m 个 bit array, 如果要将 error rate 控制在 1% 内,最多能容纳多少个元素, 此时的最佳 hash 次数 k 是多少? k 的次数过少会增加冲突, 增加 error rate, k 的次数太多又会导致 bit array 太快被沾满.

error rate 的推导过程可以看 [wiki](https://en.wikipedia.org/wiki/Bloom_filter#Probability_of_false_positives), 这里只记录几个数学结论公式, 辅助选择合适的参数构建 bloom filter.

假设 bit array 的大小为 m, 插入元素为 n, hash 次数为 k, error rate 为 p:

- m = -nlog<sub>2</sub>p / ln2
- k = log<sub>2</sub>(1/p)

也可以用这个在线版的 bloom filter 参数计算器: https://hur.st/bloomfilter

回到开始的推荐列表那个问题, 现在用 bloom filter 来存储每个用户的已读列表, 假设每个用户一天 10 条, 存储 1 年份, 10 * 365  / 8.0 / 1024 = 0.45 KB, 这是有效bits容量, 通过上面的计算器可以得出在保持 0.01 误判率的情况下, 一个用户需要总 bits 容量是 4.27KB . 1 million 的用户需要 4GB, 只需要原先实现的 7% 空间, 而且在数据库中可以只用一行来记录一个用户的数据.

这里还有问题, bloom filter 的 bit arrary 大小需要开始就划定, 而且不能更改, 如果一开始规划存 100 年的量, 一个用户就要在一开始预留 427KB 空间, 1 million 用户开始就要预留 407 GB, 一点都不省空间. 于是有人提出了 [scalable bloom filter](http://gsd.di.uminho.pt/members/cbm/ps/dbloom.pdf), 可以渐进扩展 bloom filter 的总大小,并且把全局最大 error rate 控制在一个预设大小内, 和普通的 bloom filter 比, 缺点是空间利用率会比较低.

https://github.com/jaybaird/python-bloomfilter  里提供了普通 bloom filter 和 scalable bloom filter 的实现, 打算基于它改写一版, 把数据存到 dynamodb 里去.