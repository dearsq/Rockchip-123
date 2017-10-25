# mini PCIe 转 SATA 小板 (ASM1061) 调试

Author: Younix
Platform: RK3399
OS: Android 6.0
Kernel: 4.4
Version: v2017.04

## 一、PCI 设备调试手段
### busybox lspci
`lspci` 命令查看 pci 设备。出现如下信息：
```
shell@rk3399_mid:/ $ lspci
shell@rk3399_mid:/ $ 0c:00.0 0100: 1000:0056 (rev 02)
```
如果是 Android 设备没有移植 lspci 的可能需要通过 `busybox lspci` 实现。

0c：00.0 表示含义为 bus number： device number.function number 三者组合成一个 16bit 的识别码
1. bus number：8bits 最多连接到 256 个 bus
2. device number：6bits 最多连接到 32 种装置
3. function number：3bits 最多每种装置有 8 种功能
0100: 1000:0056 表示含义为 Class ID： Vendor ID  Device ID

### pci 设备速度
通过如下命令可以获取 pci 的速度
```
shell@rk3399_mid:/ $ lspci -n -d 1000:0056 -vvv | grep -i width
```

## 二、Sata 设备调试手段

### 查看分区信息
```
cat /proc/partitions
```

### mount
```
mount
umount
```

## 三、PCIe 转 Sata 小板调试步骤

### 3.1 打开 PCIe 设备驱动
1. 在 menuconfig 中
打开相应的调试宏：BUS Support -> PCI Debugging
打开相应 PCIe 总线驱动： BUS Support -> PCI Support
打开其热插拔功能（Hot Plug）：BUS Support -> Support for PCI Hotplug

2. 在 PCIe 设备没有插上的情况下开机，得到如下 log
```
[1.157185] rockchip-pcie f8000000.pcie: no vpcie3v3 regulator found
[1.157207] rockchip-pcie f8000000.pcie: no vpcie1v8 regulator found
[1.157223] rockchip-pcie f8000000.pcie: n vpcie0v9 regulator found
[1.691995] rockchip-pcie f8000000.pcie: PCIe link training gen1 timeout!
[1.692059] rockchip-pcie: probe of f8000000.pcie failed with error -110
```
我们 PCIe 设备还未连接，出现如上 Log 为正常。
3. 将 PCIe 的设备插在板子上后。利用 busybox lspci 查看现在的 pci 设备。
```
shell@rk3399_mid:/ $ busybox lspci
00:00.0 Class 0604: 17cd:0000
01:00.0 Class 0106: 1b21:0612
```
可以看到有几个 ID，可以根据 ID 确认设备是否被识别到。
比如我们根据官方的 datasheet 知道，1b21 即 ASMEDIA 厂商的 Vendor ID，0612 即 Device ID。

4. 之后就是加载设备驱动的时候，会根据 VENDOR_ID 进行匹配。识别成功后才能加载 probe。

5. 如果没有进入 probe ，有一种情况是设备已经被一个驱动占有了，找到这个设备使用的驱动，并且去除即可。

### 3.2 打开 SATA 设备驱动
对于调试转 SATA 设备，还需要提供设备驱动的支持：

打开 PCIe 转 SATA 小板的设备驱动：Device Driver -> Serial ATA and Parallel ATA driverrs(libata) -> AHCI SATA support

确认方法为
```shell
# ls dev/block/sd*
sda
sda1
```
可以看到多出来了 sda 与 sda1，sda 即为 sata 硬盘，sda[n] 即为其分区号。

### 3.3 手动挂载硬盘
在生成设备节点后将硬盘的设备节点挂载到我们系统的目录上。
首先查看产生的设备节点相关信息。
```
shell@rk3399_mid:/ $su
shell@rk3399_mid:/ $ls dev/block/sd*
shell@rk3399_mid:/ $sda sda1 sda2 sda3 
```
可以看到我的硬盘有三个分区。

接下来手动挂载。
```
shell@rk3399_mid:/ $mount  /dev/block/sdb1 /mnt/Younix/
```
正常情况下，这样操作便可以正常挂载。
但是也有可能会产生一些报错，在后面 第四节 常见调试问题 中，我们再来描述。

