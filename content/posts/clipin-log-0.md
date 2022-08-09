+++
title = 'Clipin Log 0. Mac OS上截图&保留小工具的开发记录'
date = 2021-01-09 20:50:05
tags = ['iOS dev']
+++
源码地址：[https://github.com/hagemon/Clipin](https://github.com/hagemon/Clipin)

源码的版本为Swift 5.0。

我们将截图操作称为**Clip**，将截图钉在屏幕上的操作称为**Pin**。

该工具可以临时将一些需要对照的信息保持在屏幕最上方，（或许）能提高工作效率。

该工具在很大程度上参考了[Snip](https://github.com/isee15/Capture-Screen-For-Multi-Screens-On-Mac)的源码，且集成了[HotKey](https://github.com/soffes/HotKey)开源库监听快捷键。

本文章是作者在开发该工具过程中的记录，由简入繁地介绍该工具的创建过程，如文中有纰漏或是不健康的代码，还请各位赐教。😋

本工具还会持续地开发和维护，该文章也会不断地更新和修改。

# 项目架构介绍
根据作者个人的习惯，将项目分为以下几个模块：

- Util：一些工具类、函数、枚举和扩展
- Manager：管理Clip、Pin操作过程的单例管理器
- Clip：Clip功能相关的Controller和View
- Pin：Pin功能相关的Controller和View

Status bar应用的相关配置和快捷键监听等功能在`AppDelegate.swift`中实现。

首先进入[Status Bar应用&监听快捷键](https://hagemon.github.io/post/clipin-log-1/)。
