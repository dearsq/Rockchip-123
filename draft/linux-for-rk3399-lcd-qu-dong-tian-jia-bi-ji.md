Documentation/devicetree/bindings/video/rockchip_fb.txt



### framebuffer driver 
打开宏 `CONFIG_FB_ROCKCHIP`
```
obj-$(CONFIG_FB_ROCKCHIP) += rk_fb.o rkfb_sysfs.o bmp_helper.o screen/
obj-$(CONFIG_FB_ROCKCHIP) += display-sys.o lcdc/
```

使能 rk framebuffer driver 驱动 `kernel/driver/video/rockchip/rk_fb.c`

使能 


```dts
fb: fb{
   compatible = "rockchip,rk-fb";
   rockchip,disp-mode = <DUAL>;
      };
```

### transmitter driver
打开宏 `CONFIG_RK_TRSM`
```
obj-$(CONFIG_RK_TRSM) += transmitter/
```

使能 rk transmitter driver
`kernel/drivers/video/rockchip/transmitter`

### rk mipi dsi driver
打开宏 `CONFIG_MIPI_DSI` 对应驱动 `mipi_dsi.c`,封装的函数指针接口函数, 供 `lcd_mipi.c` 调用, 函数的具体实现在 `rk32_mipi_dsi.c` 中。

打开宏 `CONFIG_RK32_MIPI_DSI`对应驱动 `rk32_mipi_dsi.c`，Mipi driver 主文件。

```

```