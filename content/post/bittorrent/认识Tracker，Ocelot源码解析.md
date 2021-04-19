---
title: "认识BT Tracker，Ocelot源码解析"
date: 2021-03-23T10:10:00+08:00
lastmod: 2021-03-23T10:10:00+08:00
draft: false
keywords: ["BT", "Tracker", "Ocelot"]
description: ""
tags: ["BitTorrent"]
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---
### 引言
**BitTorrent**([**BT**](https://en.wikipedia.org/wiki/BitTorrent))是一种基于P2P的通信协议，提供了一种非中心化的数据、文件共享方式。那P2P节点相互如何发现呢？最简单的做法就是建议一个中心化的服务器(Tracker)，来接收各个节点的上报，比如我需要下载哪个文件，我拥有哪些文件。这样就有了我们熟悉的种子torrent文件，帮助我们和Tracker通信。[**Ocelot**](https://github.com/WhatCD/Ocelot)是国外某著名音乐站点Tracker的开源项目，借助于此项目可以从代码层面学习和认识Tracker。

![PT站架构](https://d3i71xaburhd42.cloudfront.net/a591137526d605007eb75d4c6f9554c079397f06/3-Figure1-1.png)

### Ocelot 编译及运行
笔者的系统环境为Ubuntu 16，根据[安装文档](https://github.com/WhatCD/Gazelle/wiki/Ubuntu-16.04)，我们简化一下安装Ocelot需要的依赖为：

```sh
sudo apt install mysql-server mysql-client libmysql++-dev libmysqld-dev \
libev-dev git cmake libbz2-dev libtcmalloc-minimal4 libboost-all-dev
```

安装完mysql后初始化root密码，新建table如[gazelle.sql](https://github.com/WhatCD/Gazelle/blob/master/gazelle.sql)，考虑对该文件做如下修改

```sql
SET FOREIGN_KEY_CHECKS = 0;
SET GLOBAL sql_mode = 'ONLY_FULL_GROUP_BY,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';

DROP DATABASE IF EXISTS gazelle;

CREATE DATABASE gazelle CHARACTER SET utf8 COLLATE utf8_swedish_ci;

USE gazelle;
//...
```

上述修改去掉了sql_mode配置的_STRICT_TRANS_TABLES_, _NO_ZERO_DATE_，前者会导致```insert into for update```强制校验不允许default值的字段，后者会导致gazelle.sql中日期默认值如_0000-00-00 00:00:00_失败。

修改项目中config.cpp文件的mysql登陆用户名及密码：
```c
	add("mysql_username", "root");
	add("mysql_password", "xxxx");
```

笔者使用的C++ IDE为Clion，打开Ocelot工作目录，编写cmake编译配置文件CMakeLists.txt如
```
cmake_minimum_required(VERSION 3.19)
project(Ocelot)

set(CMAKE_CXX_STANDARD 14)

include_directories(. /usr/include/mysql/)
link_directories(/usr/lib/x86_64-linux-gnu/)

add_executable(Ocelot
        config.cpp
        config.h
        db.cpp
        db.h
        events.cpp
        events.h
        misc_functions.cpp
        misc_functions.h
        ocelot.cpp
        ocelot.h
        report.cpp
        report.h
        response.cpp
        response.h
        schedule.cpp
        schedule.h
        site_comm.cpp
        site_comm.h
        user.cpp
        user.h
        worker.cpp
        worker.h)

target_link_libraries(Ocelot
        ev
        mysqlpp
        pthread
        boost_system
        boost_iostreams)
```

这样，项目就可以启动了，运行后会看到控制台输出：
```
Using default config because './ocelot.conf' couldn't be opened
Start Ocelot with -c <path> to specify config file if necessary
Clearing xbt_files_users and resetting peer counts...done
Loaded 1 users
Loaded 1 torrents
Loaded 0 tokens
Assuming no whitelist desired, disabling
Sockets up on port 34000, starting event loop!
0 open, 0 connections (0/s), 0 requests (0/s)
0 open, 0 connections (0/s), 0 requests (0/s)
0 open, 0 connections (0/s), 0 requests (0/s)
...

```

### 测试
通过抓包发现，BT客户端Announce请求如
```sh
curl -H 'Host: 127.0.0.1' -H 'User-Agent: qBittorrent/4.3.2' --compressed 'http://127.0.0.1:3400/[32位passkey]/announce?info_hash=[40位种子pass_hash]&peer_id=[20位peer_id]&port=32779&uploaded=10984485&downloaded=270144869&left=0&corrupt=0&key=82D19544&numwant=200&compact=1&no_peer_id=1&supportcrypto=1&redundant=0'
```

参照上面的请求，在mysql中插入2条数据：
```mysql
insert into users_main 
	(`Username`, `IP`, `Enabled`,`torrent_pass`, `Email`, `PassHash`, `Secret`, `Title`, `PermissionID`) 
	values ('xxx', '127.0.0.1', 1,'[32位passkey]', '', '', '', '',0); 
insert into torrents
	(`GroupID`, `info_hash`, ...)
	values('0', '[40位种子pass_hash]', ...)
```

重启Ocelot，发送上面的curl请求。发现torrents，xbt_files_users有数据更新，初步测试通过。愉快地调试代码吧！

### 未完待续
[ev处理流程，BT协议及编码]

---

> 版权所有，转载请注明来源
