---
layout:     post
title:      "rockgo实现思路"
subtitle:   "分布式游戏服务器框架"
date:       2022-09-01 23:36:00
author:     "Yaoxh6"
catalog: true
header-style: text
tags:
  - go
  - rpc
  - ecs
  - 分布式
---

# 前言
最近刚加入了[rockgo](https://github.com/zllangct/rockgo)开源项目,一边阅读源码一边记录。该项目是基于ECS思想,go语言实现的游戏服务端框架。截至今日,master分支实现的功能还不完全基于ECS,并且还有一些要完善。先对master分支现存的代码分析,后续完善代码补充文档。
