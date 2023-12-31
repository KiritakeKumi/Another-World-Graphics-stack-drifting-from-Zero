### **Linux内核中的DRM：从原理到实践**

#### **1. 引言 (5分钟)**

- 图形的基础概念

  :

  - 从单一的文本模式终端到今天的高分辨率3D显示，计算机的图形展示经历了长足的发展。

- Linux下的图形堆栈简介

  :

  - 在Linux中，图形堆栈包括从应用程序、窗口系统（例如Xorg或Wayland）、DRM/KMS到最终的硬件显示。每一层都有其特定的任务，共同确保图形内容正确、快速地呈现给用户。

#### **2. 什么是DRM？ (10分钟)**

- DRM的历史和发展

  :

  - DRM始于20世纪90年代，旨在为Linux提供直接的、无中介的图形硬件访问。

- DRM的主要组成部分

  :

  - DRM不仅仅是一个驱动，它包括内存管理、命令队列、同步和模式设置等多个部分。

- 与其他系统组件交互

  :

  - DRM通常与窗口系统（例如X或Wayland）配合使用，为应用程序提供硬件加速。

#### **3. DRM的核心概念 (10分钟)**

- 内存管理和显存

  :

  - GEM（Graphics Execution Manager）和TTM（Translation Table Maps）是DRM中的内存管理方案。它们确保图形数据在需要的时候能够快速地传输给GPU。

​       	

​			***GEM（Graphics Execution Manager）***

*GEM是一个在Linux内核中的API，专门为图形驱动设计，用于统一显存管理。GEM的出现主要是为了解决之前显存管理方式中存在的缺陷和限制。*

#### ***1. GEM的主要目标：***

- ***简化内存管理**：GEM提供了一种简单的方式来管理显存，使得开发人员可以更容易地编写驱动程序。*
- ***提高性能**：通过充分利用硬件加速和高效的显存管理，GEM可以提高图形渲染的性能。*
- ***提供统一的用户空间接口**：GEM为用户空间应用程序提供了一组统一的API，使得应用程序可以更容易地与不同的图形驱动程序进行交互。*

#### ***2. GEM的核心概念：***

- ***BO (Buffer Object)**：这是GEM的基础数据结构，代表了一个连续的内存区域，可以被CPU和GPU共享访问。*
- ***Handle**：当用户空间请求一个Buffer Object时，它会收到一个句柄（Handle），而不是直接的内存地址。这样做的目的是为了安全和抽象。*
- ***Fencing**：为了确保数据的一致性和完整性，GEM引入了围栏（Fencing）机制，这可以确保在一个新操作开始之前，前一个操作已经完成。*

#### ***3. GEM如何工作：***

1. ***分配和管理显存**：应用程序通过GEM API请求显存，GEM返回一个句柄给应用程序。*
2. ***命令提交**：应用程序填充一个命令缓冲区并通过GEM提交给GPU执行。*
3. ***同步**：GEM确保所有的命令都按照正确的顺序执行，并且在新命令开始之前完成前一个命令。*
4. ***显存释放**：当Buffer Object不再需要时，它会被标记为可释放的，GEM会负责回收其内存。*

#### ***4. 为什么GEM很重要：***

*在GEM之前，显存管理在Linux中没有统一的标准。这意味着每个图形驱动都有自己的方式来处理显存，导致了很大的复杂性和不一致性。GEM为图形驱动开发人员提供了一个清晰、一致的框架来处理显存，从而简化了驱动程序的开发过程，并提高了整体的图形性能。*

------

*GEM主要由Intel推动开发，但现在已经被多种GPU驱动采用。它的引入标志着Linux图形子系统的一个重要的进步，为提高稳定性、性能和兼容性奠定了基础。*



## ***TTM（Translation Table Maps）***

#### ***1. TTM的主要目标：***

- ***支持不同的显存类型**：TTM被设计为支持多种显存类型，如本地VRAM、AGP内存、系统RAM等。*
- ***支持高度异构的硬件**：TTM提供更大的灵活性，可以适应各种显存管理需求，这对于非统一的硬件架构特别有用。*
- ***提高显存利用率**：通过更高效地管理显存，TTM旨在减少内存碎片并优化显存的利用。*

