---
title: "Unity 3D Lecture Note 10"
date: 2018-06-26T11:38:52+08:00
lastmod: 2018-06-26T11:41:52+08:00
menu: "main"
weight: 50
author: "hansenbeast"
tags: [
    "Unity"
]
categories: [
    "Learning Notes"
]
# you can close something for this content if you open it in config.toml.
comment: false
mathjax: false
---

# 第十三章、多人游戏与网络

### 效果视频地址：

<https://pan.baidu.com/s/1BCIGL95NYnki5WnvRaGWxg>

> 参考博客：https://pmlpml.github.io/unity3d-learning/13-Multiplayer-and-Networking

由于时间有限，只在完成博客中的step-by-step基础上进行改动。

了解到的知识点：

1. 继承**NetworkBehaviour**类控制联网玩家游戏对象的运动
2. 玩家游戏对象添加NetworkIdentity，NetworkTransform，并注册NetworkManager用于Spawn。
3. [Command] ：本地玩家对象执行的代码，调用服务器执行的函数
4. [ClientRpc] ：服务器执行的代码，调用所有客户端执行函数
5. [SyncVar]：服务器授权变量，Spwan 自动同步到客户端
6. NetworkStartPosition制定玩家初始位置