---
layout:     post
title:      "Git常用操作"
date:       2018-09-08 10:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-09-08-01-bg.jpg"
catalog: true
tags:
    - Git
---


## Tag

&emsp;&emsp; Git默认提供了tag功能用于对指定的commit打上标签，以便我们可以根据tag快速找
到某个commit对应的信息，其中大致常用的命令有：

- git tag: 显示所有tag
- git tag [-m msg] \<tagname\> \<commit\>: 对某个commit添加tagname，并指定注释消息
- git checkout \<tagname\>:切换到指定tagname
- git show \<tagname\>: 显示对应tagname信息