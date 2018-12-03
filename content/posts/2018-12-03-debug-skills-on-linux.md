---
title: "Debug Skills on Linux"
date: 2018-12-03T22:47:51+08:00
draft: true
tags:
    - linux
categories:
    - tech
---

Server resources can be classified in three main categories: compute, storage, network. When we saying IAAS(infrastructure as a service), usually it means virtualization of of those resources, provide on demand resource for us.

I use cloud provider (AWS) for daily work, it helps me avoid troubleshooting a lot of issues related to hardware. Normal skills to debug on linux server are still needed

This post shows a collection of debug skills used on linux, examples are shown on ubuntu 18.04. Won't explain every command in details, just write down those I used over and over again.

## Server load

People ofen tell me: xxx service is slow, can you have a look? xxx server is down, can you have a look?

`Slow` usually means server response time is long,  