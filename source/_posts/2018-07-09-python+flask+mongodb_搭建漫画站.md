---
layout: post
cid: 39
title: python+flask+mongodb 搭建漫画站
slug: manhua
date: 2018/07/09 11:26:00
updated: 2019/10/02 13:25:47
status: draft
author: alex
categories: 
  - 默认分类
  - 技术
tags: 
  - python
  - flask
  - mongodb
---


最近想玩玩mongodb，虽然SQL数据库还是占据主导地位，但NoSQL数据库还是有一定优势的，比如在QPS上NoSQL性能就远远大于SQL。

### [传送门][1] ###

![截图][8]

主体用了 [flask][2] + [mongodb][3]

部署用了 [nginx][4] + [gunicorn][5]

前端用了国人开发的 [mdui][6]，google material design设计的css库

mongodb效率还是蛮高的，python下似乎只有[pymongo][7]这个库可以用，其实也够用了。

### 主要遇到的问题: ###
mongodb 在 `sort()` 排序上如果数据量太大就会报内存不够的错误，默认的限制只有32M大小。

    pymongo.errors.OperationFailure: Executor error during find command :: caused by :: Sort operation used more than the maximum 33554432 bytes of RAM. Add an index, or specify a smaller limit.

解决办法就是给数据集合创建索引

    db.collection.createIndex(keys, options)

`collection`为需要创建索引的集合, `keys`为需要创建的字段， `options`可选`-1`表示按降序来排列,`1`表示升序排列

创建完毕后可用以下命令查看创建的结果

    db.collection.getIndexes()


  [1]: https://www.kumaodm.com
  [2]: http://flask.pocoo.org/
  [3]: https://www.mongodb.com/
  [4]: http://nginx.org/
  [5]: http://gunicorn.org/
  [6]: https://www.mdui.org/
  [7]: http://api.mongodb.com/python/current/
  [8]: https://i.loli.net/2019/10/02/36mEjqosC7GOUT9.jpg