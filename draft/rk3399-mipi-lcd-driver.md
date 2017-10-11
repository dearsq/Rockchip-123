# RK3399 Mipi LCD Driver 代码分析

KernelVersion: 4.4.70

Documentation/devicetree/bindings/video/rockchip_fb.txt

## 概览

总的来说，RK LCD 的 driver 有如下四个部分：
1. FB 框架相关的部分
2. LCDC 控制器相关的部分
3. LCD 屏幕配置相关的部分
4. Mipi 驱动代码

```
➜  rockchip git:(master) ✗ tree ./driver/video/
.
├── backlight    背光相关
├── fbdev        FB 框架
│   └── core     FB 核心代码
│       ├── fbmem.c
│       └── fbsysfs.c
└── rockchip
    ├── rk_fb.c  平台 FB 驱动
    ├── rkfb_sysfs.c
    ├── lcdc
    │   ├── rk322x_lcdc.c
    │   └── rk322x_lcdc.h
    ├── screen
    │   ├── lcd_general.c
    │   ├── lcd_mipi.c
    │   └── rk_screen.c  屏幕配置文件共用代码
    └── transmitter   Mipi 驱动代码
        ├── rk32_mipi_dsi.c
        ├── rk32_mipi_dsi.h
        ├── mipi_dsi.c
        └── mipi_dsi.h
```

![](http://ww1.sinaimg.cn/large/ba061518gy1fk1kul1dyaj20em0mg41r.jpg)

## RK FBDEV 框架相关代码
```
drivers/video/fbdev/core/fbmem.c
drivers/video/rockchip/rk_fb.c
drivers/video/rockchip/rkfb_sysfs.c
include/linux/rk_fb.h
```

`fbmem.c` 是 upstream 的代码。它的作用在于：向上提供了和用户空间交接的接口（open/read/write/ioctl）; 向下联系平台相关的 fb 驱动 rk_fb.c。

`rk_fb.c` 是 RK 平台的 FB 驱动。

`rkfb_sysfs.c` 是


当打开宏 `CONFIG_FB_ROCKCHIP`
```
obj-$(CONFIG_FB_ROCKCHIP) += rk_fb.o rkfb_sysfs.o bmp_helper.o screen/
obj-$(CONFIG_FB_ROCKCHIP) += display-sys.o lcdc/
```

会使能 rk framebuffer driver `kernel/driver/video/rockchip/rk_fb.c`

会使能 rk lcdc driver `kernel/driver/video/rockchip/lcdc/`

会使能 rk screen 解析屏幕配置相关代码
`kernel/driver/video/rockchip/screen/`


## framebuffer driver
代码路径 `kernel/driver/video/rockchip/rk_fb.c`
```dts
fb: fb{
   compatible = "rockchip,rk-fb";
   rockchip,disp-mode = <DUAL>;
};
```

## LCDC 框架代码
这部分和具体的 LCDC 控制器相关，对于 RK3399 平台。
打开宏 `CONFIG_LCDC_RK322X`
```
obj-$(CONFIG_LCDC_RK322X) += rk322x_lcdc.o
```

## RK Screen driver
依赖打开宏 `CONFIG_FB_ROCKCHIP` ，才会编译 `video/rockchip/screen` 中的内容。
`rk_screen.c` 
默认`LCD_GENERAL` 和 `CONFIG_LCD_MIPI` 二选一。
当不需要屏的时候选 `LCD_GENERAL`。
当需要屏的时候选`CONFIG_LCD_MIPI`。

### 对屏参文件的解析
屏相关的 dts 文件一般在 `kernel/arch/arm64/boot/dts/` 中。
分为四个部分，mipi host 配置、屏电源控制配置、屏初始化序列配置和屏参配置。
`drivers/video/rockchip/screen/lcd_mipi.c` 中负责解析 mipi host 配置、屏电源控制配置、屏初始化序列配置的解析。
`drivers/video/of_display_timing.c` 中负责解析 屏参。

#### Mipi Host 配置
我们直接看 dts 文件
```
```

#### 屏电源控制


#### 屏初始化序列

#### 屏参配置


## Transmitter driver
打开宏 `CONFIG_RK_TRSM`
```
obj-$(CONFIG_RK_TRSM) += transmitter/
```
使能 rk transmitter driver
`kernel/drivers/video/rockchip/transmitter`

### rk mipi dsi driver
打开宏 `CONFIG_RK32_MIPI_DSI`对应驱动 `rk32_mipi_dsi.c`，Mipi driver 主文件。寄存器以及结构体的定义在 `rk32_mipi_dsi.h`。

打开宏 `CONFIG_MIPI_DSI` 对应驱动 `mipi_dsi.c`,封装的函数指针接口函数, 供 `lcd_mipi.c` 调用, 函数的具体实现在 `rk32_mipi_dsi.c` 中。Mipi 协议相关的宏定义以及函数指针结构体定义在 `mipi_dsi.h`。




