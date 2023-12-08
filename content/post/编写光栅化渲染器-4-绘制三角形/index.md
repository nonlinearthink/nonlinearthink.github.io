---
title: 编写光栅化渲染器（四）绘制三角形
description: 
date: 2023-12-06T22:42:50+08:00
tags:
    - Computer Graphics
    - C++
categories:
    - 编写光栅化渲染器
image: 
comments: true
---

## 重心坐标（Barycentric Coordinate）

重心坐标可以被用来描述一个三角形内点的位置，定义如下：

$$
P=(1-\mu-\nu)A+\mu B+\nu C
$$

- 当 $1-\mu-\nu$、$\mu$、$\nu$ 均大于 0 小于 1 时，$P$ 位于三角形内部
- 当其中有一个分量等于 0 时，$P$ 在三角形边上
- 当其中有两个变量等于 0 时，$P$ 在某个顶点上

重心坐标的例子：

![Barycentric Coordinate](1.png)
