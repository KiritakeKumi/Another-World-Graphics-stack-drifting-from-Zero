## 第四章 第二节 DRM 内部结构



### 前言

DRM 层为图形驱动程序提供多种服务，其中许多服务由它通过 libdrm 提供的应用程序接口驱动，libdrm 是包装大部分 DRM ioctl 的库。其中包括 vblank 事件处理、内存管理、输出管理、帧缓冲区管理、命令提交和防护、挂起/恢复支持以及 DMA 服务。

首先，我们回顾一些典型的驱动程序初始化要求，例如设置命令缓冲区、创建初始输出配置和初始化核心服务。后续部分更详细地介绍了核心内部结构，提供了实现说明和示例。



#### 1.1 初始化驱动程序

每个 DRM 驱动程序的核心都是一个结构。驱动程序通常静态初始化 drm_driver 结构，然后将其传递给 分配设备实例。设备实例完全初始化后，可以使用 进行注册（这使得可以从用户空间访问它）。

[`struct drm_driver`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_driver)[`drm_dev_alloc()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_dev_alloc)[`drm_dev_register()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_dev_register)

该结构包含描述驱动程序及其支持的功能的静态信息，以及指向 DRM 核心将调用以实现 DRM API 的方法的指针。我们将首先浏览静态信息字段，然后详细描述后面各节中使用的各个操作。

[`struct drm_driver`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_driver)[`struct drm_driver`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_driver)



##### 1.1.1 驱动信息



###### 1.1.1.1 主要、次要和补丁级别



int major; int minor; int patchlevel; DRM 核心通过主要、次要和补丁级别三元组来识别驱动程序版本。该信息在初始化时打印到内核日志，并通过 DRM_IOCTL_VERSION ioctl 传递到用户空间。



主要编号和次要编号还用于验证传递给 DRM_IOCTL_SET_VERSION 的请求驱动程序 API 版本。当驱动程序 API 在次要版本之间发生更改时，应用程序可以调用 DRM_IOCTL_SET_VERSION 来选择 API 的特定版本。如果请求的主要版本不等于驱动程序主要版本，或者请求的次要版本大于驱动程序次要版本，则 DRM_IOCTL_SET_VERSION 调用将返回错误。否则，将使用请求的版本调用驱动程序的 set_version() 方法。



###### 1.1.1.2 名称、描述和日期



字符*名称；字符*描述；字符*日期；驱动程序名称在初始化时打印到内核日志，用于IRQ注册并通过DRM_IOCTL_VERSION传递到用户空间。

驱动程序描述是通过 DRM_IOCTL_VERSION ioctl 传递到用户空间的纯粹信息字符串，否则内核不会使用它。

驱动程序日期的格式为 YYYYMMDD，旨在标识驱动程序最新修改的日期。然而，由于大多数驱动程序都无法更新它，因此它的价值基本上没有用处。DRM 核心在初始化时将其打印到内核日志，并通过 DRM_IOCTL_VERSION ioctl 将其传递到用户空间。



##### 1.1.2 模块初始化



该库提供在模块初始化和关闭期间注册 DRM 驱动程序的帮助程序。提供的帮助程序的行为类似于特定于总线的模块帮助程序，例如 module_pci_driver()，但遵循控制 DRM 驱动程序注册的其他参数。

下面是为 PCI 总线上的设备初始化 DRM 驱动程序的示例。



```
struct pci_driver my_pci_drv = {
};

drm_module_pci_driver(my_pci_drv);
```



生成的代码将测试是否启用了 DRM 驱动程序并注册 PCI 驱动程序 my_pci_drv。对于更复杂的模块初始化，您仍然可以在驱动程序中使用[`module_init()`](https://dri.freedesktop.org/docs/drm/driver-api/basics.html#c.module_init)and 。

[`module_exit()`](https://dri.freedesktop.org/docs/drm/driver-api/basics.html#c.module_exit)



##### 管理帧缓冲区 Aperture 使用



图形设备可能受不同驱动程序支持，但在任何给定时间只能有一个驱动程序处于活动状态。

许多系统在启动过程的早期加载通用图形驱动程序，例如 EFI-GOP 或 VESA 。在稍后的启动阶段，它们用专用的、特定于硬件的驱动程序替换通用驱动程序。要接管设备，专用驱动程序首先必须删除通用驱动程序。DRM 孔径功能管理 DRM 帧缓冲区内存的所有权以及驱动程序之间的切换。

DRM 驱动程序应使用 [`drm_aperture_remove_conflicting_framebuffers()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_aperture_remove_conflicting_framebuffers) 在其探测函数的顶部调用。该函数删除当前与给定帧缓冲区内存关联的任何通用驱动程序。如果帧缓冲区位于 PCI BAR 0，则 rsp 代码如下例所示。



```
static const struct drm_driver example_driver = {
        ...
};

static int remove_conflicting_framebuffers(struct pci_dev *pdev)
{
        resource_size_t base, size;
        int ret;

        base = pci_resource_start(pdev, 0);
        size = pci_resource_len(pdev, 0);

        return drm_aperture_remove_conflicting_framebuffers(base, size,
                                                            &example_driver);
}

static int probe(struct pci_dev *pdev)
{
        int ret;

        // Remove any generic drivers...
        ret = remove_conflicting_framebuffers(pdev);
        if (ret)
                return ret;

        // ... and initialize the hardware.
        ...

        drm_dev_register();

        return 0;
}
```

PCI 设备驱动程序应该调用 [`drm_aperture_remove_conflicting_pci_framebuffers()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_aperture_remove_conflicting_pci_framebuffers)并让它自动检测帧缓冲区。

不知道帧缓冲区位置的设备驱动程序应调用[`drm_aperture_remove_framebuffers()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.drm_aperture_remove_framebuffers)，这会删除已知帧缓冲区的所有驱动程序。

容易被其他驱动程序删除的驱动程序（例如通用 EFI 或 VESA 驱动程序）必须将自己注册为给定帧缓冲区内存的所有者。

帧缓冲区内存的所有权是通过调用获得的[`devm_aperture_acquire_from_firmware()`](https://dri.freedesktop.org/docs/drm/gpu/drm-internals.html#c.devm_aperture_acquire_from_firmware)。

成功后，驱动程序就是帧缓冲区范围的所有者。如果帧缓冲区已被另一个驱动程序拥有，则该函数将失败。请参阅下面的示例。

```
static int acquire_framebuffers(struct drm_device *dev, struct platform_device *pdev)
{
        resource_size_t base, size;

        mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);
        if (!mem)
                return -EINVAL;
        base = mem->start;
        size = resource_size(mem);

        return devm_acquire_aperture_from_firmware(dev, base, size);
}

static int probe(struct platform_device *pdev)
{
        struct drm_device *dev;
        int ret;

        // ... Initialize the device...
        dev = devm_drm_dev_alloc();
        ...

        // ... and acquire ownership of the framebuffer.
        ret = acquire_framebuffers(dev, pdev);
        if (ret)
                return ret;

        drm_dev_register(dev, 0);

        return 0;
}
```
