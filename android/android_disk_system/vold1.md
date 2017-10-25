# Vold 1

## 源码结构

```
.
├── Android.mk
├── Asec.h
├── bench
│   └── benchgen.py
├── Benchmark.cpp
├── BenchmarkGen.h
├── Benchmark.h
├── CheckBattery.cpp
├── CheckBattery.h
├── CleanSpec.mk
├── CommandListener.cpp
├── CommandListener.h
├── CryptCommandListener.cpp
├── CryptCommandListener.h
├── cryptfs.c
├── cryptfs.h
├── Devmapper.cpp
├── Devmapper.h
├── Disk.cpp
├── Disk.h
├── EmulatedVolume.cpp
├── EmulatedVolume.h
├── Ext4Crypt.cpp
├── Ext4Crypt.h
├── fs
│   ├── Ext4.cpp
│   ├── Ext4.h
│   ├── F2fs.cpp
│   ├── F2fs.h
│   ├── Ntfs.cpp
│   ├── Ntfs.h
│   ├── Vfat.cpp
│   └── Vfat.h
├── hash.h
├── Loop.cpp
├── Loop.h
├── main.cpp
├── MoveTask.cpp
├── MoveTask.h
├── NetlinkHandler.cpp
├── NetlinkHandler.h
├── NetlinkManager.cpp
├── NetlinkManager.h
├── PrivateVolume.cpp
├── PrivateVolume.h
├── Process.cpp
├── Process.h
├── PublicVolume.cpp
├── PublicVolume.h
├── ResponseCode.cpp
├── ResponseCode.h
├── secdiscard.cpp
├── sehandle.h
├── tests
│   ├── Android.mk
│   └── VolumeManager_test.cpp
├── TrimTask.cpp
├── TrimTask.h
├── Utils.cpp
├── Utils.h
├── vdc.c
├── VoldCommand.cpp
├── VoldCommand.h
├── VoldUtil.c
├── VoldUtil.h
├── VolumeBase.cpp
├── VolumeBase.h
├── VolumeManager.cpp
└── VolumeManager.h

```

## 关键类

VolumeBase 类：描述的是具体的磁盘设备。
PrivateVolume/PublicVolume： 继承自 VolumeBase。其中保存着描述磁盘的信息以及操作磁盘的函数。

VolumeManager: 用于管理 Volume 类;
NetlinkManager: 用于与内核 uevent 事件通信，接受后转发给 VolumeManager 。通信会用到 NetlinkListener 和 SocketListener 类的函数。

CommandListener：接受来自 VolumeManager 的事件，通过 socket 通信方式发送给 MountService。

MountService：接受来自 CommandListener 事件。Android Binder Service，运行在 system_server 进程，用于和 Vold 消息进行通信，比如 MountService 向 Vold 发送挂载 SDCard 的命令; 接受来自 Vold 的外设热插拔的事件。

## main.cpp 
我们从 main.cpp 入口开始看。

```cpp
int main(int argc, char** argv) {
    /********************************************************************************** 
    **以下四个类声明四个指针对象： 
    **VolumeManager :管理所有存储设备(volume对象)； 
    **CommandListener:监听Framework下发的消息，并分析命令，调用响应的操作函数； 
    **CryptCommandListener：
    **NetlinkManager :监听Linux内核的热插拔事件，uevent事件 
    **********************************************************************************/  
    VolumeManager *vm;
    CommandListener *cl;
    CryptCommandListener *ccl;
    NetlinkManager *nm;

    parse_args(argc, argv);

    sehandle = selinux_android_file_context_handle();
    if (sehandle) {
        selinux_android_set_sehandle(sehandle);
    }

    // Quickly throw a CLOEXEC on the socket we just inherited from init
    fcntl(android_get_control_socket("vold"), F_SETFD, FD_CLOEXEC);
    fcntl(android_get_control_socket("cryptd"), F_SETFD, FD_CLOEXEC);

    mkdir("/dev/block/vold", 0755);

    /* For when cryptfs checks and mounts an encrypted filesystem */
    klog_set_level(6);

    /* Create our singleton managers */
    if (!(vm = VolumeManager::Instance())) {
        LOG(ERROR) << "Unable to create VolumeManager";
        exit(1);
    }

    if (!(nm = NetlinkManager::Instance())) {
        LOG(ERROR) << "Unable to create NetlinkManager";
        exit(1);
    }

    if (property_get_bool("vold.debug", false)) {
        vm->setDebug(true);
    }

    cl = new CommandListener();
    ccl = new CryptCommandListener();
    vm->setBroadcaster((SocketListener *) cl);
    nm->setBroadcaster((SocketListener *) cl);

    if (vm->start()) {
        PLOG(ERROR) << "Unable to start VolumeManager";
        exit(1);
    }

    if (process_config(vm)) {
        PLOG(ERROR) << "Error reading configuration... continuing anyways";
    }

    if (nm->start()) {
        PLOG(ERROR) << "Unable to start NetlinkManager";
        exit(1);
    }

    coldboot("/sys/block");
//    coldboot("/sys/class/switch");

    /*
     * Now that we're up, we can respond to commands
     */
    if (cl->startListener()) {
        PLOG(ERROR) << "Unable to start CommandListener";
        exit(1);
    }

    if (ccl->startListener()) {
        PLOG(ERROR) << "Unable to start CryptCommandListener";
        exit(1);
    }

    // Eventually we'll become the monitoring thread
    while(1) {
        sleep(1000);
    }

    LOG(ERROR) << "Vold exiting";
    exit(0);
}

```























