---
title: Concurrency in Go, Reading Notes
author: will
type: post
date: 2017-04-19T16:26:58+00:00
url: /2017/04/19/concurrency-in-go-reading-notes/
categories:
  - tech
tags:
  - book
  - golang

---
A few notes taken when reading <Concurrency in Go>

<!--more-->

## Deadlock

Coffman Conditions theory is used to help detect potential deadlock problem.

If your problem meet the following 4 conditions, it may come to deadlock.

  1. Mutual Exclusion: A concurrent process holds exclusive rights to a resource at any one time.
  2. Wait for condition: A concurrent process must simultaneously hold a resource and be waiting for an additional resource
  3. No Preemption: A resource held by a concurrent process can only be released by that process, so it fulfills this condition. 
  4. Circular Wait: A concurrent process must be waiting on a chain of other concurrent processes which are in turn waiting on it , so it fulfills this final condition too

If we can avoid any one of these conditions, we can avoid deadlock, but in real world, its&#8217;s usually very hard

## Livelock

Two processes try to avoid deadlock at same cadence, the result is no work is accomplished.

## Starvation

Starvation is any situation where a concurrent process cannot get all the resources it needs to perform work.

Eg: two goroutines share a same lock, If one goroutine acquires the lock for too long time, another goroutine will be waiting too long, like it&#8217;s starving 🙁

## Concurrency safety

When you handling existing code, consider following question to determine whether it&#8217;s concurrency safey:

  1. Who is responsible for concurrency ?
  2. How is the problem space mapped onto concurrency primitives? (how to divide the problem to make it run concurrently)
  3. Who is response for synchronization ?

## Concurrency and Parallelism

Concurrency is about code, parallelism is about the state running program.

We write concurrent code to wish it can run in parallelly. (but if the running machine only have one core, the program will not be parallelism)

## Lock or Channel

`sync` package provide mutexes as well, how to choose it between channel?

If trying to transfer onwership of data, use channel.

If trying to guard internal state of a struct, use mutex.

Channel is more composable than lock.

## Example

    func main() {
            var wg sync.WaitGroup
            for _, v := range []string{"a", "b", "c"} {
                    wg.Add(1)
                    go func() {
                            defer wg.Done()
                            fmt.Println(v)
                    }()
            }
            wg.Wait()
    }
    

If the loop is very faster, the program will exit before goroutines start, the ouput will be three `c`.

If there is heavy logic in loop, result will be nondeterministic.

The right way is pass `v` to the closure to make a copy, then every goroutine will hold the exact value passed to it.

## Misc

A goroutine consume a few kilobytes ram.

System thread context switch will cost about 1 ~ 2 us per round(send message between threads).

Goroutine context switch cost about 200 ns per round (send message between goroutines), about 92% faster than os thread.

Use `sync.Cond`, if you need to suspend a goroutine.