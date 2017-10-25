# mount 用法小记

mount 命令实际上是调用 mount 系统调用。
用法：  
```
#include <sys/mount.h>
int mount(const char *source, const char *target,
   const char *filesystemtype, unsigned long mountflags, const void *data);
int umount(const char *target);
int umount2(const char *target, int flags);
```

参数：   
source：将要挂上的文件系统，通常是一个设备名。
target：文件系统所要挂在的目标目录。
filesystemtype：文件系统的类型，可以是"ext2"，"msdos"，"proc"，"nfs"，"iso9660" 。。。
mountflags：指定文件系统的读写访问标志，可能值有以下
MS_BIND：执行bind挂载，使文件或者子目录树在文件系统内的另一个点上可视。
MS_DIRSYNC：同步目录的更新。
MS_MANDLOCK：允许在文件上执行强制锁。
MS_MOVE：移动子目录树。
MS_NOATIME：不要更新文件上的访问时间。
MS_NODEV：不允许访问设备文件。
MS_NODIRATIME：不允许更新目录上的访问时间。
MS_NOEXEC：不允许在挂上的文件系统上执行程序。
MS_NOSUID：执行程序时，不遵照set-user-ID 和 set-group-ID位。
MS_RDONLY：指定文件系统为只读。
MS_REMOUNT：重新加载文件系统。这允许你改变现存文件系统的mountflag和数据，而无需使用先卸载，再挂上文件系统的方式。
MS_SYNCHRONOUS：同步文件的更新。
MNT_FORCE：强制卸载，即使文件系统处于忙状态。
MNT_EXPIRE：将挂载点标志为过时。
data：文件系统特有的参数。
    
返回说明：   
成功执行时，返回0。失败返回-1，errno被设为以下的某个值   
EACCES：权能不足，可能原因是，路径的一部分不可搜索，或者挂载只读的文件系统时，没有指定 MS_RDONLY 标志。
EAGAIN：成功地将不处于忙状态的文件系统标志为过时。
EBUSY：一. 源文件系统已被挂上。或者不可以以只读的方式重新挂载，因为它还拥有以写方式打开的文件。二. 目标处于忙状态。
EFAULT： 内存空间访问出错。
EINVAL：操作无效，可能是源文件系统超级块无效。
ELOOP ：路径解析的过程中存在太多的符号连接。
EMFILE：无需块设备要求的情况下，无用设备表已满。
ENAMETOOLONG：路径名超出可允许的长度。
ENODEV：内核不支持某中文件系统。
ENOENT：路径名部分内容表示的目录不存在。
ENOMEM： 核心内存不足。
ENOTBLK：source不是块设备。
ENOTDIR：路径名的部分内容不是目录。
EPERM ： 调用者权能不足。
ENXIO：块主设备号超出所允许的范围。


## 挂载系统
命令格式：mount [-t vfstype] [-o options] device dir 

1.-t vfstype 指定文件系统的类型，通常不必指定。mount 会自动选择正确的类型。常用类型有： 
光盘或光盘镜像：iso9660 
DOS fat16文件系统：msdos 
Windows 9x fat32文件系统：vfat 
Windows NT ntfs文件系统：ntfs 
Mount Windows文件网络共享：smbfs 
UNIX(LINUX) 文件网络共享：nfs 

2.-o options 主要用来描述设备或档案的挂接方式。常用的参数有： 
loop：用来把一个文件当成硬盘分区挂接上系统 
ro：采用只读方式挂接设备 
rw：采用读写方式挂接设备 
iocharset：指定访问文件系统所用字符集 



## 重新挂载系统
**Android 中碰到 Read Only 问题需要 remount 文件系统**

在终端下操作Android系统，我们访问系统分区的时候，经常遇到Only Read 的问题，此时需要以读写方式重新挂载需要操作的分区。

1、重新挂载根分区  
`mount -o remount  / `
2、以读写的模式重新挂载 根分区
`mount -o remount, rw /`
3、以不含suid的模式重新挂载根分区
`mount -o remount, nosuid / `
4、重新挂载 `system` 分区，需要读写`system/app/`目录下系统自带的 apk 安装包时使用。
`mount -o remount, system/`

## 挂载目录
与其说是挂载目录，不如说 “链接”，更容易理解。
但是偏偏这种情况无法使用 软链接，所以通过 `mount` 来达到类似于链接的目的。

为希望将 `/system/bin/` 目录通过 `ln -s` 挂载到 `/sdcard0` 下。
但是 system 分区是 ext4（基于内核的文件系统），sdcard 是 fuse 文件系统（基于用户空间），对于 Android，无法胯文件系统进行软链接（Linux 系统可以）。

此时可以采用 `mount -o bind`

1、挂载目录：把/system/test/ 挂载到/storage/sdcard0/Disk_Test目录下
```shell
mkdir /system/test
mkdir /storage/sdcard0/Disk_Test
touch /system/test/younix_mount_test

mount -o bind /system/test/ /storage/sdcard0/Disk_Test

ls /sotrage/sdcard0/Disk_Test/
younix_mount_test
``` 
或
```shell
busybox mount --bind  /system/test/ /storage/sdcard0/Disk_Test
```
2、取消挂载
```
umount /storage/sdcard0/Disk_Test
```

3、开机直接挂载
修改/etc/fstab文件
```
<file system> <mount point> <type> <options> <dump> <pass>  
/system/test/ /storage/sdcard0/Disk_Test none  rw,bind 0 0  
```

## 参考文章
linux mount (挂载命令)详解：http://tutu.spaces.eepw.com.cn/articles/article/item/70737