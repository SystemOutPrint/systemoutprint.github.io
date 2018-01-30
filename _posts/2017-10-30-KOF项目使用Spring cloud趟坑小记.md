---
layout: post
title:  "KOF项目使用Spring Cloud趟坑小记"
date:   2017-10-30
tags:
    - Spring
    - Spring Cloud
    - 游戏
---

* 经常有没使用过maven的同学过来说我这个又编译不过了，在mvn compile之前，先把kof-common使用mvn install到本地仓库。(有精力的可以在内网搭个私服)
* config server要在依赖配置中心的服务之前起。