---
layout: post
cid: 4
title: shadowsocks安装配置优化
slug: shadowsocks
date: 2017/09/22 16:19:00
updated: 2017/12/08 23:09:52
status: publish
author: alex
categories: 
  - 默认分类
  - 技术
tags: 
  - linux
  - python
---


参考：
https://github.com/shadowsocks/shadowsocks/blob/master/README.md
**python版:**
安装配置比较简单
Debian / Ubuntu:

    apt-get install python-pip
    pip install shadowsocks

**参数说明：**
**-p 端口号**
**-k 密码**
**-m 加密方式**

前台运行:

    ssserver -p 443 -k password -m rc4-md5


如果要后台运行：

    sudo ssserver -p 443 -k password -m rc4-md5 --user nobody -d start


如果要停止：

    sudo ssserver -d stop


如果要检查日志：

    sudo less /var/log/shadowsocks.log

用 -h 查看所有参数。也可以使用配置文件进行配置。

配置文件相关:

    https://github.com/shadowsocks/shadowsocks/wiki/Configuration-via-Config-File

创建一个config配置文件  /etc/shadowsocks.json. Example:

    {
        "server":"my_server_ip",
        "server_port":8388,
        "local_address": "127.0.0.1",
        "local_port":1080,
        "password":"mypassword",
        "timeout":300,
        "method":"aes-256-cfb",
        "fast_open": false
    }

前台运行:

    ssserver -c /etc/shadowsocks.json


后台运行:

    ssserver -c /etc/shadowsocks.json -d start
    ssserver -c /etc/shadowsocks.json -d stop

shadowsocks还有一个C语言编译版，叫shadowssocks-libev
编译完占用内存比python版小，兼容python版的运行命令和配置文件，安装配置复杂点.
参考: https://github.com/shadowsocks/shadowsocks/blob/master/README.md

**服务器优化:**

官方要给的优化参考: https://github.com/shadowsocks/shadowsocks/wiki/Optimizing-Shadowsocks

Linux 4.9+ 内核可以启用Google BBR 拥塞算法来加速TCP
我们用的谷歌VM服务器默认内核已经是4.10了，可以直接使用BBR

开启:

    echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
    echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

保存并生效:

    sysctl -p

参考
http://blog.csdn.net/dog250/article/details/52830576
http://blog.leanote.com/post/quincyhuang/google-bbr
https://www.zhihu.com/question/53559433/answer/135903103