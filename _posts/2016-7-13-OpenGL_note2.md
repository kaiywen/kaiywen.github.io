---
layout:     post
title:      "OpenGL 学习笔记（二）"
subtitle:   "坐标变换"
date:       2016-8-5 14:34:00
author:     "YuanBao"
header-img: "img/post-book2.jpeg"
header-mask: 0.35
catalog: true
tags:
 - programming
 - OpenGL
---

在介绍完 OpenGL 的概念和 Mac 下环境设置以后，接下来我们先从整体上介绍一下 OpenGL 的渲染流程。我们知道，OpenGL 中的任何物体都是使用三维坐标表示的，而经过渲染之后最终显示在我们眼前的却是屏幕中的二维坐标物体。因此，渲染流程其实就是**从三维坐标转换到二维屏幕像素的过程**。尽管这个定义看起来很简单，但是却包含了两个复杂的流程：空间坐标变换和像素渲染。本文介绍第一个过程，也就是**空间坐标变换**。

## 坐标变换







