---
title: "用 AWS Personalize 做推荐系统"
date: 2020-02-18T14:38:56+08:00
draft: true
categories:
- tech
tags:
- aws
---

这几天测试了下 aws 的 personalize service, 看看能不能替换掉产品里现有的一些推荐逻辑.

大致的流程:

- 导入数据
- 选择 recipe 进行 training, 得到一个 solution version
- 选择 solution version 创建 compaign
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
        "name": "Users",
        "namespace": "com.amazonaws.personalize.schema",
        "fields": [
            {
                "name": "user_id",
                "type": "string"
            },
            {
                "name": "birthday",
                "type": "int"
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
            },
            {
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
- HRNN, 基于循环神经网络, 只使用 User-Item interaction, 特点是可以根据时序给用户的历史行为设定权重, 每个用户得到的推荐都是不一样的, input 是 `user_id`, output 是 list of `item_id`. RNN 作用于用户推荐的缺点是无法记录太长的用户行为, HRNN 把用户的历史行为划分成不同的 session, 假定用户在同一个 session 内兴趣都是一样的, 比如现在我在找战争电影, 那我接下来一段时间的浏览行为都和战争电影有关系, 如果我超过半小时没有活动, 就进入一个新的 session.  
- HRNN-metadata, 基于 HRNN, metadata 指的是 User 和 Item dataset, 如果这两者有比较好的可用于分类的数据, 比如 age, location, gender 等, 加入训练可能得到比 HRNN 更好的数据.
- HRNN-coldstart, 基于 HRNN-metadata, 使用场景是你要经常加入新的 item, 一般新的 item 都有比较少的用户交互(这样的 item 被认为是 coldstart), 而你又希望只给用户推荐新的东西, 而不是那些老的很热门的东西, 这个 recipe 能 filer 出一个数据子集来进行 train, 比如只推荐５天内用户交互行为少于 10 次的 item.
- Personalized-Ranking 基于 HRNN, 只使用 User-Item interaction, 给一个 `user_id` 和 一个列表的 `item_id`, 根据用户的历史交互情况对 item 进行排序后返回. 使用场景如先用 ElasticSearch 根据用户输入的 keyword 找出一列 item, 再由 personalize 根据用户的历史
行为对结果进行排序.

### 评估 solution 的效果

## 创建 compaign

## Get recommendations 
