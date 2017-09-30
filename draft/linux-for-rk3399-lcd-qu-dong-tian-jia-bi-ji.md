# Linux for RK3399 LCD Porting

KernelVersion: 4.4.70

Documentation/devicetree/bindings/video/rockchip_fb.txt

## 概览

总的来说，RK Framebuffer 的 driver 有如下四个部分：
1. FB 框架相关的部分
2. LCDC 控制器相关的部分
3. LCD 屏幕相关的部分
4. LCD 电源操作的板级配置部分

```
➜  rockchip git:(master) ✗ tree ./driver/video/
.
├── backlight    背光相关
├── fbdev        FB 框架
│   └── core     FB 核心代码
│       ├── fbmem.c
│       └── fbsysfs.c
└── rockchip
    ├── rk_fb.c 
    ├── rkfb_sysfs.c
    ├── lcdc
    │   ├── rk322x_lcdc.c
    │   └── rk322x_lcdc.h
    ├── screen
    │   ├── lcd_general.c
    │   ├── lcd_mipi.c
    │   └── rk_screen.c
    └── transmitter
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
include/linux/rk_screen.h
```
fbmem.c 是 upstream 的代码。它的作用在于：向上提供了和用户空间交接的接口（open/read/write/ioctl）; 向下联系平台相关的 fb 驱动 rk_fb.c。

打开宏 `CONFIG_FB_ROCKCHIP`
```
obj-$(CONFIG_FB_ROCKCHIP) += rk_fb.o rkfb_sysfs.o bmp_helper.o screen/
obj-$(CONFIG_FB_ROCKCHIP) += display-sys.o lcdc/
```

会使能 rk framebuffer driver `kernel/driver/video/rockchip/rk_fb.c`

会使能 rk lcdc driver `kernel/driver/video/rockchip/lcdc/`

会使能 rk screen 解析屏幕配置相关代码
`kernel/driver/video/rockchip/screen/`

### framebuffer driver

```dts
fb: fb{
   compatible = "rockchip,rk-fb";
   rockchip,disp-mode = <DUAL>;
};
```

### LCDC 框架代码
这部分和具体的 LCDC 控制器相关，对于 RK3399 平台。
打开宏 `CONFIG_LCDC_RK322X`
```
obj-$(CONFIG_LCDC_RK322X) += rk322x_lcdc.o
```
### rk screen driver
依赖打开宏 `CONFIG_FB_ROCKCHIP` ，才会编译 `video/rockchip/screen` 中的内容。
`rk_screen.c` 
默认`LCD_GENERAL` 和 `CONFIG_LCD_MIPI` 二选一。
当不需要屏的时候选 `LCD_GENERAL`。
当需要屏的时候选`CONFIG_LCD_MIPI`。






### transmitter driver
打开宏 `CONFIG_RK_TRSM`
```
obj-$(CONFIG_RK_TRSM) += transmitter/
```

使能 rk transmitter driver
`kernel/drivers/video/rockchip/transmitter`

### rk mipi dsi driver
打开宏 `CONFIG_RK32_MIPI_DSI`对应驱动 `rk32_mipi_dsi.c`，Mipi driver 主文件。寄存器以及结构体的定义在 `rk32_mipi_dsi.h`。

打开宏 `CONFIG_MIPI_DSI` 对应驱动 `mipi_dsi.c`,封装的函数指针接口函数, 供 `lcd_mipi.c` 调用, 函数的具体实现在 `rk32_mipi_dsi.c` 中。Mipi 协议相关的宏定义以及函数指针结构体定义在 `mipi_dsi.h`。






drivers/video/rockchip/screen/
|_ lcd_mipi.c /* 屏参 dtsi 文件的解析 */

```

```




