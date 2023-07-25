## 第四章 第二节 DRM 内部结构



### 前言

DRM 层为图形驱动程序提供多种服务，其中许多服务由它通过 libdrm 提供的应用程序接口驱动，libdrm 是包装大部分 DRM ioctl 的库。其中包括 vblank 事件处理、内存管理、输出管理、帧缓冲区管理、命令提交和防护、挂起/恢复支持以及 DMA 服务。

首先，我们回顾一些典型的驱动程序初始化要求，例如设置命令缓冲区、创建初始输出配置和初始化核心服务。后续部分更详细地介绍了核心内部结构，提供了实现说明和示例。



#### 1.1 初始化驱动程序

每个 DRM 驱动程序的核心都是一个结构。驱动程序通常静态初始化 drm_driver 结构，然后将其传递给 分配设备实例。设备实例完全初始化后，可以使用 进行注册（这使得可以从用户空间访问它）。

[`struct drm_driver`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_driver)[`drm_dev_alloc()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_dev_alloc)[`drm_dev_register()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_dev_register)

该结构包含描述驱动程序及其支持的功能的静态信息，以及指向 DRM 核心将调用以实现 DRM API 的方法的指针。我们将首先浏览静态信息字段，然后详细描述后面各节中使用的各个操作。

[`struct drm_driver`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_driver)[`struct drm_driver`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_driver)



