---
title: 编写光栅化渲染器（三）绘制直线
description: 
date: 2023-12-05T23:12:03+08:00
tags:
    - Computer Graphics
    - C++
    - Digital Differential Analyzer
    - Bresenham's Line
categories:
    - 编写光栅化渲染器
image: 
comments: true
---

## Digital Differential Analyzer(DDA)

DDA 是最容易理解和简单的直线画法，虽然它的性能不是最好的，算法的描述如下：

假设存在屏幕空间上的两个点 $(x1, y1)$ 和 $(x2, y2)$

1. 计算 $dx=x2-x1$，$dy=y2-y1$。
2. 计算斜率 $k=\frac{dy}{dx}$。
3. `x` 从 `x1` 出发，每次向 `x2` 移动一个单位，计算 $y=y1+k(x–x1)$。

在实际写程序的时候，有一些可以优化的地方，比如可以计算增量，优化过后的 DDA 算法的 C++ 实现：

```cpp
void DDA(int x0, int y0, int x1, int y1)
{
    // 计算 dx & dy 
    int dx = x1 - x0;
    int dy = y1 - y0;

    // 计算需要计算多少步像素
    int steps = abs(dx) > abs(dy) ? abs(dx) : abs(dy);

    // 计算 x & y 每一步的增量
    float xinc = dx / (float)steps;
    float yinc = dy / (float)steps;

    // 循环绘制每一个像素
    float x = x0;
    float y = y0;
    for (int i = 0; i <= steps; i++) {
        putpixel(round(x), round(y), RED);
        x += xinc;
        y += yinc;
    }
}
```

## Bresenham's Line

从算法分析的角度来说，DDA 已经是 `O(n)` 时间复杂度了，但是还有比它更高效的算法，原因是

![Bresenham's Line](1.png)
