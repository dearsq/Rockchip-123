# Linux DRM（三）代码分析

本篇是 DRM 的第三篇文章。
在 《Linux DRM (一) Display Server》 中我们了解了 DRM  诞生的历史。
在 《Linux DRM (二) 基本概念和特性》 中我们了解了一些基本的概念。
现在，我们终于要向 DRM 源码进军了。

## 一、概览

不知大家是否还记得，之前我有引用 Wiki 中对 DRM 的介绍，这里我们再回顾一下：
DRM 由两个部分组成：
一是 Kernel 的子系统，这个子系统对硬件 GPU 操作进行了一层框架封装。
二是 提供了一个 libdrm 库，里面封装了一系列 API，用来进行图像显示。
整体来看和 Android 上所采用的 Direct Frame Buffer 差不多。
Android Kernel 走的是 FB 的框架，并在 HAL 抽象出一个 FBDEV，来进行 FB IOCTL 统一管理。
DRM 就相当于直接对图形设备集中处理，并且多出了一个 libdrm 库。

其整体脉络如下：

![](http://ww1.sinaimg.cn/large/ba061518gy1fl2l3bdc2kj20ib0fvjtt.jpg)

## 源码文件

![](http://ww1.sinaimg.cn/large/ba061518ly1fky0lv0ulnj20o70njaco.jpg)

## component framework
在讲述启动过程之前，先简单了解一下 component  framework。

因为 drm下挂了许多的设备, 启动顺序经常会引发各种问题:
1. 一个驱动完全有可能因为等另一个资源的准备, 而probe deferral, 导致顺序不定
2. 子设备没有加载好, 主设备就加载了, 导致设备无法工作
3. 子设备相互之间可能有时序关系,不定的加载顺序,可能带来有些时候设备能工作,有些时候又不能工作
4. 现在编kernel是多线程编译的,编译的前后顺序也会影响驱动的加载顺序.

这时就需要有一个统一管理的机制, 将所有设备统合起来, 按照一个统一的顺序加载,
Display-subsystem正是用来解决这个问题的, 依赖于component的驱动, 通过这个驱动,
可以把所有的设备以组件的形式加在一起, 等所有的组件加载完毕后, 统一进行bind/unbind.

代码路径 `drivers/base/component.c`

以下为rockchip drm master probe阶段component 主要逻辑, 为了减小篇幅, 去掉了无关的代码:
```
static int rockchip_drm_platform_probe(struct platform_device *pdev)
{
    for (i = 0;; i++) {
        /* ports指向了vop的设备节点 */
        port = of_parse_phandle(np, "ports", i);
        component_match_add(dev, &match, compare_of, port->parent);
    }
    for (i = 0;; i++) {
        port = of_parse_phandle(np, "ports", i);
        /* 搜查port下的各个endpoint, 将它们也加入到match列表 */
        rockchip_add_endpoints(dev, &match, port);
    }
    return component_master_add_with_match(dev, &rockchip_drm_ops, match);
}

static void rockchip_add_endpoints(...)
{
    for_each_child_of_node(port, ep) {
        remote = of_graph_get_remote_port_parent(ep);
            /* 这边的remote即为和vop关联的输出设备, 即为edp, mipi或hdmi */
        component_match_add(dev, match, compare_of, remote);
    }
}
```

## 启动过程
图自 markyzq gitbook:
![](http://ww1.sinaimg.cn/large/ba061518ly1fky11y98n8j20fo0krtao.jpg)

基于 component 框架，在 probe 阶段
1. 解析 dts 中各个设备的信息
2. 加到 component match 列表中
3. 设备加载完毕后，master 设备进行 bind


## RK DRM Device Driver
### device tree
```
display_subsystem: display-subsystem {
    compatible = "rockchip,display-subsystem";
    ports = <&vopl_out>, <&vopb_out>;
    status = "disabled";
};

- compatible: Should be "rockchip,display-subsystem"
- ports: Should contain a list of phandles pointing to display interface port
  of vop devices. vop definitions as defined in
  kernel/Documentation/devicetree/bindings/display/rockchip/rockchip-vop.txt

```
### drm driver
代码路径
```
drivers/gpu/drm/rockchip/rockchip_drm_drv.c
drivers/gpu/drm/rockchip/rockchip_drm_drv.h
```
```
static struct drm_driver rockchip_drm_driver = {
    .driver_features    = DRIVER_MODESET | DRIVER_GEM |
                  DRIVER_PRIME | DRIVER_ATOMIC |
                  DRIVER_RENDER,
    .preclose        = rockchip_drm_preclose,
    .lastclose        = rockchip_drm_lastclose,
    .get_vblank_counter    = drm_vblank_no_hw_counter,
    .open            = rockchip_drm_open,
    .postclose        = rockchip_drm_postclose,
    .enable_vblank        = rockchip_drm_crtc_enable_vblank,
    .disable_vblank        = rockchip_drm_crtc_disable_vblank,
    .gem_vm_ops        = &rockchip_drm_vm_ops,
    .gem_free_object    = rockchip_gem_free_object,
    .dumb_create        = rockchip_gem_dumb_create,
    .dumb_map_offset    = rockchip_gem_dumb_map_offset,
    .dumb_destroy        = drm_gem_dumb_destroy,
    .prime_handle_to_fd    = drm_gem_prime_handle_to_fd,
    .prime_fd_to_handle    = drm_gem_prime_fd_to_handle,
    .gem_prime_import    = drm_gem_prime_import,
    .gem_prime_export    = drm_gem_prime_export,
    .gem_prime_get_sg_table    = rockchip_gem_prime_get_sg_table,
    .gem_prime_import_sg_table    = rockchip_gem_prime_import_sg_table,
    .gem_prime_vmap        = rockchip_gem_prime_vmap,
    .gem_prime_vunmap    = rockchip_gem_prime_vunmap,
    .gem_prime_mmap        = rockchip_gem_mmap_buf,
#ifdef CONFIG_DEBUG_FS
    .debugfs_init        = rockchip_drm_debugfs_init,
    .debugfs_cleanup    = rockchip_drm_debugfs_cleanup,
#endif
    .ioctls            = rockchip_ioctls,
    .num_ioctls        = ARRAY_SIZE(rockchip_ioctls),
    .fops            = &rockchip_drm_driver_fops,
    .name    = DRIVER_NAME,
    .desc    = DRIVER_DESC,
    .date    = DRIVER_DATE,
    .major    = DRIVER_MAJOR,
    .minor    = DRIVER_MINOR,
};
```

### vop driver
代码路径：
```
drivers/gpu/drm/rockchip/rockchip_drm_vop.c
drivers/gpu/drm/rockchip/rockchip_vop_reg.c
```
结构体：
```
struct vop;
// vop 驱动根结构, 一个vop对应一个struct vop结构

struct vop_win;
// 描述图层信息, 一个硬件图层对应一个struct vop_win结构
```

寄存器读写：
为了兼容各种不同版本的vop, vop驱动里面使用了寄存器级的抽象, 由一个结构体来保存抽象关系, 这样主体逻辑只需要操作抽象后的功能定义, 由抽象的读写接口根据抽象关系写到真实的vop硬件中.

示例:
```
  static const struct vop_win_phy rk3288_win23_data = {
       .enable = VOP_REG(RK3288_WIN2_CTRL0, 0x1, 4),
  }
  static const struct vop_win_phy rk3368_win23_data = {
       .enable = VOP_REG(RK3368_WIN2_CTRL0, 0x1, 4),
  }
```
rk3368和rk3288图层的地址分布不同, 但在结构定义的时候, 可以将不同的硬件图层bit
映射到同一个enable功能上, 这样vop驱动主体调用`VOP_WIN_SET(vop, win, enable, 1);`
时就能操作到真实的vop寄存器了.




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

#### 2.2.3 映射物理内存
```
struct drm_mode_map_dumb arg;
void *map;
int ret;

memset(&arg, 0, sizeof(arg));
arg.handle = bo->handle;
ret = drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &arg);
if (ret)
    return ret;
map = drm_mmap(0, bo->size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, arg.offset);
if (map == MAP_FAILED)
   return -EINVAL;
bo->ptr = map
```

#### 2.2.4 解除物理内存映射
```
drm_munmap(bo->ptr, bo->size);
bo->ptr = NULL;
```

#### 2.2.5 释放内存
```
struct drm_mode_destroy_dumb arg;
int ret;

memset(&arg, 0, sizeof(arg));
arg.handle = bo->handle;
ret = drmIoctl(bo->fd, DRM_IOCTL_MODE_DESTROY_DUMB, &arg);
if (ret)
    fprintf(stderr, "failed to destroy dumb buffer: %s\n", strerror(errno));
```

#### 2.2.6 释放 gem handle
```
struct drm_gem_close args;
memset(&args, 0, sizeof(args));
args.handle = bo->handle;
drmIoctl(bo->fd, DRM_IOCTL_GEM_CLOSE, &args);
```

#### 2.2.7 export dmafd
```
int export_dmafd;
ret = drmPrimeHandleToFD(bo->fd, bo->handle, 0, &export_dmafd);
// drmPrimeHandleToFD是会给dma_buf加引用计数的
// 使用完export_dmafd后, 需要使用  close(export_dmafd)来减掉引用计数
```

#### 2.2.8 import dmafd
```
ret = drmPrimeFDToHandle(bo->fd, import_dmafd, &bo->handle);
// drmPrimeHandleToFD是会给dma_buf加引用计数的
// 使用完bo->handle后, 需要对handle减引用, 参看free gem handle部分.
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
