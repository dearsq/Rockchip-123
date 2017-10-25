# Android 磁盘管理系统

## 基本结构
由四大部分组成：
1. Linux kernel：用于检测热插拔事件；通过 uevent 向 Vold 的 NetlinkManager 发送 Uevent 事件;
2. Vold：全称 Volume Daemon。用于管理外部存储器设备的 Native 守护进程。Android 没有使用 Linux 平台下的 udev 来处理，于是 Google 写了一个类似 udev 功能的vold，充当了 kernel 与 framework 之间的桥梁；它由三部分组成 VolumeManager、NetlinnkManager、CommandListener。
3. Framework：Android 的核心框架，(仅仅磁盘管理这部分)负责操作 vold，给vold下发操作命令；
4. UI：Android 的系统应用，与  Framework 进行交互，用于挂载/卸载 SD Card / PCIe SATA。

这四个部分之间的相互联系均是使用  socket 进行通信，没有使用到传统的 API 调用，降低了系统各个模块之间的耦合性。

## 源码路径
**Vold:**system/vold
**Framework: **frameworks/base/services/java/com/android/server
**UI: **packages/apps/Settings/src/com/android/settings/deviceinfo/