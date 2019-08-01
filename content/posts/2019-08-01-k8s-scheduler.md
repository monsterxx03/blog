---
title: "k8s Scheduler internal"
date: 2019-08-01T16:30:10+08:00
draft: true
categories:
- tech
tags:
- k8s
---

k8s scheduler internal:

get unscheduled pod from internal `scheduling_queue`, and try to schedule the pod.

podInformer register event handler in `pkg/scheduler/eventhandlers.go`. When pod Add/Update/Delete, will update `scheduling_queue`

`scheduling_queue` implemented as `PriorityQueue`(`pkg/scheduler/internal/queue/scheduling_queue.go`)

activeQ is a heap(head has highest priority). if pod have priority defined, will sort by priority, else compare by the timestamp they
added into scheduling queue, earlier have higher priority(`pkg/scheduler/internal/scheduling_queue.go:activeQComp`).

podBackoffQ is a heap, ordered by backoff expiry (`podsCompareBackoffCompleted`)


TODO: chose host logic

After get a SuggestedHost, bind pod to host asynchronously.