### 3.4 实现自动挂载
#### 3.4.1 修改 fstab
```diff
diff --git a/device/rockchip/rk3399/fstab.rk30board.bootmode.emmc b/device/rockchip/rk3399/fstab.rk30board.bootmode.emmc
index b2d4c75..059e73b 100755
--- a/device/rockchip/rk3399/fstab.rk30board.bootmode.emmc
+++ b/device/rockchip/rk3399/fstab.rk30board.bootmode.emmc
@@ -22,4 +22,7 @@
# for usb3.0
 /devices/platform/usb@*/*.dwc3*     auto vfat defaults      voldmanaged=usb:auto 
 
+# for pcie
+/devices/platform/*.pcie*           auto   vfat    defaults    voldmanaged=pcie:auto
+
 /dev/block/zram0                                none                swap      defaults                                              zramsize=533413200
```
#### 3.4.2 修改 VOLD 添加对 pcie 转 sata 设备的支持
```diff
diff --git a/system/vold/Disk.cpp b/system/vold/Disk.cpp
index 7daa9b7..97b7cd8 100755
--- a/system/vold/Disk.cpp
+++ b/system/vold/Disk.cpp
@@ -65,6 +65,7 @@ static const unsigned int kMajorBlockScsiN = 133;
 static const unsigned int kMajorBlockScsiO = 134;
 static const unsigned int kMajorBlockScsiP = 135;
 static const unsigned int kMajorBlockMmc = 179;
+static const unsigned int kMajorBlockPcie = 259;
 
 static const char* kGptBasicData = "EBD0A0A2-B9E5-4433-87C0-68B6B72699C7";
 static const char* kGptAndroidMeta = "19A710A2-B3CA-11E4-B026-10604B889DCF";
@@ -230,6 +231,16 @@ status_t Disk::readMetadata() {
         }
         break;
     }
+    case kMajorBlockPcie: {
+        std::string path(mSysPath + "/device/device/vendor");                                                                            
+        std::string tmp;
+        if (!ReadFileToString(path, &tmp)) {
+            PLOG(WARNING) << "Failed to read vendor from " << path;
+            return -errno;
+        }
+        mLabel = tmp;
+        break;
+    }
     default: {
         LOG(WARNING) << "Unsupported block major type" << major(mDevice);
         return -ENOTSUP;
@@ -301,6 +312,7 @@ status_t Disk::readPartitions() {
                 case 0x0b: // W95 FAT32 (LBA)
                 case 0x0c: // W95 FAT32 (LBA)
                 case 0x0e: // W95 FAT16 (LBA)
+		case 0x83: // W95 FAT16 (LBA)
                     createPublicVolume(partDevice);
 		    validParts = true;
                     break;
@@ -512,6 +524,14 @@ int Disk::getMaxMinors() {
         }
         return atoi(tmp.c_str());
     }
+    case kMajorBlockPcie: {
+        std::string tmp;
+        if (!ReadFileToString(kSysfsMmcMaxMinors, &tmp)) {
+            LOG(ERROR) << "Failed to read max minors";
+            return -errno;
+        }
+        return atoi(tmp.c_str());
+    }
     }
 
     LOG(ERROR) << "Unsupported block major type " << major(mDevice);
```

## 四、调试问题汇总

### PCIe 供电
PCIe 供电没有打开的情况下，需要在 dts 添加 power supply：
```
index 4763727..677ed9d 100644
--- a/arch/arm64/boot/dts/rockchip/rk3399-sapphire.dtsi
+++ b/arch/arm64/boot/dts/rockchip/rk3399-sapphire.dtsi
@@ -179,6 +179,17 @@
                rockchip,pwm_id= <2>;
                rockchip,pwm_voltage = <1000000>;
        };
+
+  vcc3v3_3g: vcc3v3-3g-regulator {
+    compatible = "regulator-fixed";
+    enable-active-high;
+    regulator-always-on;
+    regulator-boot-on;
+    gpio = <&gpio0 2 GPIO_ACTIVE_HIGH>;
+    pinctrl-names = "default";
+    pinctrl-0 = <&pcie_3g_drv>;
+    regulator-name = "vcc3v3_3g";
+  };
 };
 
 &cpu_l0 {
@@ -511,6 +522,7 @@
        num-lanes = <4>;
        pinctrl-names = "default";
        pinctrl-0 = <&pcie_clkreqn>;
+  phy-supply = <&vcc3v3_3g>;
        status = "okay";
 };
 
@@ -658,6 +670,14 @@
 };
 
 &pinctrl {
+
+  pcie {
+  pcie_3g_drv: pcie-3g-drv {
+    rockchip,pins =
+      <0 2 RK_FUNC_GPIO &pcfg_pull_up>;
+    };
+  };
+  

```

### mount 失败
我在手动挂载硬盘的时候，碰到了这样的问题。
```
[  167.457458] F2FS-fs (sda1): Magic Mismatch, valid(0xf2f52010) - read(0x0)
[  167.457483] F2FS-fs (sda1): Can't find valid F2FS filesystem in 2th superblock
```
这是因为默认对硬盘的文件系统不支持。
利用电脑尝试将硬盘格式化为 Fat 格式 或者 ext4 格式的系统后，重新 Mount ，问题解决。


