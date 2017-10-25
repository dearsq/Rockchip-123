# Linux PCIe Driver

## 基本概念

**Linux PCI 部分包括  PCI host driver 和 PCI device driver。**
PCI host driver 是内核自带的，已经实现好的。
PCI device driver 是我们需要完成/移植的，比如网卡驱动、PCIe转Sata小板驱动 等。

![](http://ww1.sinaimg.cn/large/ba061518gy1fknn7oqotjj20eu08v3yk.jpg)


## PCI 驱动框架

这里假设大家对 Linux 系统有一定程度的了解，比如字符设备、块设备、设备驱动程序的标准接口 `file_ops`、设备驱动程序的标准结构。
基于此来介绍一些 PCIe 设备的驱动框架。

