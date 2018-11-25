---
title: "Design"
date: 2018-11-25T13:35:01+08:00
draft: true
tags:
    - book
categories:
    - tech
---

最近读了一本书 `<a philosophy of software design>`,

书里总结了很多常见的坏设计范例和一些建议. 看完倒没有那种眼前一亮打开新世界的感觉, 有过几年编程经验的, 多多少少应该都碰到过书里的问题, 但作者用很棒的语言把这些 bad simle 总结了出来.

以往碰到同事写出类似代码的时候, 我直觉上觉得不好, 但又很难用简练的话说出哪里不对, 难以说服别人, 看了这本书算是好好得整理了一下思路. 结合自己以往的经验, 讨论下其中的一些内容.

## Complexity

作者全书的大观点是, 软件开发的各种技巧, 规范, 最终目标都是控制项目的复杂度 (complexity).

Complexity 会带来两个问题: 难以理解, 难以修改.

这个观点相当赞同, 做了一段时间 infra 后, 感觉系统设计大部分时间其实都在做 tradeoff. 我做 tradeoff 的目的是什么? 控制 complexity!写代码也是一个理.

复杂度怎么来的? 作者观点是来自 dependency 和 vagueness 的积累.

## Red flags

## Doc and comment