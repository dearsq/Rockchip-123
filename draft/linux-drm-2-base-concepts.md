# Linux DRM (二) 基本概念和特性

在《Linux DRM (一) Display Server》我们了解了 DRM 诞生的历史缘由。
本篇我们朝着 DRM 本尊再走几步，先介绍几个 DRM 的基本概念。

## 一、楔子

上篇文章中我们有讲过 DRM 是 linux 下的图形渲染架构，用来管理显示输出、buffer 分配的。
应用程序可以直接操纵 drm 的 ioctl 或者是用 framebuffer 提供的接口进行显示相关操作。
后来大家觉得这样太 low 了，干脆封装成一个库吧。于是 libdrm 诞生了，它是一个库，其中提供了一系列友好的控制封装，让我们可以更加方便的进行显示控制。

要弄明白 DRM 是怎么把用户的绘图输出到显示屏上，我们绕不开了解这几个概念：

* Framebuffer
* CRTC
* Encoder
* Connector
* Display Device(LCD)

然后我们会介绍一下 DRM 的一些特性。

最后再用较少的篇幅为大家介绍一下 RK 平台 DRM Driver 所依赖的 Component Framework。

## 二、DRM 涉及的基本概念

![](http://ww1.sinaimg.cn/large/ba061518ly1fkx39mievyj20o5092gmf.jpg)

记住这张图~

让我们一起关注一下绿色方框中的五个 Block。

### 2.1 DRM Framebuffer 
它是一块内存区域，我把它理解为一块画布，驱动和应用层都能访问它。画画之前需要将它格式化，我们需要设定你要画油画还是国画（色彩模式，比如 RGB24，YUV 等），画布需要多大（分辨率）。

### 2.2 CRTC
直译为 阴极摄像管上下文。我要说它的作用是读取当前扫描缓冲区的像素数据并借助于PLL电路从其生成视频模式定时信号。你可能就开始嘟囔着抱怨什么鬼了。简单点，说话的方式简单点，简单的来说他就是显示输出的上下文，我把它理解为扫描仪。它对内连接 Framebuffer 地址，对外连接 Encoder。
它会扫描你画布（Framebuffer）上的内容，叠加上 Planes 的内容，传给 Encoder。

### 2.3 Planes
直译为 平面。它和 framebuffer 一样是内存地址。它的作用是干什么呢？想象这样一个场景，笔者正在很不专心地一边看动作大片一边写文章。动作大片每一帧的变化都很大，需要全幅更新，写文章半天挤不出一个字，基本上不需要更新。笔者的顽皮将显卡的使用拉上了两个极端。一种是全幅高速更新的 Video Mode，一种是文字交互这种小范围的 Graphics Mode。此时轮到 Planes 出马了，它给 Video 刷新提供了高速通道，使 Video 单独为一个图层，可以叠加在 Graphic 上或之下，并具有缩放等功能。

看看本节开始的那张图，Planes 是可以有多个的，所以最后供扫描仪（CRTC）扫描的画（图像）实际上往往是Framebuffer 和 Planes 的组合（Blending）。

### 2.4 Encoder 
直译为 编码器。它的作用就是将内存的 pixel 像素 编码（转换）为显示器所需要的信号。

你可以把它的作用想象为现在要将你的画在全世界不同的显示设备（Display Device）上显示，自然需要将其转化为不同的电信号，比如 DVID、VGA、YPbPr、CVBS、Mipi、eDP 等。

所以我们需要这样一个 Encoder 来进行信号转换的工作。它和 CRTC 之间的交互就是我们所说的 ModeSetting，其中包含了前面提供的色彩模式、还有时序（Timing）等。

### 2.5 Connector
直译为 连接器。Connector 常常对应于物理连接器 (VGA, DVI, FPD-Link, HDMI, DisplayPort, S-Video ...) 他会连接将一个物理显示输出设备 (monitor, laptop panel, ...) 。
与当前物理连接的输出设备相关的信息（如连接状态，EDID数据，DPMS状态或支持的视频模式）也存储在 Connector 内。


## 三、DRM 中的特性

### 3.1 GEM
Graphics Execution Manager
因为视频存储器大小增加，图形 API 比如 OpenGL 不断变复杂。在上下文切换时重新初始化显卡状态的话开销会相当大。另外，现在 Linux 桌面需要需要一个最佳的方式来和合成管理器共享屏幕外缓存。
这些需求导致需要开发新的方法来管理内核中的图形缓存区，即 GEM。

#### 3.1.1 GEM 内存管理
GEM 提供了一套 API，其拥有明确的内存管理原语。通过 GEM，用户空间程序可以创建、处理、销毁 GPU 视频存储器中的内存对象。这些对象被统一称为 “GEM 对象”，从用户空间的角度来看，他们是一直存在的，所以在程序重新获得 GPU 的时候不需要重新加载他们。当用户空间的程序需要大量的视频内存空间（来存储 framebuffer、纹理数据 或是 GPU 所需要的其他数据），它都会使用 GEM API 来为 DRM Driver 请求分配内存。这套 API 还会提供填充 buffer 和 释放 的功能。并且可以在用户空间的进程（因意外或是正常终止而）关闭 DRM 设备描述符时释放相关内存。

GEM 也提供了两个及以上的用户空间进程访问同一个 DRM 设备(也就是共享同一个 GEM 对象)的方法。GEM 提供了一个方法，flink（其实这个方法很不安全），来从 GEM 句柄中获取 GEM 名字。进程通过 IPC 将 GEM 名字（32bit整型）传递给另一个进程。接收到 GEM 名字的进程于是可以获得一个本地的 GEM 句柄，用以指向原始的 GEM 对象。
所以如果有一个恶意的第三方应用知道了 GEM 名字，就可以访问和修改 GEM 对象的内容。所以后来通过引入 DMA-BUF 机制来克服这个缺陷。

#### 3.1.2 GEM 内存同步
除了管理视频存储空间以外，视频内存管理器另一个重要的任务是处理 GPU 和 CPU 之间的内存同步（memory synchronization）问题。
因为现在的内存架构非常复杂，通常涉及了系统内存、视频内存的多级缓存（caches）。因此，视频内存管理器为了确保 GPU 和 CPU 共享数据的一致性还需要负责处理缓存的一致性。这意味着视频内存管理系统高度依赖于 GPU 硬件和年内存架构因此驱动具有特异性。

GEM 定义了用于内存同步的“内存域”，而这些内存域与GPU无关，它们采用 UMA 存储器架构设计，使其不再适用于其他内存架构，如具有单独 VRAM 的内存架构。因此，对外，DRM 驱动会向用户空间暴露统一的 GEM API，对内，驱动中会实现更适合特定硬件和内存架构的不同的内存管理器。

### 3.2 KMS
Kernel Mode Setting

为了正常工作，视频卡和图形适配器必须负责设置一些模式，包括屏幕分辨率、色彩深度、刷新率等，将其设置为自己或是拓展屏所支持的参数范围内。这个动作就是 mode-setting，它通常需要访问图形硬件 —— 即去操作视频卡的寄存器 的能力。

在开始使用 framebuffer 之前，或者是由应用程序或者是用户改变 mode 时，进行 mode-setting。

#### 3.2.1 UMS
最开始，用户空间想要使用图形的 framebuffer 也需要负责 mode-setting，因此他们需要访问 video 硬件的权限。类 Unix 系统中的 Display Server 比如 X Server 就是一个很好的例子，它的 mode-setting 被放在每种特定的图形卡各自的 DDX driver 中。

这种方法被称为 UMS（User space Mode-Setting），有几个严重的问题。它不仅打破了操作系统建立的硬件和程序的隔离，也让多进程同时进行 mode-setting 时，硬件会产生不一致的状态。
所以为了避免这些冲突，X Server 成为了实时 mode-settinng 的唯一用户空间程序，其他用户空间程序都依赖 X Server 来进行 mode-setting。
最初 mode-setting 是放在 X Server 的启动过程中，不过后来在其运行过程也可以 mode-setting 。

但是 UMS 有很多问题。
比如在 Linux 系统启动过程中，Linux 内核必须为 virtual console 设置 "minnimal text" mode。
另外，framebuffer driver 中也包含了配置 framebuffer 设备 mode 的代码。
为了解决这些冲突，有些 Display Server（比如 X Server）通过保存从图形环境切换到文本虚拟控制台时的 mode-setting 状态，并且在切换回去的时候恢复。
这又导致了新的问题，比如切换时的闪烁，输出设备的显示失败甚至损毁。


#### 3.2.2 mode-setting
为了解决这些问题，mode-setting 被单独放到了 Kernel 中，准确的说是 DRM 模块中。
然后，包括 X Server 在内的每个进程都可以命令内核来实现 mode-setting 操作，内核也会确保操作的一致性。这些加入 DRM 模块的新的内核 API 和 代码所执行的 mode-setting 的操作被称为 **KMS**。

它有太多好处了，第一个就是从 Kernel （console、fbdev）和 Userspace（X Server DDX Driver）干掉了那些重复的 mode-setting 的代码。第二就是对于图形操作系统不再需要关心 mode-setting 部分代码的编写了。第三就是因为提供了这种单一集中式的模式管理，console 和 X Server 不同实例的切换变得更加容易了。第四就是 mode-setting 放到内核后，可以从启动过程就开始使用它（这个在曾经也会导致闪烁问题）。

另外，因为 KMS 是内核的一部分，它也就可以去使用 kernel space 的很多资源（比如中断）。因为交由内核去管理了，在 suspend/resume 后的模式恢复变得更简单了。内核还让新显示设备的热插拔更加容易。

由于 mode-setting 和内存管理密切相关 ———— framebuffer 基本就是 内存buffer ———— 故它和图像内存管理紧密集成。这也是为什么它被放到 DRM 模块而不是独立作为一个子系统的原因。

#### 3.2.3 KMS Driver
为了不破坏 DRM API 的向后兼容性，KMS 为 DRM Driver 提供了一个特殊的特性。任何 DRM Driver 在注册 DRM Core 的时候需要选择是否采用 DRIVER_MODESET 标志，用来表示是否支持 KMS API。
支持 KMS API 的驱动为了和传统的 DRM Driver 驱动区分往往被称为 KMS Driver。

#### 3.2.4 KMS Device Mode

KMS 负责塑造和管理输出设备，将他们抽象为一系列的硬件模块（这些硬件模块常常会在显示控制器的显示输出管道上）。
这些模块我们前面都有介绍，CRTC、Planes、Encoder、Connector 。但是当时介绍的比较通俗，下面是 Wikipedia 上面的谷歌翻译版本：

CRTCs：每个 CRTC（来自 CRT 控制器）代表显示控制器的扫描引擎，指向 framebuffer。 CRTC 的目标是读取当前扫描缓冲区的像素数据并借助于PLL电路从其生成视频模式定时信号。
CRTC 的数量决定了硬件可以同时处理多少个独立的输出设备，因此为了使用多头配置，每个显示设备至少需要一个 CRTC。
两个或多个CRTC也可以在克隆模式下工作，如果它们扫描的是相同的帧缓冲区，便可以将相同的图像发送到多个输出设备。

Connectors: 连接器表示显示控制器从要显示的扫描输出操作发送视频信号的位置。
通常，KMS 中 Connectors 对应于物理连接器
(VGA, DVI, FPD-Link, HDMI, DisplayPort, S-Video ...) 他会连接将一个物理显示输出设备 (monitor, laptop panel, ...) 。
与当前物理连接的输出设备相关的信息（如连接状态，EDID数据，DPMS状态或支持的视频模式）也存储在连接器内。

Encoders: 显示控制器必须使用适合于预期连接器的格式 对 来自CRTC的视频模式定时信号 进行编码。编码器代表能够执行其中一个编码的硬件块。 连接器一次只能从一个编码器接收信号，每种类型的连接器只支持一些编码。还可能会有其他的物理限制，并不是每个CRTC连接到每个可用的编码器，限制了CRTC编码器连接器的可能组合。

Planes: plane 不是硬件块，而是包含供给扫描引擎（CRTC）的缓冲器的内存对象。 保存帧缓冲区的平面称为主平面，每个CRTC必须有一个关联的 plane ，因为它是 CRTC 确定参数的来源。参数包括，视频模式 - 显示分辨率（宽和高），像素大小，像素格式，刷新率等。


## 四、component 框架
RK 平台的 DRM 还依赖了 component 框架。
下面是内核邮件列表中关于 RK Socs DRM Driver Patch 的讨论：`https://lkml.org/lkml/2014/12/2/161`

在邮件开头我们可以看到 RK 平台 DRM Driver 诞生依赖 15 个版本中的主要变化。
其中有提到很重要的一点是其采用了 component 框架。

因为 DRM 下挂载了很多的设备，启动顺序可能会引发问题：

1. 驱动因为等待另一个资源的准备，产生 probe deferral 导致顺序不定。
2. 子设备没有加载好，主设备就加载了，导致设备无法工作。
3. 子设备相互之间可能有时序关系,不定的加载顺序,可能带来有些时候设备能工作,有些时候又不能工作。
4. 现在编 kernel 是多线程编译的,编译的前后顺序也会影响驱动的加载顺序。

对于 RK 平台因为有用到多个 VOP，需要在所有 VOP 都启动后再开启 DRM Driver 的 probe，即推迟 probe。

此时需要一个统一的管理机制，将所有设备统合起来按统一顺序进行加载，等所有组件加载完毕后，在进行他们和 master 的 bind。

更详细的关于介绍 component 的文章可以参考 component 作者与其他人讨论 component framework 的过程： `https://patchwork.kernel.org/patch/3431851/`

我们对 component 的接触会止步于 drm master probe 中的component 部分。我们将在下章分析 drm driver 代码中单独用一节分析在 rockchip drm master probe 中 component 的主要逻辑。

## 参考文章
```
wikipedia drm：https://en.wikipedia.org/wiki/Direct_Rendering_Manager
landley drm：http://www.landley.net/kdocs/htmldocs/drm.html
ubuntu drm kms：http://manpages.ubuntu.com/manpages/zesty/en/man7/drm-kms.7.html
https://lkml.org/lkml/2014/12/2/161
https://patchwork.kernel.org/patch/3431851/
```


