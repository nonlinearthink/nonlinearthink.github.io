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

SDL (Simple DirectMedia Layer) 是一个开源、跨平台，非常简洁的多媒体层，作者目前在 `Valve` 任职，也就是大家最喜欢的 `Steam`，我们基本上只需要用到窗口和一些事件处理的功能就行。

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

### 创建 Renderer

在创建 `SDL_Window*` 后插入这个代码

```cpp
// 创建 SDL 渲染器
SDL_Renderer* renderer = SDL_CreateRenderer(window, -1, 0);
if (renderer) {
    SDL_Log("create renderer failed.");
    SDL_Quit();
    return nullptr;
}
```

这个渲染器其实已经实现了我们整个系列要做的事情，不过我们的目的是通过手写一个光栅化渲染器提高自己的图形学编程水平，所以不要在意，我们就是要重复造轮子。

```cpp
while (!isQuit) {
    ...

    // 设置默认颜色
    SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
    // 清空屏幕
    SDL_RenderClear(renderer);

    // 发送数据到 GPU
    SDL_RenderPresent(renderer_.get());
}
```

上面的例子可以看到 `SDL_Renderer` 底层实现其实是 GPU，我们的最终光栅化渲染程序其实是在它把数据提交给 GPU 之前在 CPU 中修改。

```cpp
SDL_DestroyRenderer(renderer);
```

**不要忘记释放堆指针。**

### 获取和更改 Surface

在 SDL 中，有两种方法可以去做渲染图形，一种是通过 `SDL_Surface`，一种是通过 `SDL_Texture`。

区别在于 `SDL_Surface` 是在 CPU 中，而 `SDL_Texture` 在 GPU 中，并且也可以通过 `SDL_Surface` 更新 `SDL_Texture`。

因为我们这里只需要写软渲染器，所以只使用 `SDL_Surface`。

```cpp
while (!isQuit) {
    ...

    SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
    SDL_RenderClear(renderer);

    // 获取窗口的 Surface
    SDL_Surface* surface = SDL_GetWindowSurface(window);
    // 更改 Surface 颜色
    SDL_FillRect(surface, NULL, SDL_MapRGB(surface->format, 0, 255, 0));
    // 更新窗口的 Surface
    SDL_UpdateWindowSurface(window);

    SDL_RenderPresent(renderer_.get());
}
```

## 改写 C 风格的代码

SDL 是一个 C 的库，而不是 C++，这样风格的代码在 C++ 中是不安全而且非常奇怪的。所以我们需要做一些改造。

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

>尝试自己实现 `SDL_Renderer` 的 `Deleter`

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
        soft_renderer_ =
            std::make_unique<SoftRenderer::Renderer>(width, height);
    }
    ~RenderApplication() { SDL_Quit(); }

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

        renderer_ =
            Unique_SDL_Renderer_Ptr(SDL_CreateRenderer(window_.get(), -1, 0));
        if (!renderer_) {
            SDL_Log("create renderer failed.");
            SDL_Quit();
            return false;
        }

        render_target_texture_ = Unique_SDL_Texture_Ptr(SDL_CreateTexture(
            renderer_.get(), SDL_PIXELFORMAT_RGBA8888, 0, width, height));
        if (!render_target_texture_) {
            SDL_Log("create renderer target texture failed.");
            SDL_Quit();
            return false;
        }

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

        SDL_SetRenderDrawColor(renderer_.get(), 0, 0, 0, 255);
        SDL_RenderClear(renderer_.get());

        Unique_SDL_Surface_Ptr surface(SDL_CreateRGBSurfaceWithFormat(
            0, width, height, 32, SDL_PIXELFORMAT_RGBA8888));
        soft_renderer_->PrepareRender((Uint32*)surface->pixels);

        render();

        SDL_UpdateTexture(render_target_texture_.get(), NULL, surface->pixels,
                          surface->pitch);

        SDL_RenderCopy(renderer_.get(), render_target_texture_.get(), NULL,
                       NULL);

        SDL_RenderPresent(renderer_.get());
        SDL_Delay(1000 / 60);
    }

    virtual void render() {}

protected:
    std::unique_ptr<SoftRenderer::Renderer> soft_renderer_;
    Unique_SDL_Window_Ptr window_;
    Unique_SDL_Renderer_Ptr renderer_;
    Unique_SDL_Texture_Ptr render_target_texture_;
    std::unique_ptr<SDL_Event> event_;
};
```