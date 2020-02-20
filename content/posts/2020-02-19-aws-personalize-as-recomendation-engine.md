---
title: "用 AWS Personalize 做推荐系统"
date: 2020-02-18T14:38:56+08:00
draft: true
categories:
- tech
tags:
- aws
- personalize
---

这几天测试了下 aws 的 personalize service, 看看能不能替换掉产品里现有的一些推荐逻辑.

大致的流程:

- 导入数据
- 选择 recipe 进行 training, 得到一个 solution version
- 选择最佳 solution version 创建 compaign
- 调用 api, 根据 compaign 得到 recommendations

一些 iam 权限相关的设置就不写了, 具体看文档吧, 这里只记录下主要步骤.

## 导入数据

首先需要准备用来 training 的数据, 分成三种数据集:

- User
- Item
- User-Item interaction

其中 User-Item interaction 是必须的 dataset, 所有 recipe 都会用到, User 和 Item 被称作 metadata dataset, 只有个别 recipe 会用到.

每个 dataset 创建的时候需要建立一个 schema, 来描述各自的结构(avro 格式).

example User schema:

    {
        "type": "record",
        "name": "Users",与与
        "namespace": "com.amazonaws.personalize.schema",
        "fields": [
            {
                "name": "user_id",
                "type": "string"
            },
            {
                "name": "birthday",
                "type": "int与
            },
            {
                "name": "gender",
                "type": "string",
                "categorical": true
            },
            {
                "name": "location",
                "type": "string",
                "categorical": true
            }
        ],
        "version": "1.0"
    }

其中 `user_id` 是必填字段, 其他都是可选的自定义字段, 其他字段如果是 string, 必须加上`"categorical": true`, 表示它是用来分类的.

example  Item schema:
与
    {
        "type": "record",
        "name": "Items",
        "namespace": "com.amazonaws.personalize.schema",
        "fields": [
            {
                "name": "item_id",
                "type": "string"
            },
            {
                "name": "category",
                "type": "string",
                "categorical": true,
            },
            {
                "name": "bookmarks",
                "type": "int"
            }
        ],
        "version": "1.0"
    }

`item_id` 必填, 其他性质同 User dataset.

example User-Item interaction schema:

    {
        "type": "record",
        "name": "Interactions",
        "namespace": "com.amazonaws.personalize.schema",
        "fields": [
            {
                "name": "user_id",
                "type": "string"
            },
            {
                "name": "item_id",
                "type": "string"
            },与{
                "name": "event_type",
                "type": "string"
            },
            {
                "name": "event_value",
                "type": "float"
            },
            {
                "name": "timestamp",
                "type": "long"
            }
        ],
        "version": "1.0"
    }

`user_id`, `item_id`, `timestamp` 必填,  `event_type`, `event_value` 可选.

这张 schema 用来记录 user 和 item 的交互信息.

比如 item 是一系列帖子, 那我们就能用 `event_type` 来记录用户的 view, upvote, downvote, reply 等不同行为, `event_value` 可以给不同的 `event_type` 设置不同的权重.

training 的时候也能根据 `event_type` 做筛选, 只使用一种来 training. 或 `event_value` 高于某个值的数据进行 training.

之后在 dataset 页面里创建一个 import job, 就能从 s3 导入数据了(这里需要正确设置 iam role 和 s3 bucket policy). 大概需要十几分钟.

### 不方便的地方 

- dataset schema 没法更改, 只能创建一个新的 dataset 然后再建新的 dataset schema, 重新导入数据.
- 界面上没法删除 dataset schema, 只能用 aws cli 进行删除.
- schema 中定义的每一个字段,导入时都不能为空.

### 增量数据

User-Item interaction 数据可以通过创建 event tracker 来增量更新, 不需要重新 train.

User 和 Item 没办法增量更新，只能重新 import 全量数据, 这个有点傻，不过使用的 recipe 没用到它们就无所谓.


## 选择 recipe 进行 training

支持以下几种预定义的 recipe：

