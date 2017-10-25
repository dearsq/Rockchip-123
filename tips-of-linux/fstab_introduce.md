# fstab 文件格式和解析代码分析


Author: Younix
Platform: RK3399
OS: Android 6.0
Kernel: 4.4
Version: v2017.04

## 一、格式说明
```
<src> <mount point>  <filesystem type> <mount flags parameters>     <fs_mgr_flags> 
/dev/.. /mnt/internal_sd  vfat               defaults         voldmanaged=internal_sd:14,nomulated
```


### 1.1 src 
表示 待挂载的设备节点路径

### 1.2 mount point 
表示 挂载点，即 被挂载的目录

### 1.3 filesystem type
表示 所挂载磁盘的文件系统类型

### 1.4 mount flags parameters 
表示 指定所挂载的文件系统的一些参数，如下 
async/sync : 设置是否为同步方式运行 
auto/noauto : 当下载mount -a 的命令时，此文件系统是否被主动挂载。默认为auto 
rw/ro : 是否以以只读或者读写模式挂载 
exec/noexec : 限制此文件系统内是否能够进行”执行”的操作 
user/nouser : 是否允许用户使用mount命令挂载 
suid/nosuid ： 是否允许SUID的存在 
usrquota : 启动文件系统支持磁盘配额模式 
grpquota : 启动文件系统对群组磁盘配额模式的支持 
defaults ： 同时具有rw,suid,dev,exec,auto,nouser,async等默认参数的设置


## 二、加载、解析、执行

### 2.1 从 init 开始
kernel 加载完后第一个执行的就是 init 进程，init 进程会根据 init.rc 规则启动进程或者服务。
对于我们 rk3399 的板子：
我采用的是 rk3399_mid 工程
```
./rk3399/rk3399_mid/init.rc:9:import /init.${ro.hardware}.rc

./common/init.rk30board.bootmode.emmc.rc:7:    mount_all fstab.rk30board
```
这个 `fstab.rk30board` 实际上是 `/fstab.rk30board.bootmode.emmc` 的软链接。

### 2.2 mount_all
`mount_all` 定义在 `system/core/init/keywords.h`
```
KEYWORD(mount_all,   COMMAND, 1, do_mount_all)
```
即会调用 `do_mount_all(fstab.rk30board) `

### 2.3 do_mount_all
代码在 `system/core/init/builtins.c`
```c
/*
 * This function might request a reboot, in which case it will
 * not return.
 */
int do_mount_all(int nargs, char **args)
{
    pid_t pid;
    int ret = -1;
    int child_ret = -1;
    int status;
    const char *prop;
    struct fstab *fstab;

    if (nargs != 2) {
        return -1;
    }

    /*
     * Call fs_mgr_mount_all() to mount all filesystems.  We fork(2) and
     * do the call in the child to provide protection to the main init
     * process if anything goes wrong (crash or memory leak), and wait for
     * the child to finish in the parent.
     */
    pid = fork();
    if (pid > 0) {
        /* Parent.  Wait for the child to return */
        int wp_ret = TEMP_FAILURE_RETRY(waitpid(pid, &status, 0));
        if (wp_ret < 0) {
            /* Unexpected error code. We will continue anyway. */
            NOTICE("waitpid failed rc=%d, errno=%d\n", wp_ret, errno);
        }

        if (WIFEXITED(status)) {
            ret = WEXITSTATUS(status);
        } else {
            ret = -1;
        }
    } else if (pid == 0) {
        /* child, call fs_mgr_mount_all() */
        klog_set_level(6);  /* So we can see what fs_mgr_mount_all() does */
        fstab = fs_mgr_read_fstab(args[1]);     //解析分区文件fstab
        child_ret = fs_mgr_mount_all(fstab);
        fs_mgr_free_fstab(fstab);
        if (child_ret == -1) {
            ERROR("fs_mgr_mount_all returned an error\n");
        }
        _exit(child_ret);
    } else {
        /* fork failed, return an error */
        return -1;
    }

    if (ret == FS_MGR_MNTALL_DEV_NEEDS_ENCRYPTION) {
        property_set("vold.decrypt", "trigger_encryption");
    } else if (ret == FS_MGR_MNTALL_DEV_MIGHT_BE_ENCRYPTED) {
        property_set("ro.crypto.state", "encrypted");
        property_set("vold.decrypt", "trigger_default_encryption");
    } else if (ret == FS_MGR_MNTALL_DEV_NOT_ENCRYPTED) {
        property_set("ro.crypto.state", "unencrypted");
        /* If fs_mgr determined this is an unencrypted device, then trigger
         * that action.
         */
        action_for_each_trigger("nonencrypted", action_add_queue_tail);
    } else if (ret == FS_MGR_MNTALL_DEV_NEEDS_RECOVERY) {
        /* Setup a wipe via recovery, and reboot into recovery */
        ERROR("fs_mgr_mount_all suggested recovery, so wiping data via recovery.\n");
        ret = wipe_data_via_recovery();
        /* If reboot worked, there is no return. */
    } else if (ret > 0) {
        ERROR("fs_mgr_mount_all returned unexpected error %d\n", ret);
    }
    /* else ... < 0: error */

    return ret;
}
```
我们看到，首先会通过 `fs_mgr_read_fstab` 读取/解析 
fstab文件 将其中的内容存在名为 fstab 的结构体中。


### 2.4 fs_mgr_read_fstab
./fs_mgr_fstab.c
```
        fstab->recs[cnt].fs_mgr_flags = parse_flags(p, fs_mgr_flags,
                                                    &flag_vals, NULL, 0);
        fstab->recs[cnt].key_loc = flag_vals.key_loc;
        fstab->recs[cnt].verity_loc = flag_vals.verity_loc;
        fstab->recs[cnt].length = flag_vals.part_length;
        fstab->recs[cnt].label = flag_vals.label;
        fstab->recs[cnt].partnum = flag_vals.partnum;
        fstab->recs[cnt].swap_prio = flag_vals.swap_prio;
        fstab->recs[cnt].zram_size = flag_vals.zram_size;
```

然后通过 `fs_mgr_mount_all` 对文件系统进行挂载。

### 2.5 fs_mgr_mount_all
其代码在 `.system/core/fs_mgr/fs_mgr.c` 这里不再进一步分析。





