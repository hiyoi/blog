---
layout: post
cid: 34
title: python - 一只async爬虫
slug: python-async-spider
date: 2018/01/03 17:22:00
updated: 2019/10/02 13:31:50
status: publish
author: alex
categories: 
  - 默认分类
  - 技术
tags: 
  - python
  - asyncio
  - aiohttp
---


最近有个业务场景:
需要抓出[coinmarketcap][1]下的所有虚拟币链接里面的官网地址

![coin][2]
![website][3]

比如 [bitcoin][4] 里面的 [website][5] 和 [website2][6]

这样算起来大概有1300多个url，如果直接采用顺序抓取，那io瓶颈会十分明显

使用requests来爬取
```
# coding: utf-8
from __future__ import print_function
import requests
from bs4 import BeautifulSoup
from bs4 import SoupStrainer
import csv
import os
import time


class Spider(object):
    def __init__(self):
        self.session = requests.session()
        self.targetUrl = None
        self.headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36',
        }

    def getResponse(self, url=None):
        try:
            self.session.headers = self.headers
            self.targetUrl = url
            resp = self.session.get(self.targetUrl, headers=self.headers)
            return resp
        except Exception as e:
            print(e)

    def getCoinList(self):
        url = 'https://files.coinmarketcap.com/generated/search/quick_search.json'
        try:
            resp = self.getResponse(url)
        except Exception as e:
            print(e)
        return resp.json()

    def getWebSite(self, slug):
        url = 'https://coinmarketcap.com/currencies/%s/' % slug
        try:
            html = self.getResponse(url).content
            only_a_title = SoupStrainer('ul', attrs={'class': 'list-unstyled'})
            soup = BeautifulSoup(html, "lxml", parse_only=only_a_title)
            links = soup.select('span[title="Website"] ~ a')
            r = []
            for x in links:
                r.append(x['href'])
            return ' , '.join(r)
        except Exception as e:
            print(e)
            return ''

    def dumpCSV(self):
        path = os.path.join(os.path.split(os.path.realpath(__file__))[0], 'coin.csv')
        results = self.getCoinList()
        with open(path, 'w') as f:
            fieldnames = ['rank', 'name', 'symbol', 'website']
            wr = csv.DictWriter(f, fieldnames=fieldnames)
            wr.writeheader()
            i = 1
            for x in results:
                print(i, x['name'])
                websites = self.getWebSite(x['slug'])
                wr.writerow({'name': x['name'], 'symbol': x['symbol'], 'rank': x['rank'], 'website': websites})
                i += 1


if __name__ == '__main__':
    spider = Spider()
    start = time.time()
    spider.dumpCSV()
    print(time.time() - start)

```
执行下来，大概需要830s，也就是13分钟。
这样的速度实在太慢，改用asyncio+aiohttp，性能暴增
```
# coding:utf-8
import aiohttp
import asyncio
from bs4 import BeautifulSoup
from bs4 import SoupStrainer
import json
import csv
import os
import time
import sys

# Linux下使用uvloop代替自带的loop
if sys.platform == 'linux':
    import uvloop
    asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())

# winddows下使用IOCP
if sys.platform == 'win32':
    loop = asyncio.ProactorEventLoop()
    asyncio.set_event_loop(loop)


async def getWeb(slug, sem):
    headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36'}
    url = 'https://coinmarketcap.com/currencies/{}/'.format(slug)
    try:
        with await sem:
            async with aiohttp.ClientSession() as session:
                async with session.get(url, headers=headers, timeout=10) as resp:
                    return await resp.text()
    except Exception as e:
        print(e)


# 先抓取列表
async def getList():
    global result
    url = 'https://files.coinmarketcap.com/generated/search/quick_search.json'
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            result = await resp.text()


async def main(slug, sem):
    global link
    # 因为数量不是很大,这里直接用字典来缓存结果
    # 数量多的话可以使用redis来做缓存
    link = {}
    try:
        html = await getWeb(slug, sem)
        # 使用beautifulsoup解析文档
        only_a_title = SoupStrainer('ul', attrs={'class': 'list-unstyled'})
        soup = BeautifulSoup(html, "lxml", parse_only=only_a_title)
        links = soup.select('span[title="Website"] ~ a')
        r = []
        if links is not None:
            for x in links:
                r.append(x['href'])
            link[slug] = ','.join(r)
            print(r)
    except Exception as e:
        print(e)

try:
    start = time.time()
    loop = asyncio.get_event_loop()
    loop.run_until_complete(getList())

    # 控制并发数
    sem = asyncio.Semaphore(200)
    tasks = [main(x['slug'], sem) for x in json.loads(result)]

    loop.run_until_complete(asyncio.wait(tasks))
except Exception as e:
    print(e)

path = os.path.join(os.path.split(os.path.realpath(__file__))[0], 'coin.csv')

# 把抓取结果写到csv文件里面
with open(path, 'w') as f:
    fieldnames = ['rank', 'name', 'symbol', 'website']
    wr = csv.DictWriter(f, fieldnames=fieldnames)
    wr.writeheader()
    for x in json.loads(result):
        print(x['rank'], x['name'])
        s = x['slug']
        if s in link:
            site = link[s]
        else:
            site = ''
        wr.writerow({'name': x['name'], 'symbol': x['symbol'], 'rank': x['rank'], 'website': site})


print(time.time() - start)

```
执行下来只要不到1分钟，性能增加了10倍不止
带宽利用率也达到了最大化
![band][7]


  [1]: https://coinmarketcap.com/all/views/all/
  [2]: https://i.loli.net/2019/10/02/XS5Y1yMOoQFfptn.jpg
  [3]: https://i.loli.net/2019/10/02/4L3c5RsTXfDCNIY.jpg
  [4]: https://coinmarketcap.com/currencies/bitcoin/
  [5]: https://bitcoin.org/en/
  [6]: https://www.bitcoin.com/
  [7]: https://i.loli.net/2019/10/02/Ef6ikvj9oZGB4nq.jpg