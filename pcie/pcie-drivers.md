# Linux PCIe Driver

## 基本概念

**Linux PCI 部分包括  PCI host driver 和 PCI device driver。**
PCI host driver 是内核自带的，已经实现好的。
PCI device driver 是我们需要完成/移植的，比如网卡驱动、PCIe转Sata小板驱动 等。

![](http://ww1.sinaimg.cn/large/ba061518gy1fknn7oqotjj20eu08v3yk.jpg)


## PCI 驱动框架



## 参考文章
浅谈Linux PCI设备驱动（一）：http://blog.csdn.net/linuxdrivers/article/details/5849698

PCIe wikipedia：
https://en.wikipedia.org/wiki/PCI_Express

第1章 PCI总线的基本知识
http://blog.sina.com.cn/s/blog_6472c4cc0100qbvw.html

LDD 12.1.1 PCI 寻址
https://www.kancloud.cn/kancloud/ldd3/61020



Android6.0 MountService和vold详解（一）Mountservice的初始化
http://blog.csdn.net/kc58236582/article/details/50428741