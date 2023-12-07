---
title: 编写光栅化渲染器（二）SDL库简介
description: 
date: 2023-12-05T12:08:06+08:00
tags:
    - C++
    - SDL
categories:
    - 编写光栅化渲染器
image: 
comments: true
---

我们的第一个目标就是创建一个窗口，创建窗口有很多库都能做，如果你不想要用任何第三方库也不需要跨平台，用 Win32 API 就可以做，不过微软的匈牙利命名法实在是抽象的，要有心里准备。

## SDL 库简介

SDL (Simple DirectMedia Layer) 是一个开源、跨平台、轻量级的多媒体层，作者目前在 `Valve` 任职，也就是大家最喜欢的 `Steam`，我们基本上只需要用到 `SDL_Window` 和  `SDL_Surface` 就可以。

这里默认大家拥有 C++ 和 CMake 基础，不介绍怎么安装这个库了，直接开始。

### 创建 Window

```cpp
#define SDL_MAIN_HANDLED

#include "SDL.h"

int main(int argc, char* argv[]) { 
    // 定义窗口大小
    const int width = 640;
    const int height = 480;

    // SDL 初始化
    if (SDL_Init(SDL_INIT_EVENTS) < 0) {
        SDL_Log("SDL init failed");
        return 1;
    }

    // 创建 SDL 窗口
    SDL_Window* window = SDL_CreateWindow(
        "SoftRenderer", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED,
        wWidth, wHeight, SDL_WINDOW_SHOWN);
    if (!window) {
        SDL_Log("create window failed");
        SDL_Quit();
        return 1;
    }

    // 一直循环，直到触发了 SDL_QUIT 事件（窗口被关闭）
    bool isQuit = false;
    SDL_Event event;
    while (!isQuit) {
        while (SDL_PollEvent(&event)) {
            if (event.type == SDL_QUIT) {
                isQuit = true;
            }
        }
    }

    // 释放资源
    SDL_DestroyWindow(window);
    SDL_Quit();

    return 0;
}
```

这是一个最简单的 SDL 创建窗口的程序。

### SDL CPU 渲染：获取和更改 Surface

从 Window 可以获取一个 Surface，它和 Window 的大小相同高，并且它里面有一个 `pixels` 指针，可以访问每一个像素的数据。

```cpp
while (!isQuit) {
    ...
    // 获取窗口的 Surface
    SDL_Surface* surface = SDL_GetWindowSurface(window);
    // 像素数据在 surface->pixels 中
    // 更改 Surface 颜色
    SDL_FillRect(surface, NULL, SDL_MapRGB(surface->format, 0, 255, 0));
    // 更新窗口的 Surface
    SDL_UpdateWindowSurface(window);
}
```

从 Window 获取的 Surface 的格式可能并不是我们想要的。

默认的 Window Surface 每一个像素是 32 位，但是实际上用的只有 24 位，我们就需要给每个像素写入这样：

```cpp
0x00FF0000 // RGB 红色
```

因为计算机中存储非 2 的幂次大小的数据会有性能问题，所以 SDL 这里数据是 32 位的，实际使用却是 24 位，头两位是无意义的。

我们可能更希望使用 RGBA 格式的数据，RGBA 刚好就是 32 位：

```cpp
0xFF0000FF // RGBA 红色
```

这个时候我建议自己额外建一个 Surface，渲染完之后再更新给 Window 的 Surface。

```cpp
while (!isQuit) {
    ...
    // 创建 RGBA 的 render surface
    SDL_Surface* render_surface = SDL_CreateRGBSurfaceWithFormat(
            0, width, height, 32, SDL_PIXELFORMAT_RGBA8888);

    // 渲染 render surface
    SDL_FillRect(render_surface, NULL, SDL_MapRGBA(surface->format, 0, 255, 0, 255));

    // 获取 window surface
    SDL_Surface* window_surface = SDL_GetWindowSurface(window);
    // 更新 render surface 到 window surface
    SDL_BlitSurface(render_surface, NULL, window_surface, NULL);
    // 更新 window surface 到 screen
    SDL_UpdateWindowSurface(window);

    // 释放 render surface
    SDL_FreeSurface(render_surface);
}
```

### SDL GPU 渲染：Renderer 和 Texture

>这一小节为补充内容，如果目标是做 CPU 的软渲染器，不需要了解。

在 SDL 中，有两种方法可以去做渲染图形，一种是通过 `SDL_Surface`，另外一种是通过 `SDL_Texture`。

区别在于 `SDL_Surface` 是在 CPU 中，而 `SDL_Texture` 在 GPU 中。

使用 GPU 渲染必须要先创建 Renderer：

```cpp
SDL_Renderer* renderer = SDL_CreateRenderer(window, -1, 0);
if (!renderer) {
    SDL_Log("create renderer failed.");
    SDL_Quit();
    return 1;
}
```

还需要创建一张 Texture，我们不会直接绘制 Renderer，而是绘制在 Texture 上，在计算机图形学中，这种图像绘制在纹理上而不是屏幕上的技术叫做 `Render Target Texture`。

