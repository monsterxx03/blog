---
title: "用 google spreadsheet 计算房贷等额本息还款"
date: 2018-02-25T13:59:11+08:00
draft: true
categories:
  - life
---

看房看的眼花缭乱, 网上有很多房贷计算器, 但都没什么方便的比价功能, 能直观得看到首付和月供就好啦, 但也不想注册，不想老收中介的广告,索性用 google spreadsheet 自己做个简单的吧.

## PMT 公式

可以用 PMT 公式来计算等额本息的月供金额: https://support.google.com/docs/answer/3093185?hl=zh-Hans


    PMT(rate, number_of_periods, present_value, [future_value, end_or_beginning])


rate 是每期的利率, 如果商贷基准利率是 0.049, 每月还款, rate 就是 0.049/12.

number_of_periods 是贷款期数, 年数 * 12.

present_value 是贷款数额.

PMT 公式算出的值是负数，需要取反.

实际例子:

    商贷基准利率=0.049
    利率倍数=1
    首付百分比=0.3
    贷款年数=30
    公积金贷款利率=0.0325
    公积金贷款数额=300000
    房产单价=14000
    建筑面积=85

房屋总价=14000 * 85=1190000
首付= 1190000 * 0.3 = 357000
商贷=1190000-357000-300000=533000
 
月供= -PMT(0.049/12 * 1, 30*12, 533000) - PMT(0.0325/12, 30 * 12, 300000) =4134

## 抓取实时房价

有了 PMT 公式，就能很容易的把各个楼盘的月供情况列在一起了，有没有办法让 spreadsheet 显示楼盘的最新单价呢?

可以用 spreadsheet 的 IMPORTXML 函数从网页中用xpath提取元素.

    IMPORTXML(url, xpath)

价格从房天下抓取, 例子:

    IMPORTXML("http://yunheyihaofurc.fang.com/", "/html/body/div[3]/div[3]/div[2]/div[1]/div[2]/div[1]/span")

文档里没有说 IMPORTXML 的刷新频率, 但我隔天打开文档的时候它会自动刷新价格，也并不需要精准的价格提醒，就这样吧.