# Linux 下的 DRM（一）Display Server 历史

## 一、Display Server

### X Windows 和 X Server
> The X Window System (X11, or shortened to simply X) is a windowing system for bitmap displays, common on UNIX-like computer operating systems.
X provides the basic framework for a GUI environment: drawing and moving windows on the display device and interacting with a mouse and keyboard. X does not mandate the user interface – this is handled by individual programs. As such, the visual styling of X-based environments varies greatly; different programs may present radically different interfaces

X Windows 是用于位图显示的窗口系统，常用于 UNIX 系的操作系统上。它为 GUI 环境提供了基本框架：在显示设备上绘制和移动窗口，并与鼠标和键盘进行交互。
它在设计之初，便秉承**“提供机制，而非策略”**的理念。（是的，和 Unix 设计哲学相同）所以它对用户接口不作要求，比如它提供了生成窗口（window）的方法，却对窗口呈现方式不作任何要求。
另一个设计特点是它是**基于 Client/Server 网络模型**。不论本地还是远程应用程序，都统一通过 C/S 模型来运作。

我们看一下 X Windows 的架构图（图来自 imtx.me 作者TualatriX）：

![](http://ww1.sinaimg.cn/large/ba061518gy1fkbr7xnevlj20et0e3dgk.jpg)

拿一个简单的应用场景举例。点击按钮引发按钮更新动作。
1. 内核捕获鼠标点击事件并发送给X server。
2. X Server 会计算该把这一事件发送给哪个窗口（事实上，窗口位置是由 Compositor 控制的，X Server 并不能够正确的计算 Compositor 做过特效变化之后的按钮的正确位置）。
3. 应用程序对此事件进行处理（将引发按钮更新动作）。但是，在此之前它得向X Server 发送绘制请求。
4. X Server 接收到这条绘制请求，然后把它发给视频驱动来渲染。X 还计算了更新区域，并且这条“垃圾信息”发送给了Compositor。
5. 这时，Compositor 知道它必须要重新合成屏幕上的一块区域。当然，这还是要向X Server发送绘制请求的。
6. 开始绘制。但是 X Server 还会去做一些不必要的本职工作（窗口重叠计算、窗口剪裁计算等）。

我们通过一个实例理解了上面的架构图。
它呈现出了一个缺点：
X Client --  X Server -- Compositor 
三者的交互比较耗时，不是很高效。除了交互方面的问题，其实 X Server 还会做一些重复无意义的工作，这里不再赘述。
有痛点就有解决方案，新一代的 Display Server 呼之欲出。
接下来 Wayland 诞生了。

### Wayland
Wayland is A Simple Display Server。
它在诞生之初的使命是用于改善 X Server 的不足并取代它。
但是现在看来它所作的不仅仅是替代 X Window 下的 X Server，也不仅仅是要取代 X Widnow。而是要颠覆 Linux 桌面上的 X Server/X Client 的概念。
以 **Compositor/Client 的结构**取而代之。
同时它复用了所有 Linux 内核图形和 I/O 技术：KMS、GEM、DRM、evdev。

其架构图如下（图来自 imtx.me 作者TualatriX）：

![](http://ww1.sinaimg.cn/large/ba061518ly1fkbs43oyw8j208v0dwgm4.jpg)

同样是上面那个实例，其流程如下：
1. 内核捕获鼠标点击事件并发送给Wayland Compositor。
2. 由于是直接发给Wayland Compositor的，所以Wayland Compositor会正确地计算出按钮的位置。同时它会把这一事件发送给按钮所在的应用程序来处理。
3. 应用程序直接渲染，无需向Wayland Compositor请求。只需在绘制完成之后向Wayland Compositor发送一条信息表明这块区域被更新了。
4. Wayland Compositor收到这条信息后，立即重新合成整个桌面。

所以基于 Wayland 的整个任务流程非常简单：
1. 基于Wayland协议，处理evdev的信息；
2. 通知Client（即应用程序）对相关事件做出反应（至于应用程序想怎么反应，Compositor不需要过问）；
3. 收到Client的状态更新，重新合成图形或管理新的图形布局。

简单的说，Waylannd 就是一个去除 X Window 中不必要的设计、充分利用现代 Linux 内核图形技术（DRM 子系统中的 KMS、GEM、DRM）的一个显示机制，它的出现是自然而然的，它的使命不是为了消灭X Window，而是将Linux的图形技术发挥至更高的一个境界。传统的X Window（即经典X应用、Gtk 1.x/2.x等旧应用），也会在相当长一段时间内得到继续支持，通过Wayland Client的形式跑在Wayland Compositor上，直到最终升级、取代或被淘汰。

有了以上背景，接下来我们便能更好的理解 DRM Subsystem 了。

## 二、DRM Subsystem
> In computing, the Direct Rendering Manager (DRM), a subsystem of the Linux kernel, interfaces with the GPUs of modern video cards. DRM exposes an API that user-space programs can use to send commands and data to the GPU, and to perform operations such as configuring the mode setting of the display. DRM was first developed as the kernel space component of the X Server's Direct Rendering Infrastructure,[1] but since then it has been used by other graphic stack alternatives such as Wayland.
User-space programs can use the DRM API to command the GPU to do hardware-accelerated 3D rendering and video decoding as well as GPGPU computing.
摘自 wipipedia

DRM，英文全称 Direct Rendering Manager, 即 直接渲染管理器。
它是为了解决多个程序对 Video Card 资源的协同使用问题而产生的。它向用户空间提供了一组 API，用以访问操纵 GPU。

### fbdev
Linux 早在很久以前就已经有一个名为 **fbdev** 的 API ，用于管理显卡的 framebuffer，但是它不能用于处理 基于视频卡的 GPU 的 3D 加速的需求。
这些 Video Card 常常需要设置或者管理 卡内存（Video RAM）中的一些命令队列。将命令分配给 GPU，还需要管理 Video RAM 本身的 Buffer 和 Free Space。

在最初的用户空间的程序（比如 X Server）可以直接管理这些资源，但这些程序通常表现的就仿佛他们是唯一去获取这些资源的一样。当有多个程序试图去以自己的方式同时控制 Video Card 资源时，就会崩溃。


### DRM
DRM 的诞生就是用来处理**多个程序对 Video Card 资源的协同使用问题**。
DRM 获得对 Video Card 的独占访问权限，它负责初始化和维护命令队列、Video RAM 以及其他相关的硬件资源。

要使用 GPU 的程序将请求发送给 DRM，由 DRM 作为仲裁来避免冲突。
DRM 到现在已经涵盖了以前由用户空间程序处理的很多功能，比如 帧缓存区的管理和模式设置，内存共享对象和内存同步。其中一些拓展具有特定的名称，比如图形执行管理器 GEM 或者内核模式设置 KMS，这些都是属于 DRM 子系统。
DRM 同时也负责处理 GPUs 切换的问题。

在下一章，我们会开始介绍 Linux 源码中 DRM 的软件架构。

参考文章：
https://en.wikipedia.org/wiki/X_Window_System
http://zqdevres.qiniucdn.com/data/20101106130912/index.html
https://imtx.me/archives/1573.html
https://imtx.me/archives/1574.html
https://en.wikipedia.org/wiki/Direct_Rendering_Manager