```cpp
SDL_Texture* rtt = SDL_CreateTexture(app->_renderer, SDL_PIXELFORMAT_RGBA8888,
                                      0, width, height);
if (!rtt) {
    SDL_Log("create renderer target texture failed.");
    SDL_Quit();
    return 1;
}
```

每帧调用：

```cpp
while (!isQuit) {
    ...
    // 设置默认背景颜色
    SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
    // 清空屏幕
    SDL_RenderClear(renderer);

    // 创建一个 Surface 来更改像素
    SDL_Surface* surface = SDL_CreateRGBSurfaceWithFormat(
            0, width, height, 32, SDL_PIXELFORMAT_RGBA8888);

    // 渲染，更改像素
    SDL_FillRect(surface, NULL, SDL_MapRGBA(surface->format, 0, 255, 0, 255));

    // 更新 Surface 到 Texture 上
    SDL_UpdateTexture(rtt, NULL, surface->pixels, surface->pitch);
    SDL_FreeSurface(surface);

    // 覆盖 Texture 到 Renderer 里面
    SDL_RenderCopy(renderer, rtt, NULL, NULL);

    // 发送数据到 GPU
    SDL_RenderPresent(renderer);
}
```

## 改写 C 风格的代码

SDL 是一个 C 的库，而不是 C++，这样风格的代码在 C++ 中是不安全和不便利的。所以我们需要做一些改造。

### 使用智能指针

SDL 中有很多成对的比如 `SDL_CreateWindow` 和 `SDL_DestroyWindow` 函数。如果我们能自动释放就好了，可以用智能指针来解决这个问题。

尝试重写智能指针的 `Deleter`：

```cpp
struct SDL_Window_Deleter {
    void operator()(SDL_Window* window) const { SDL_DestroyWindow(window); }
};
using Unique_SDL_Window_Ptr = std::unique_ptr<SDL_Window, SDL_Window_Deleter>;
```

使用起来也格外方便：

```cpp
Unique_SDL_Window_Ptr window = Unique_SDL_Window_Ptr(SDL_CreateWindow(
            "SoftRenderer", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED,
            width, height, SDL_WINDOW_SHOWN | SDL_WINDOW_RESIZABLE));
```

>尝试自己实现 `SDL_Surface` 的 `Deleter`

### 使用 Class 封装

之前的 SDL 代码大致可以分为两个部分，一个是在 `while` 循环之前的初始化阶段，还有一个是在 `while` 循环内部的每帧执行阶段。

```cpp
RenderApplication app(width, height);
if (!app->InitApplication()) {
    return 1;
}

while (app->running) {
    app->Tick();
}
```

最终，`RenderApplication` 长这个样子：

```cpp
class RenderApplication {
public:
    int width, height;
    bool running;

    RenderApplication(int _width, int _height)
        : width(_width), height(_height), running(true) {
        renderer_ = std::make_unique<SoftRenderer::Renderer>(
            width, height, SoftRenderer::Color::Black());
    }
    ~RenderApplication() {
        // 这里不需要释放，因为 window 会释放它
        window_surface_ = nullptr;
        SDL_Quit();
    }

    bool InitApplication() {
        if (SDL_Init(SDL_INIT_EVENTS) < 0) {
            SDL_Log("SDL init failed");
            return false;
        }

        window_ = Unique_SDL_Window_Ptr(SDL_CreateWindow(
            "SoftRenderer", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED,
            width, height, SDL_WINDOW_SHOWN | SDL_WINDOW_RESIZABLE));
        if (!window_) {
            SDL_Log("create window failed.");
            SDL_Quit();
            return false;
        }

        window_surface_ = SDL_GetWindowSurface(window_.get());

        event_ = std::make_unique<SDL_Event>();

        return true;
    }

    void Tick() {
        while (SDL_PollEvent(event_.get())) {
            if (event_->type == SDL_QUIT) {
                running = false;
                return;
            }
        }

        // 渲染前的准备
        Unique_SDL_Surface_Ptr render_surface(SDL_CreateRGBSurfaceWithFormat(
            0, width, height, 32, SDL_PIXELFORMAT_RGBA8888));
        renderer_->PrepareRender((Uint32*)render_surface->pixels);

        // 清空屏幕
        renderer_->Clear();

        // 自定义渲染函数
        render();

        // 更新 renderer surface 到 window surface
        SDL_BlitSurface(render_surface.get(), NULL, window_surface_, NULL);
        // 更新 window surface 到 screen
        SDL_UpdateWindowSurface(window_.get());

        // 保持稳定的帧数
        SDL_Delay(1000 / 60);
    }

    virtual void render() {}

protected:
    std::unique_ptr<SoftRenderer::Renderer> renderer_;
    Unique_SDL_Window_Ptr window_;
    SDL_Surface* window_surface_;
    std::unique_ptr<SDL_Event> event_;
};
```