#### ***2. TTM的核心概念：***

- ***TTM Objects**：TTM对象表示一个显存区块。每个对象都有一个与其相关联的显存类型（如VRAM、AGP、系统RAM）。*
- ***Bind & Unbind**：这是TTM的两个主要操作。当一个TTM对象被绑定时，它被映射到一个特定的显存地址。解绑操作则将对象从该地址中移除。*
- ***Eviction**：当显存不足时，TTM可以决定将某些TTM对象“驱逐”出显存。这些对象的内容可能被移动到系统RAM或其他位置。*
- ***Booting**：在TTM中，“booting”是指将一个TTM对象从一个显存类型移动到另一个显存类型。*

#### ***3. TTM如何工作：***

1. ***显存分配**：应用程序或驱动请求一个TTM对象，指定所需的显存类型和大小。*
2. ***映射管理**：TTM对象可以被绑定到显存，使其可以被GPU访问。此映射可以在运行时改变，以优化性能或管理显存。*
3. ***命令提交与数据传输**：应用程序可以将数据上传到TTM对象，并提交命令给GPU，指示它如何处理该数据。*
4. ***显存管理**：TTM在运行时会监控显存的使用情况，以确保最佳性能。当显存紧张时，TTM可以选择移动、驱逐或换出TTM对象。*

#### ***4. TTM与GEM的关系：***

*虽然TTM和GEM都是显存管理框架，但它们的设计哲学和目标有所不同。GEM是为Intel硬件设计的，而TTM则旨在满足更多种类的显存和硬件需求。不过，值得注意的是，在某些GPU驱动中，TTM和GEM可以并存并共同工作，其中TTM负责显存管理，而GEM负责命令调度和同步。*

------

*TTM（Translation Table Maps）是另一种在Linux内核中用于图形显存管理的框架。它主要由AMD为其显卡驱动开发，并被设计用来满足不同于Intel图形硬件的需求。TTM后来也被其他GPU制造商使用，尤其是那些具有不同内存访问模型的GPU。*



- 命令队列和GPU的任务调度

  :

  - GPU并不总是立即执行每一个任务。DRM确保任务按照正确的顺序进入队列，并在适当的时间执行。

- 同步机制

  :

  - 为了防止“撕裂”等图形错误，同步确保在前一个任务完成之前不会开始下一个任务。

- DRM和用户空间的交互

  :

  - DRM提供了ioctl系统调用，允许用户空间应用程序与其交互。

#### **4. KMS（Kernel Mode Setting）简介 (10分钟)**

- 什么是模式设置？

  :

  - 模式设置是改变显示设备的分辨率、颜色深度和刷新率的过程。

- 用户模式设置vs内核模式设置

  :

  - 传统上，模式设置是在用户空间完成的。但KMS允许这一过程在内核空间中完成，提供了更快、更稳定的切换。

- KMS的优势

  :

  - KMS可以在系统启动时即设置正确的模式，减少了从启动到图形界面的过渡时间，并提供了更加稳定的用户体验。

#### **5. DRM驱动程序结构 (10分钟)**

- 设备驱动程序和硬件交互的原理

  :

  - 驱动程序是硬件和操作系统之间的桥梁。对于GPU来说，这意味着转换操作系统的指令，使硬件能够理解并执行。

- 常见的开源GPU驱动

  :

  - 在Linux中，有多种开源GPU驱动，包括但不限于Intel的i915驱动，AMD的amdgpu驱动，以及NVIDIA的开源驱动Nouveau。

#### **6. 实践：创建和调试一个简单的DRM驱动 (10分钟)**

- 基础的驱动结构和组成

  :

  - 一个DRM驱动的基础结构包括初始化硬件、设置内存管理、定义GPU命令、以及与用户空间应用程序的交互等部分。

- 如何处理和调试驱动问题

  :

  - 当遇到问题时，内核日志、dmesg和特定的调试工具都是非常有价值的资源。







引用来源：

1.https://betawiki.net/wiki/MS-DOS_1.25

2.https://www.bilibili.com/video/BV1Jq4y1i7Jc

