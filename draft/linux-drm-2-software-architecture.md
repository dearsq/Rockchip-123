# Linux DRM (二) 软件架构

在《Linux DRM (一) Display Server 历史》我们了解了 DRM 诞生的历史缘由。
本篇我们再由浅入深，深入代码来看看 DRM 的软件架构。

## 使用 DRM 访问 Video Card
![](http://ww1.sinaimg.cn/large/ba061518gy1fke2korjhij20m80dd3z0.jpg)

> ![](http://ww1.sinaimg.cn/large/ba061518gy1fke2ld7wgoj20m80de0td.jpg)
图自 wikipedia

## DRM 软件架构
### 设备文件 cardX
DRM 处于内核空间，这意味着用户空间需要通过系统调用来申请它的服务。
不过 DRM 并没有定义它自己的系统调用。相反，它遵循“Everything is file”的原则，通过文件系统，在 `/dev/dri/` 目录下暴露了 GPU 的访问方式。
DRM 会检测每个 GPU，并生成对应的 DRM 设备，创建设备文件 `/dev/dri/cardX`与 GPU 相接。

用户空间的程序如果希望访问 GPU 则必须打开该文件，并使用 ioctl 与 DRM 通信。不同的 ioctl 对应 DRM API 的不同功能。

### DRM libdrm
libdrm 被创建以用于方便用户空间和 DRM 子系统的联系。它仅仅只提供了一些函数的包装（C)，这些函数是为 DRM API 的每一个 ioctl、常量、结构体 而写。
使用 libdrm 这个库不仅仅避免了将内核接口直接暴露给用户空间，也有代码复用等常见优点。

### DRM 代码结构
两个部分：通用的 DRM Core 和适配于不同类型硬件的 DRM Driver。
DRM Core 提供了不同 DRM 驱动程序可以注册的基本框架，
并且为用户空间提供了具有通用，独立于硬件功能的最小 ioctl 集合。
DRM Driver 实现了 API 的硬件依赖部分。
它提供了没有被 DRM core 覆盖的其余 ioctl 的实现，它也可以拓展 API，提供额外的 ioctl。比如某个特定的 DRM Driver 提供了一个增强的 API ，用户空间的 libdrm 也需要以额外的 libdrm-driver 拓展，用来使用这些额外的 ioctl。

### DRM API
DRM Core 向用户空间应用程序导出了多个接口，让相应的 libdrm 包装成函数后来使用。
DRM Driver 导出的特定设备的接口，可以通过 ioctls 和 sysfs 来供用户空间使用。

### DRM-Master 和 DRM-Auth
DRM API 中有几个 ioctl 由于并发问题仅限于用户空间的单个进程使用。
为了实现这种限制，将 DRM 设备分为 Master 和 Auth。
上述的那些 ioctl 只能被 DRM-Master 的进程调用。
打开了 `/dev/dri/cardX` 的进程的文件句柄会被标志为 master，特别是第一个掉哟该 `SET_MASTER` ioctl 的进程。如果不是 DRM-Master 的进程在使用这些限制 ioctl 的时候会返回错误。进程也可以通过 `DROP_MASTER` ioctl 放弃 Master 角色，来让其他进程变成 Master。

X Server 或者其他的 Display Server 通常会是他们所管理的 DRM 设备的 DRM-Master 进程。当 DRM 设备启动的时候哦，这些 Display Server 打开设备节点，获取 DRM-Master 权限，直到关闭设备。

对于其他的用户空间进程，还有一种办法可以获得 DRM 设备的这些 受限权限，这就是 DRM-Auth。它是一种针对 DRM 设备的验证方式，用来证明该进程已经获得了 DRM-Master 对于他们去访问受限 ioctls 的许可。

步骤：
1. DRM Client 使用 GET_MAGIC ioctl 从 DRM 设备获取一个 32bit整型的 token。并通过任何方式（通常是 IPC）传递给 DRM-Master。
2. DRM-Master 进程使用 AUTH_MAGIC ioctl 返回 token 给 DRM 设备。
3. 设备将 DRM-Master 所给的 token 和 Auth 的进行对比。通过的话就赋予进程文件句柄特殊的权限。

## DRM 中的特性

### GEM
Graphics Execution Manager
因为视频存储器大小增加，图形 API 比如 OpenGL 不断变复杂。在上下文切换时重新初始化显卡状态的话开销会相当大。另外，现在 Linux 桌面需要需要一个最佳的方式来和合成管理器共享屏幕外缓存。
这些需求导致需要开发新的方法来管理内核中的图形缓存区，即 GEM。

#### GEM 内存管理
GEM 提供了一套 API，其拥有明确的内存管理原语。通过 GEM，用户空间程序可以创建、处理、销毁 GPU 视频存储器中的内存对象。这些对象被统一称为 “GEM 对象”，从用户空间的角度来看，他们是一直存在的，所以在程序重新获得 GPU 的时候不需要重新加载他们。当用户空间的程序需要大量的视频内存空间（来存储 framebuffer、纹理数据 或是 GPU 所需要的其他数据），它都会使用 GEM API 来为 DRM Driver 请求分配内存。这套 API 还会提供填充 buffer 和 释放 的功能。并且可以在用户空间的进程（因意外或是正常终止而）关闭 DRM 设备描述符时释放相关内存。

GEM 也提供了两个及以上的用户空间进程访问同一个 DRM 设备(也就是共享同一个 GEM 对象)的方法。GEM 提供了一个方法，flink（其实这个方法很不安全），来从 GEM 句柄中获取 GEM 名字。进程通过 IPC 将 GEM 名字（32bit整型）传递给另一个进程。接收到 GEM 名字的进程于是可以获得一个本地的 GEM 句柄，用以指向原始的 GEM 对象。
所以如果有一个恶意的第三方应用知道了 GEM 名字，就可以访问和修改 GEM 对象的内容。所以后来通过引入 DMA-BUF 机制来克服这个缺陷。

#### GEM 内存同步
除了管理视频存储空间以外，视频内存管理器另一个重要的任务是处理 GPU 和 CPU 之间的内存同步（memory synchronization）问题。
因为现在的内存架构非常复杂，通常涉及了系统内存、视频内存的多级缓存（caches）。因此，视频内存管理器为了确保 GPU 和 CPU 共享数据的一致性还需要负责处理缓存的一致性。这意味着视频内存管理系统高度依赖于 GPU 硬件和年内存架构因此驱动具有特异性。

GEM 定义了用于内存同步的“内存域”，而这些内存域与GPU无关，它们采用 UMA 存储器架构设计，使其不再适用于其他内存架构，如具有单独 VRAM 的内存架构。因此，对外，DRM 驱动会向用户空间暴露统一的 GEM API，对内，驱动中会实现更适合特定硬件和内存架构的不同的内存管理器。

### KMS
Kernel Mode Setting

为了正常工作，视频卡和图形适配器必须负责设置一些模式，包括屏幕分辨率、色彩深度、刷新率等，将其设置为自己或是拓展屏所支持的参数范围内。这个动作就是 mode-setting，它通常需要访问图形硬件 —— 即去操作视频卡的寄存器 的能力。

在开始使用 framebuffer 之前，或者是由应用程序或者是用户改变 mode 时，进行 mode-setting。

#### UMS
最开始，用户空间想要使用图形的 framebuffer 也需要负责 mode-setting，因此他们需要访问 video 硬件的权限。类 Unix 系统中的 Display Server 比如 X Server 就是一个很好的例子，它的 mode-setting 被放在每种特定的图形卡各自的 DDX driver 中。

这种方法被称为 UMS（User space Mode-Setting），有几个严重的问题。它不仅打破了操作系统建立的硬件和程序的隔离，也让多进程同时进行 mode-setting 时，硬件会产生不一致的状态。
所以为了避免这些冲突，X Server 成为了实时 mode-settinng 的唯一用户空间程序，其他用户空间程序都依赖 X Server 来进行 mode-setting。
最初 mode-setting 是放在 X Server 的启动过程中，不过后来在其运行过程也可以 mode-setting 。

但是 UMS 有很多问题。
比如在 Linux 系统启动过程中，Linux 内核必须为 virtual console 设置 "minnimal text" mode。
另外，framebuffer driver 中也包含了配置 framebuffer 设备 mode 的代码。
为了解决这些冲突，有些 Display Server（比如 X Server）通过保存从图形环境切换到文本虚拟控制台时的 mode-setting 状态，并且在切换回去的时候恢复。
这又导致了新的问题，比如切换时的闪烁，输出设备的显示失败甚至损毁。


#### KMS 
为了解决这些问题，mode-setting 被单独放到了 Kernel 中，准确的说是 DRM 模块中。
然后，包括 X Server 在内的每个进程都可以命令内核来实现 mode-setting 操作，内核也会确保操作的一致性。这些加入 DRM 模块的新的内核 API 和 代码所执行的 mode-setting 的操作被称为 **KMS**。

它有太多好处了，第一个就是从 Kernel （console、fbdev）和 Userspace（X Server DDX Driver）干掉了那些重复的 mode-setting 的代码。第二就是对于图形操作系统不再需要关心 mode-setting 部分代码的编写了。第三就是因为提供了这种单一集中式的模式管理，console 和 X Server 不同实例的切换变得更加容易了。第四就是 mode-setting 放到内核后，可以从启动过程就开始使用它（这个在曾经也会导致闪烁问题）。

另外，因为 KMS 是内核的一部分，它也就可以去使用 kernel space 的很多资源（比如中断）。因为交由内核去管理了，在 suspend/resume 后的模式恢复变得更简单了。内核还让新显示设备的热插拔更加容易。

由于 mode-setting 和内存管理密切相关 ———— framebuffer 基本就是 内存buffer ———— 故它和图像内存管理紧密集成。这也是为什么它被放到 DRM 模块而不是独立作为一个子系统的原因。

#### KMS Driver
为了不破坏 DRM API 的向后兼容性，KMS 为 DRM Driver 提供了一个特殊的特性。任何 DRM Driver 在注册 DRM Core 的时候需要选择是否采用 DRIVER_MODESET 标志，用来表示是否支持 KMS API。
支持 KMS API 的驱动为了和传统的 DRM Driver 驱动区分往往被称为 KMS Driver。


KMS device model[edit]
KMS models and manages the output devices as a series of abstract hardware blocks commonly found on the display output pipeline of a display controller. These blocks are:[47]
CRTCs: each CRTC (from CRT Controller[48][33]) represents a scanout engine of the display controller, pointing to a scanout buffer (framebuffer).[47] The purpose of a CRTC is to read the pixel data currently in the scanout buffer and generate from it the video mode timing signal with the help of a PLL circuit.[49] The number of CRTCs available determines how many independent output devices can the hardware handle at the same time, so in order to use multi-head configurations at least one CRTC per display device is required.[47] Two —or more— CRTCs can also work in clone mode if they scan out from the same framebuffer to send the same image to several output devices.[49][48]
Connectors: a connector represents where the display controller sends the video signal from a scanout operation to be displayed. Usually, the KMS concept of a connector corresponds to a physical connector (VGA, DVI, FPD-Link, HDMI, DisplayPort, S-Video ...) in the hardware where an output device (monitor, laptop panel, ...) is permanently or can temporarily be attached. Information related to the current physically attached output device —such as connection status, EDID data, DPMS status or supported video modes— is also stored within the connector.[47]
Encoders: the display controller must encode the video mode timing signal from the CRTC using a format suitable for the intended connector.[47] An encoder represents the hardware block able to do one of these encodings. Examples of encodings —for digital outputs— are TMDS and LVDS; for analog outputs such as VGA and TV out, specific DAC blocks are generally used. A connector can only receive the signal from one encoder at a time,[47] and each type of connector only supports some encodings. There also might be additional physical restrictions by which not every CRTC is connected to every available encoder, limiting the possible combinations of CRTC-encoder-connector.
Planes: a plane is not a hardware block but a memory object containing a buffer from which a scanout engine (a CRTC) is fed. The plane that holds the framebuffer is called the primary plane, and each CRTC must have one associated,[47] since it's the source for the CRTC to determine the video mode —display resolution (width and height), pixel size, pixel format, refresh rate, etc.—. A CRTC might have also cursor planes associated to it if the display controller supports hardware cursor overlays, or secondary planes if it's able to scan out from additional hardware overlays and compose or blend "on the fly" the final image sent to the output device.[33]
Atomic Display[edit]
In recent years there has been an ongoing effort to bring atomicity to some regular operations pertaining the KMS API, specifically to the mode setting and page flipping operations.[33][50] This enhanced KMS API is what is called Atomic Display (formerly known as atomic mode-setting and atomic or nuclear pageflip).
The purpose of the atomic mode-setting is to ensure a correct change of mode in complex configurations with multiple restrictions, by avoiding intermediate steps which could lead to an inconsistent or invalid video state;[50] it also avoids risky video states when a failed mode-setting process has to be undone ("rollback").[51]:9 Atomic mode-setting allows to know beforehand if certain specific mode configuration is appropriate, by providing mode testing capabilities.[50] When an atomic mode is tested and its validity confirmed, it can be applied with a single indivisible (atomic) commit operation. Both test and commit operations are provided by the same new ioctl with different flags.
Atomic page flip on the other hand allows to update multiple planes on the same output (for instance the primary plane, the cursor plane and maybe some overlays or secondary planes) all synchronized within the same VBLANK interval, ensuring a proper display without tearing.[51]:9,14[50] This requirement is especially relevant to mobile and embedded display controllers, that tend to use multiple planes/overlays to save power.
The new atomic API is built upon the old KMS API. It uses the same model and objects (CRTCs, encoders, connectors, planes, ...), but with an increasing number of object properties that can be modified.[50] The atomic procedure is based on changing the relevant properties to build the state that we want to test or commit. The properties we want to modify depend on whether we want to do a mode-setting (mostly CRTCs, encoders and connectors properties) or page flipping (usually planes properties). The ioctl is the same for both cases, being the difference the list of properties passed with each one.[52]
Render nodes[edit]
In the original DRM API, the DRM device /dev/dri/cardX is used for both privileged (modesetting, other display control) and non-privileged (rendering, GPGPU compute) operations.[9] For security reasons, opening the associated DRM device file requires special privileges "equivalent to root-privileges".[53] This leads to an architecture where only some reliable user space programs (the X server, a graphical compositor, ...) have full access to the DRM API, including the privileged parts like the modeset API. The remainder user space applications that want to render or make GPGPU computations should be granted by the owner of the DRM device ("DRM Master") through the use of a special authentication interface.[54] Then the authenticated applications can render or make computations using a restricted version of the DRM API without privileged operations. This design imposes a severe constraint: there must always be a running graphics server (the X Server, a Wayland compositor, ...) acting as DRM-Master of a DRM device so that other user space programs can be granted to use the device, even in cases not involving any graphics display like GPGPU computations.[53][54]
The "render nodes" concept try to solve these scenarios by splitting the DRM user space API in two interfaces, one privileged and another non-privileged, and using separated device files (or "nodes") for each one.[9] For every GPU found, its corresponding DRM driver —if it supports the render nodes feature— creates a device file /dev/dri/renderDX, called the render node, in addition to the primary node /dev/dri/cardX.[54][9] Clients that use a direct rendering model and applications that want to take advantage of the computing facilities of a GPU, can do it without requiring additional privileges by simply opening any existing render node and dispatching GPU operations using the limited subset of the DRM API supported by those nodes —provided they have file system permissions to open the device file. Display servers, compositors and any other program that requires the modeset API or any other privileged operation must open the standard primary node that grants access to the full DRM API and use it as usual. Render nodes restricted API explicitly disallow the GEM flink operation to prevent buffer sharing using insecure GEM global names; only PRIME (DMA-BUF) file descriptors can be used to share buffers with another client, including the graphics server.[9][54]