- popularity-count, 计算全局热度, 只使用 User-Item interaction, 逻辑和线上现有的排序算法类似, 不是个性化的推荐, 任何 `user_id` 得到的推荐结果都一样的, 一般可以作为一个 baseline 的结果, 和其他 recipe 做对比.
- sims(item-to-item similarities), 传统个性化推荐系统的做法, 基于协同过滤, 只使用 User-Item interaction. 使用的时候 input 是 `item_id`, 返回相似的 item, 如果 `item_id` 不存在或没有足够的交互数据，会返回全局最热的 item. 
- HRNN, 基于循环神经网络, 只使用 User-Item interaction, 特点是可以根据时序给用户的历史行为设定权重, 每个用户得到的推荐都是不一样的, input 是 `user_id`, output 是 list of `item_id`. RNN 用于用户推荐的缺点是无法记录太长的用户行为, HRNN 把用户的历史行为划分成不同的 session, 假定用户在同一个 session 内兴趣都是一样的, 比如现在我在找战争电影, 那我接下来一段时间的浏览行为都和战争电影有关系, 如果我超过半小时没有活动, 就进入一个新的 session.  
- HRNN-metadata, 基于 HRNN, metadata 指的是 User 和 Item dataset, 如果这两者有比较好的可用于分类的数据, 比如 age, location, gender 等, 加入训练可能得到比 HRNN 更好的数据.
- HRNN-coldstart, 基于 HRNN-metadata, 使用场景是你要经常加入新的 item, 一般新的 item 都有比较少的用户交互(这样的 item 被认为是 coldstart), 而你又希望给这些新的 item 预热(warm up), 而不是那些老的很热门的东西, 这个 recipe 能 filer 出一个数据子集来进行 train, 比如只推荐５天内用户交互行为少于 10 次的 item小时 和 一个列表的 `item_id`, 根据用户的历史交互情况对 item 进行排序后返回. 使用场景如先用 ElasticSearch 根据用户输入的 keyword 找出一列 item, 再由 personalize 根据用户的历史
行为对结果进行排序.

每种 recipe 的 training 时间不一样, 在我的测试过的 recipe 中耗时依次是 HRNN-coldstart > HRNN-metadata > sims > popularity-count. popularity-count training hour 大概1小时, HRNN-coldstart training hour 花了 39 小时.

### 评估 solution 的效果

personalize 把输入的　dataset, 随机划分成90% 的 training data 和 10% 的 testing data, solution 使用　training data 创建. 测试时候会使用 training data 中最老的 90% 作为输入, 将生成结果和 testing data 中最近的 10% 进行比较.

评价 solution 好坏有以下几个 metrics, 都是数字越大越好:

- coverage: 所有查询中返回的 items 个数在所有 items 中的占比
- mean_reciprocal_rank_at_25: 在 top 25 个推荐中, 第一个命中的推荐所在位置的倒数, 假如地命中了2,10,15 位, 这个值就是 1/2,　如果你对单个排名最高的推荐感兴趣, 可以看这个数字.
- normalized_discounted_cumulative_gain_at_K: 这个指标根据命中的推荐所在位置,每个给一个权重, 再相加. 基本假设是排名越靠前的推荐用户越感兴趣, 权重就越高. 权重系数是`1/log(1+position)`, 比如查询结果中第2,5位命中了，　计算 `1/log(1+2) + 1/log(1+5)`, 得到的值叫DCG(discounted cumulative gain), 而理想情况下最好当然是命中第1,2位了,理想值应该是 `1/log(1+1) +小时/log(1+2)`, 这个理想值叫做 ideal DCG. 我们需要的 NDCG = DCG/idea DCG, 即　`(1/log(1+2) + 1/log(1+5))/(1/log(1+1) + 1/log(1+2)) = 0.6241`
- precision_at_K: 前 K 个推荐中命中数目除以K.

白话一点, personlaize 的评估 metrics 评价标准是推荐结果中的命中推荐出现的越早越好, 命中数目越多越好.

## 创建 compaign

solution 就是 train 好的 model, 要使用它需要创建一个关联的 compaign, 才能在 api 中调用. 创建 compaign 时候指定需要的最小 tps, 如果实际 tps 超过这个值会自动 scale up. campaign 也可以理解成一个 api endpoint

## Get recommendations 

用 python 获取推荐的例子:

    import boto3


    client = boto3.client('personalize-runtime')

    response = client.get_recommendations(
        campaignArn='arn:aws:personalize:us-east-1:xxxxx:campaign/name',
        userId='111111',
    )

根据 user id 得到推荐的 recommendation, 适用于 popularity-count, hrnn, hrnn-metadata, hrnn-coldstart.


    response = client.get_recommendations(
        campaignArn='arn:aws:personalize:us-east-1:xxxxx:campaign/name',
        itemId='2222',
    )

有 item id 得到相似 item 推荐, 适用于 sims.

    response = client.get_personalized_ranking(
        campaignArn='string',
        inputList=[
            'aaaa',
            'bbbb',
            ...
        ],
        userId='11111',
    )

输入 user id, 和一列 item id, 根据用户的历史兴趣对 inputList 进行排序返回. 适用于 Personalized-Ranking.