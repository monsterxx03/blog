---
title: Debug python performance issue with pyflame
author: will
type: post
date: 2017-06-05T09:50:44+00:00
url: /2017/06/05/debug-python-performance-issue-with-pyflame/
categories:
  - tech
tags:
  - python

---
pyflame is an opensource tool developed by uber: https://github.com/uber/pyflame

It can take snapshots of running python process, combined with [flamegraph.pl][1], can output flamegraph picture of python call stacks. Help analyze bottleneck of python program, needn&#8217;t inject any perf code into your application, and overhead is very low.

## Basic Usage

sudo pyflame -s 10 -x -r 0.001 $pid | ./flamegraph.pl > perf.svg

  * -s, how many seconds to run
  * -r, sample rate (seconds)

Your output will be something like following:

![][2]

Longer bar means more sample points located in it, which means this part code is slower so it has a higher chance seen by pyflame.

In my case, the output graph has a long IDLE part. Pyflame can detect call stacks who are holding GIL, so if the running code doesn&#8217;t hold GIL, pyflame don&#8217;t know what it&#8217;s doing, it will label them as IDLE. Following cases will be considered as IDLE:

  * Your process is sleeping, do nothing.
  * Waiting for IO.(eg: Your application is calling a very slow RPC server)
  * Call libs written in C.

The right part is real application logic code. You can check this part to get a sense of overview performance of your code.

Also you can exclude the IDLE part from graph if you don&#8217;t care about them, just apply `-x`

## Work with uwsgi

When perfing uwsgi worker process, pyflame may fail to detect the running python version, a workaround is apply `-p2` or `-p3` option to tell it.

 [1]: https://raw.githubusercontent.com/brendangregg/FlameGraph/master/flamegraph.pl
 [2]: https://blog.monsterxx03.com/wp-content/uploads/2017/06/Screen-Shot-2017-06-05-at-5.18.23-PM-300x160.png