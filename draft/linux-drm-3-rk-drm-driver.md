# DRM 代码分析


### 2.1 设备文件 cardX
DRM 处于内核空间，这意味着用户空间需要通过系统调用来申请它的服务。
不过 DRM 并没有定义它自己的系统调用。相反，它遵循“Everything is file”的原则，通过文件系统，在 `/dev/dri/` 目录下暴露了 GPU 的访问方式。
DRM 会检测每个 GPU，并生成对应的 DRM 设备，创建设备文件 `/dev/dri/cardX`与 GPU 相接。X 为 0-15 的数值，默认是 Card0。

用户空间的程序如果希望访问 GPU 则必须打开该文件，并使用 ioctl 与 DRM 通信。不同的 ioctl 对应 DRM API 的不同功能。

### 2.2 用户空间的内存操作

我们定义一个外部的内存结构来更好的描述如何进行 userspace 的 drm 操作。
首先使用到 DRM 相关操作的时候需要引用 `drm.h`。

```
#include <drm.h>
struct bo {
   int fd;
   void *ptr;
   size_t size;
   size_t offset;
   size_t pitch;
   unsigned handle;
};
```
#### 2.2.1 获取设备节点
```
bo->fd = open("/dev/dri/card0"),O_RDWR,0);
```

#### 2.2.2 分配内存空间
```
struct drm_mode_create_dumb arg;
int handle, size, pitch;
int ret;
memset(&arg, 0, sizeof(arg));
arg.bpp = bpp;
arg.width = width;
arg.height = height;
ret = drmIoctl(bo->fd, DRM_IOCTL_MODE_CREATE_DUMB, &arg);
if (ret) {
    fprintf(stderr, "failed to create dumb buffer: %s\n", strerror(errno));
    return ret;
}
bo->handle = arg.handle;
bo->size = arg.size;
bo->pitch = arg.pitch;
```


### 2.2 DRM libdrm
libdrm 被创建以用于方便用户空间和 DRM 子系统的联系。它仅仅只提供了一些函数的包装（C)，这些函数是为 DRM API 的每一个 ioctl、常量、结构体 而写。
使用 libdrm 这个库不仅仅避免了将内核接口直接暴露给用户空间，也有代码复用等常见优点。

### 2.3 DRM 代码结构
分为两个部分：通用的 DRM Core 和适配于不同类型硬件的 DRM Driver。
DRM Core 提供了不同 DRM 驱动程序可以注册的基本框架，
并且为用户空间提供了具有通用，独立于硬件功能的最小 ioctl 集合。
DRM Driver 实现了 API 的硬件依赖部分。
它提供了没有被 DRM core 覆盖的其余 ioctl 的实现，它也可以拓展 API，提供额外的 ioctl。比如某个特定的 DRM Driver 提供了一个增强的 API ，用户空间的 libdrm 也需要以额外的 libdrm-driver 拓展，用来使用这些额外的 ioctl。

### 2.4 DRM API
DRM Core 向用户空间应用程序导出了多个接口，让相应的 libdrm 包装成函数后来使用。
DRM Driver 导出的特定设备的接口，可以通过 ioctls 和 sysfs 来供用户空间使用。

### 2.5 DRM-Master 和 DRM-Auth
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


##
DRM 在源码中的架构 (图自 Mark.Yao）：

![](https://markyzq.gitbooks.io/rockchip_drm_integration_helper/content/zh/picture/drm.png)



使用 DRM 访问 Video Card （图自 wikipedia)：
>![](http://ww1.sinaimg.cn/large/ba061518gy1fke2korjhij20m80dd3z0.jpg)
没有 DRM 时，用户空间进程访问 GPU 的方式
![](http://ww1.sinaimg.cn/large/ba061518gy1fke2ld7wgoj20m80de0td.jpg)
有 DRM 后，用户空间访问 GPU 的方式
