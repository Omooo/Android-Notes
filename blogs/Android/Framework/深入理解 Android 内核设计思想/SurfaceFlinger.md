---
SurfaceFlinger
---

#### 概述

SurfaceFlinger 是负责绘制应用 UI 的核心，从名字上可以看出其功能是将所有的 Surface 合成工作。不论使用什么渲染 API，所有的东西最终都是渲染到 Surface 上，Surface 代表 BufferQueue 的生产者端，并且由 SurfaceFlinger 消费。Android 平台所创建的 Window 都由 Surface 所支持，所有可见的 Surface 渲染到显示设备都是通过 SurfaceFlinger来完成的。

SurfaceFlinger 进程是由 init 进程创建的，运行在独立的 SurfaceFlinger 进程。Android 应用程序必须跟 SurfaceFlinger 进程交互，才能完成将应用 UI 绘制到 FrameBuffer（帧缓冲区）。这个交互便涉及到进程间的通信，采用 Binder IPC 方式，名为 "SurfaceFlinger" 的 Binder 服务端运行在 SurfaceFlinger 进程。



