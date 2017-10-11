# Linux DRM (二) 软件架构

## DRM 
DRM 处于内核空间，这意味着用户空间需要通过系统调用来申请它的服务。
不过 DRM 并没有定义它自己的系统调用。相反，它遵循“Everything is file”的原则，通过文件系统，在 `/dev/dri/` 目录下暴露了 GPU 的访问方式。
DRM 会检测每个 GPU，并将他们称为 DRM 设备，并且会创建设备文件 `/dev/dri/cardX`与其相接。
用户空间的程序如果希望访问 GPU 则必须打开该文件，并使用 ioctl 与 DRM 通信。不同的 ioctl 对应 DRM API 的不同功能。

## DRM libdrm
libdrm 被创建以用于方便用户空间和 DRM 子系统的联系。它仅仅只提供了一些函数的包装（C)，这些函数是为 DRM API 的每一个 ioctl、常量、结构体 而写。
使用 libdrm 这个库不仅仅避免了将内核接口直接暴露给用户空间，也有代码复用等常见优点。


## DRM 的组成
两个部分：通用的 DRM core 和适配于不同类型硬件的 DRM Driver。
DRM Core 提供了不同 DRM 驱动程序可以注册的基本框架，
并且为用户空间提供了具有通用，独立于硬件功能的最小 ioctl 集合。
DRM Driver 实现了 API 与硬件（GPU）相关的部分。
它提供了没有被 DRM core 覆盖的其余 ioctl 的实现，它也可以拓展 API，提供额外的 ioctl。