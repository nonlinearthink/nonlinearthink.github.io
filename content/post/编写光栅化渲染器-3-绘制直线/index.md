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
math: true
---

## DDA Line

DDA(Digital Differential Analyzer) 正如其名，就是最直观的直线画法，原始的算法的描述如下：

假设存在屏幕空间上的两个点 $(x1, y1)$ 和 $(x2, y2)$

1. 计算 $dx=x2-x1$，$dy=y2-y1$。
2. 计算斜率 $k=\frac{dy}{dx}$。
3. `x` 从 `x1` 出发，每次向 `x2` 移动一个单位，计算 $y=y1+k(x–x1)$。

```cpp
void DDA(int x0, int y0, int x1, int y1)
{
    // 计算 dx & dy & k
    int dx = x1 - x0;
    int dy = y1 - y0;
    float k = dy / dx;

    // 循环绘制每一个像素
    for (int x = x1; x <= x2; x++) {
        putpixel(x, round(y1 + k * (x - x1)), RED);
    }
}
```

原始算法看起来很可靠，但是仍然有一些可以优化的地方，因为直线是线性且均匀的，所以假如提前计算好了每一次循环的增量，就可以避免 `y1 + k * (x - x1)` 中的浮点数乘法。

优化过后的 DDA 算法如下：

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

从上面 DDA 的优化案例可以看出，避免浮点数操作就是优化画线算法的关键。

`Bresenham's Line` 相比 DDA 不仅有更少的浮点数运算，而且没有浮点数和整数的类型转换。

算法的核心思想如下：

![Bresenham's Line](1.png)

在 $(x_k, y_k)$ 的位置时候，可能走向 $(x_k+1, y_k)$ 也可能走向 $(x_k+1, y_k+1)$，显然，斜线的交点更加靠近谁，就往哪个方向走。

斜率:

$$
k = \frac{\Delta y}{\Delta x}
$$

对于每一次循环，执行：

$$
x_{i+1} = x_i + 1\\
e_{i+1} = e_i + k\\
$$

同时，始终保证 $0 < e < 1$：

$$
e_{i+1} = e_{i+1} - 1, e_{i+1} > 1
$$

最后，得出这个点的 `y`：

$$
y_{i+1} = \begin{cases}
   y_i+1 &\text{if } e_{i+1} \gt 0.5\cr
   y_i &\text{if } e_{i+1} \le 0.5\cr
\end{cases}
$$

---

上面的算法是 `Bresenham's Line` 的基本思想，还需要进一步优化，减少浮点数运算。

可能产生浮点数的地方是 $k = \frac{\Delta y}{\Delta x}$ 和 $e_{i+1} \gt 0.5$，所以我们最后再把上面所有的过程乘以 $2\Delta x$。

最终，我们的代码如下，其中 $\Delta x = x_2 - x_1, \Delta y = y_2 - y_1$：

```cpp
void bresenham(int x1, int y1, int x2, int y2)
{
    int m = 2 * (y2 - y1);
    int slope_error = m - (x2 - x1);
    for (int x = x1, y = y1; x <= x2; x++) {
        putpixel(x, y, RED);

        slope_error += m;

        if (slope_error >= 0) {
            y++; 
            slope_error -= 2 * (x2 - x1); 
        }
    }
}
```

> 最后的代码虽然看起来简洁，但是因为优化过，第一次接触容易摸不着头脑。

## 其他算法

还有一种叫 [Mid-Point Line](https://www.geeksforgeeks.org/mid-point-line-generation-algorithm/) ，因为它即没有 `DDA` 简单直接，也没有 `Bresenham` 效率高，这里就不介绍了。
