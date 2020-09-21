---
layout: post
cid: 29
title: python - 一个asyncio的例子
slug: python-asyncio
date: 2017/12/22 13:49:00
updated: 2018/01/03 17:23:39
status: publish
author: alex
categories: 
  - 默认分类
  - 技术
tags: 
  - python
---


asyncio已经在python3的标准库中，是未来的趋势。
[官方文档][1]

直接上代码:
```
import asyncio
import time
 
now = lambda: time.time()
 
#模拟异步任务
async def do_some_work(x):
    print('Waiting: ', x)
    await asyncio.sleep(x)
    return 'Done after {}s'.format(x)
 
start = now()
 
coroutine1 = do_some_work(4)
coroutine2 = do_some_work(4)
coroutine3 = do_some_work(3)

#起三个任务
tasks = [
    asyncio.ensure_future(coroutine1),
    asyncio.ensure_future(coroutine2),
    asyncio.ensure_future(coroutine3)
]
 
loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait(tasks))
 
for task in tasks:
    print('Task ret: ', task.result())
 
print('TIME: ', now() - start)
```
输出:
```
Waiting:  4
Waiting:  4
Waiting:  3
Task ret:  Done after 4s
Task ret:  Done after 4s
Task ret:  Done after 3s
TIME:  4.003447771072388
```

三个任务，每个任务分别需要4秒，4秒，3秒，当任务遇到**await**关键字时，就会挂起，然后执行其他任务，总的消耗时间大致上会接近4秒(取决于最长消耗时间的任务)，实际上会额外消耗一点时间用于任务切换，但是所有任务均在一个线程内，这个开销基本上可以忽略不计。


  [1]: https://docs.python.org/3/library/asyncio.html