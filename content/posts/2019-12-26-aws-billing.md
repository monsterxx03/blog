---
title: "聊聊 AWS 的计费模式"
date: 2019-12-26T12:08:47+08:00
draft: true
categories:
  - tech
tags:
  - aws
---

网上经常有人诟病 AWS 的计费模式复杂, 喜欢国内那种打包式的售卖方式, 这个可能受限于每个公司的财务流程, 预算制定方式, 合不合国情,本文不讨论.
仅从开发者的角度介绍下 AWS 部分 service 的计费方式, 以及为什么我觉得它是 developer friendly 的.

云计算本质上就是资源池化, 按需取用, 没啥黑科技, 本文也不讨论云计算的意义和相比自建否能省钱等问题, 近 10 年无数人讨论过了.

PS: 那些为了蹭一年 free plan 然后抱怨什么偷跑流量, 偷偷扣费的大哥就省省吧, AWS 根本不是给个人用的, 老老实实用 lightsail 得了. 

## EC2

一台 EC2 instance 的价格组成:

- instance fee, 实际支付的是 CPU+RAM 的费用
- EBS fee, server 的根分区都是 EBS volume, 按 EBS 计费(31GB 的 volume 使用 1 小时, 和 1GB 的volume 使用 744 小时价格相同).
- data transfer fee, 这部分组成比较复杂，在 network 部分单独讲.  
- EIP fee, 每台 instance 挂一个 EIP 是免费的(eip 使用状态不收费, 闲置收费), 超过一个 eip, 每个按小时再收费.

instance fee 又分 on-demand/resersed/spot instance.

- On demand: 比如 linux, us-east-1, m5.large(2 vCPU, 8GiB memory) 0.096$/h, linux 实例不足一小时按秒付费(最少1min), 其他需要
license 的系统(rhel, sles, windows...) 按小时收费. 
- Spot instance: 使用 aws 的闲置资源,价格比 on demand 便宜很多, 最少能到1/10, m5.large 等常用实例价格基本为 on-demand 的 1/3,
但 spot instance 随时会被关闭(aws 需要资源的时候), 会提前 2min 通过 cloudwatch 发送一个通知让你做清理工作, 比较适合跑 hadpoo 等
task 可以被重调度的工作, 或作为 k8s 的 worker node 跑 stateless 的服务.


dedicated instance:
