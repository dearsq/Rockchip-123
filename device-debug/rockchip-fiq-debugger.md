# RK FIQ 快速调试模式

## 进入 FIQ 模式
串口中快速输入 `fiq`
```
root@linaro-alip:~# fi
Welcome to irq debugger mode
Enter ? to get command help
debug> 
debug> 
debug> 
```

## help 查看用法
```
debug> help
FIQ Debugger commands:
 pc            PC status
 regs          Register dump
 allregs       Extended Register dump
 bt            Stack trace
 reboot [<c>]  Reboot with command <c>
 reset [<c>]   Hard reset with command <c>
 irqs          Interupt status
 kmsg          Kernel log
 version       Kernel version
 sleep         Allow sleep while in FIQ
 nosleep       Disable sleep while in FIQ
 console       Switch terminal to console
 cpu           Current CPU
 cpu <number>  Switch to CPU<number>
 ps            Process list
 sysrq         sysrq options
 sysrq <param> Execute sysrq with <param>
```

## pc 查看机器状态
```
debug> pc   
 pc ffffff8008899cc0 cpsr 60000145 mode EL1h
```

## reg allregs 查看当前寄存器的状态
dump 出寄存器的状态。
```
debug> regs
  x0 0000000000000000   x1 0000000000000000
  x2 ffffff80088c00b8   x3 0000000000000020
  x4 0006acfc8ea0b200   x5 00000001b9aca940
  x6 00000000000121a2   x7 0000000000000000
  x8 00000032b5193519   x9 ffffff8008081000
 x10 0000000000001000  x11 0000000000000000
 x12 0000000034d5d91d  x13 0000000000000000
 x14 0000000000000000  x15 0000000000000000
 x16 0000000000000000  x17 0000000000000000
 x18 0000000030d00800  x19 000000464d89ac22
 x20 0000000000000002  x21 ffffffc0786ac400
 x22 ffffff8009285f18  x23 0000000000000002
 x24 0000000000000004  x25 000000464cba669c
 x26 ffffff80092216c8  x27 0000000002bdc1f0
 x28 000000000301001c  x29 ffffff8009133e90
 x30 ffffff8008899cbc   sp ffffff8009133e90
  pc ffffff8008899cc0 cpsr 60000145 (EL1h)
  
 debug> allregs
  x0 0000000000000000   x1 0000000000000000
  x2 ffffff80088c00b8   x3 0000000000000020
  x4 002339f25bbb10c0   x5 00000001e0fbc580
  x6 0000000000013542   x7 0000000000000000
  x8 00000032b5193519   x9 ffffff8008081000
 x10 0000000000001000  x11 0000000000000000
 x12 0000000034d5d91d  x13 0000000000000000
 x14 0000000000000000  x15 0000000000000000
 x16 0000000000000000  x17 0000000000000000
 x18 0000000030d00800  x19 0000004cb30898aa
 x20 0000000000000002  x21 ffffffc0786ac400
 x22 ffffff8009285f18  x23 0000000000000002
 x24 0000000000000004  x25 0000004cb298efb4
 x26 ffffff80092216c8  x27 0000000002bdc1f0
 x28 000000000301001c  x29 ffffff8009133e90
 x30 ffffff8008899cbc   sp ffffff8009133e90
  pc ffffff8008899cc0 cpsr 60000145 (EL1h)
 sp_el0   ffffff8009130000
 elr_el1  ffffff8008899cc0
 spsr_el1 60000145
```
## bt 用来跟踪栈信息
```
debug> bt
pid: 0  comm: swapper/0
  x0 0000000000000000   x1 0000000000000000
  x2 ffffff80088c00b8   x3 0000000000000020
  x4 001c4fed878b5fc0   x5 00000002646cb280
  x6 0000000000017772   x7 0000000000000000
  x8 00000032b5193519   x9 ffffff8008081000
 x10 0000000000001000  x11 0000000000000000
 x12 0000000034d5d91d  x13 0000000000000000
 x14 0000000000000000  x15 0000000000000000
 x16 0000000000000000  x17 0000000000000000
 x18 0000000030d00800  x19 0000006218182cfe
 x20 0000000000000002  x21 ffffffc0786ac400
 x22 ffffff8009285f18  x23 0000000000000002
 x24 0000000000000004  x25 000000621751124a
 x26 ffffff80092216c8  x27 0000000002bdc1f0
 x28 000000000301001c  x29 ffffff8009133e90
 x30 ffffff8008899cbc   sp ffffff8009133e90
  pc ffffff8008899cc0 cpsr 60000145 (EL1h)

cpuidle_enter_state+0x1a4/0x244:
  pc ffffff8008899cc0   sp ffffff8009133e90   fp ffffff8009133e90
cpuidle_enter+0x34/0x44:
  pc ffffff8008899dd4   sp ffffff8009133ea0   fp ffffff8009133ee0
call_cpuidle+0x60/0x70:
  pc ffffff80080db33c   sp ffffff8009133ef0   fp ffffff8009133f10
cpu_startup_entry+0x22c/0x2b8:
  pc ffffff80080db578   sp ffffff8009133f20   fp ffffff8009133f40
rest_init+0x78/0x80:
  pc ffffff8008bd6df0   sp ffffff8009133f50   fp ffffff8009133f90
start_kernel+0x3d0/0x3e4:
  pc ffffff8009010bc0   sp ffffff8009133fa0   fp ffffff8009133fa0
__primary_switched+0x30/0x6c:
  pc ffffff80090101c4   sp ffffff8009133fb0   fp 0000000000000000
```

## reboot reset 重启
`reboot [<c>]` 软件重启
`reset [<c>]` 硬件重启
可以不加 `[<c>]` 选项，也可以加上 loader 选项用来进入烧录模式。

## irqs 列出所有中断的状态
```
debug> irqs
irqnr       total  since-last   status  name
    8:          0           0    31e00  ???
   14:          0           0    31600  arch_timer
   15:      57646       57646    31600  arch_timer
   17:       7585        7585      104  rk_timer
   20:          0           0      104  ff6d0000.dma-controller
   21:        865         865      104  ff6d0000.dma-controller
   22:          0           0      104  ff6e0000.dma-controller
   23:          0           0      104  ff6e0000.dma-controller
   24:          0           0      104  eth0
   25:       7583        7583      104  dw-mci
   26:          0           0      104  dw-mci
   27:      10129       10129      104  mmc1
   28:          0           0      104  ehci_hcd:usb5
   29:          0           0      104  ohci_hcd:usb6
   30:          0           0      104  ehci_hcd:usb1
   31:          0           0      104  ohci_hcd:usb2
   32:        837         837      104  ff100000.saradc
   33:       1140        1140      104  ff3c0000.i2c
   34:        438         438      104  ff110000.i2c
   36:         22          22      104  debug
   37:          0           0      104  rockchip_thermal
   38:         99          99      104  ff3d0000.i2c
   40:          0           0    10d04  ???
   42:          1           1      104  rk_pwm_irq
   43:          0           0      104  ff650000.vpu_service
   44:          0           0      104  ff650000.vpu_service
   46:          0           0      104  ff660000.rkvdec
   52:       1299        1299      104  ff9a0000.gpu
   53:        953         953      104  ff9a0000.gpu
   54:          0           0      104  ff9a0000.gpu
   55:          0           0      104  ff8f0000.vop
   56:        726         726      504  ff900000.vop
   57:          0           0      104  ff940000.hdmi
   59:          0           0    10d04  ???
   60:          0           0    10d04  ???
   61:          0           0    10d04  ???
   62:          0           0    10d04  ???
   63:          0           0    10d04  ???
   67:        180         180      504  bcmsdh_sdmmc
   68:          1           1        1  bt_default_wake_host_irq
   69:          1           1        3  GPIO Key Power
   77:          0           0        3  hall_mh248
   98:          1           1      508  fusb302
  117:          0           0      108  rk808
  224:          0           0      104  rockchip_usb2phy
  225:          0           0      104  rockchip_usb2phy
  227:          0           0      104  xhci-hcd:usb3
  233:          0           0     8400  RTC alarm
  237:          0           0      104  rockchip_usb2phy
  238:          0           0      104  rockchip_usb2phy_bvalid
  239:          0           0      104  rockchip_usb2phy

```


## kmsg 打印 Kernel Log

## version 查看 versionn 版本
```
debug> version
Linux version 4.4.70 (younix@YounixPC) (gcc version 5.4.0 20160609 (Ubuntu/Linaro 5.4.0-6ubuntu1~16.04.4)7
```
## sleep nosleep 允许/禁止睡眠

## console 回到 Console 模式

```
debug> console
console mode

root@linaro-alip:~# 
root@linaro-alip:~# 
root@linaro-alip:~# 
```

## cpu 
`cpu` 显示当前使用的 cpu
`cpu <number>` 切换到第 number 个 cpu 使用。

## ps 
打印所有的进程信息
```
debug> ps 
pid   ppid  prio task            pc
    1     0  120 systemd       S ffffff8008084f28
    2     0  120 kthreadd      S ffffff8008084f28
    3     2  120 ksoftirqd/0   S ffffff8008084f28
    4     2  120 kworker/0:0   S ffffff8008084f28
    5     2  100 kworker/0:0H  S ffffff8008084f28
    6     2  120 kworker/u12:0 S ffffff8008084f28
    7     2  120 rcu_sched     S ffffff8008084f28
    8     2  120 rcu_bh        S ffffff8008084f28
    9     2    0 migration/0   S ffffff8008084f28
   10     2    0 watchdog/0    S ffffff8008084f28
   11     2    0 watchdog/1    S ffffff8008084f28
   12     2    0 migration/1   S ffffff8008084f28

```

## sysrq 
`sysrq` 显示其支持的选项
很强大的命令。
```
debug> sysrq
[ 1027.452877] sysrq: SysRq : HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) memory-full-oom-kill(f) kill-all-tasks(i) thaw-filesystems(j) sak(k) show-backtrace-all-active-cpus(l) show-memory-usage(m) nice-all-RT-tasks(n) poweroff(o) show-registers(p) show-all-timers(q) unraw(r) sync(s) show-task-states(t) unmount(u) force-fb(V) show-blocked-tasks(w) dump-ftrace-buffer(z)  
debug> 

```
### HELP 显示支持的命令

### 调整 loglevel 
选项 数字 0-9

### b 重启

### c 直接崩溃
```
debug> sysrq c
[  110.195819] sysrq: SysRq : Trigger a crash
[  110.196226] Unable to handle kernel NULL pointer dereference at virtual address 00000000
[  110.196945] pgd = ffffff8009355000
[  110.197253] [00000000] *pgd=000000007fffe003, *pud=000000007fffe003, *pmd=0000000000000000
[  110.198007] Internal error: Oops: 96000045 [#1] SMP
[  110.198443] Modules linked in:
[  110.198734] CPU: 0 PID: 0 Comm: swapper/0 Not tainted 4.4.70 #49
[  110.199266] Hardware name: Rockchip RK3399 Excavator Board (Linux Opensource) (DT)
[  110.199938] task: ffffff8009147190 ti: ffffff8009130000 task.ti: ffffff8009130000
[  110.200614] PC is at sysrq_handle_crash+0x24/0x30
[  110.201041] LR is at __handle_sysrq+0xa4/0x150
[  110.201442] pc : [<ffffff8008425f64>] lr : [<ffffff80084269e0>] pstate: 200001c5
[  110.202095] sp : ffffffc07ff01d60
[  110.202395] x29: ffffffc07ff01d60 x28: ffffff8009130000 
[  110.202880] x27: ffffff8009130000 x26: 0000000000000000 
[  110.203364] x25: ffffff8009133d60 x24: 0000000000000000 
[  110.203847] x23: 0000000000000063 x22: 0000000000000007 
[  110.204331] x21: ffffff8009155000 x20: ffffff8009155000 
[  110.204814] x19: ffffff80091b5240 x18: 000000000002fceb 
[  110.205298] x17: 0000000000000000 x16: 0000000000000000 
[  110.205781] x15: 000000000000000a x14: 0ffffffffffffffe 
[  110.206265] x13: 0000000000000030 x12: 0101010101010101 
[  110.206748] x11: 7f7f7f7f7f7f7f7f x10: 73ff67726071621f 
[  110.207231] x9 : 7f7f7f7f7f7f7f7f x8 : ffffff80092ad450 
[  110.207715] x7 : 0000000000000000 x6 : ffffff80092957f7 
[  110.208198] x5 : 0000000000000001 x4 : 0000000000000000 
[  110.208681] x3 : 0000000000000000 x2 : 0000000000000040 
[  110.209164] x1 : 0000000000000000 x0 : 0000000000000001 

```


### e 关掉所有任务
```
debug> sysrq e
[  479.822795] sysrq: SysRq : Terminate All Tasks
[  479.823585] systemd-journald[189]: Received SIGTERM.
```
### f 清理内存
选项 f OOM 清理无用内存
```
debug> sysrq f
[  462.414355] sysrq: SysRq : Manual OOM execution
```
### i 杀死所有的任务
选项 i 杀掉所有 task

### j 解冻所有被冻结的 fs
```
debug> sysrq j
[ 1236.456558] sysrq: SysRq : Emergency Thaw of all frozen filesystems
```

### k sak
```
debug> sysrq k
[ 1276.673330] Emergency Thaw on mmcblk1p7
[ 1276.673702] sysrq: SysRq : SAK
```

### l 打印出有效 CPUs 的 backtrace

### m 显示内存状况
选项 m 显示内存状态
```
debug> sysrq m
[  926.770551] sysrq: SysRq : Show Memory
[  926.770914] Mem-Info:
[  926.771155] active_anon:16200 inactive_anon:5062 isolated_anon:0
[  926.771155]  active_file:11705 inactive_file:19006 isolated_file:0
[  926.771155]  unevictable:0 dirty:22 writeback:0 unstable:0
[  926.771155]  slab_reclaimable:6601 slab_unreclaimable:4644
[  926.771155]  mapped:17519 shmem:5156 pagetables:572 bounce:0
[  926.771155]  free:420091 free_pcp:660 free_cma:0
[  926.774116] DMA free:1680364kB min:5616kB low:7020kB high:8424kB active_anon:64800kB inactive_anon:20248kB active_file:46820kB inactive_file:76024kB unevictable:0kB isolated(anon):0kB isolated(file):0kB present:2095104kB managed:1974036kB mlocked:0kB dirty:88kB writeback:0kB mapped:70076kB shmem:20624kB slab_reclao
[  926.778064] lowmem_reserve[]: 0 0 0
[  926.778401] DMA: 51*4kB (UME) 8*8kB (UME) 24*16kB (UE) 7*32kB (UME) 2*64kB (UE) 34*128kB (UM) 9*256kB (ME) 9*512kB (ME) 1*1024kB (M) 4*2048kB (UME) 405*4096kB (M) = 1680364kB
[  926.779919] 35869 total pagecache pages
[  926.780269] 0 pages in swap cache
[  926.780570] Swap cache stats: add 0, delete 0, find 0/0
[  926.781034] Free swap  = 0kB
[  926.781295] Total swap = 0kB
[  926.781556] 523776 pages RAM
[  926.781817] 0 pages HighMem/MovableOnly
[  926.782160] 30267 pages reserved

```
show-registers(p) show-all-timers(q) unraw(r) sync(s) show-task-states(t) unmount(u) force-fb(V) show-blocked-tasks(w) dump-ftrace-buffer(z)  


### n 使用 nice 优化所有实时任务
使用 nice 优化 cpu 分配到不同进程时间片的多少
```
debug> sysrq n
[ 1385.123664] sysrq: SysRq : Nice All RT Tasks
[ 1385.124142] systemd-journald[899]: /dev/kmsg buffer overrun, some messdebs l st.
```
### o 下电
poweroff 
然而 RK Kernel 默认不支持关机充电。所以此命令无效。

### p 打印所有的 regster 的内容

### q 打印所有的 timer 的状态

### r unraw 
//作用暂时不明

### s 同步
和 linux 下的 sync 用法一致

### t 显示 task 状态

### u unmount 某些设备

### V 
force-fb 
//作用暂时不明

### w 显示被阻塞的 task

### z 打印出 ftrace 的内容
Ftrace是一个内部示踪器，旨在帮助开发人员和 系统的设计人员找到内核中发生了什么。
它可以用于调试或分析延迟和 在用户空间之外发生的性能问题。
https://www.kernel.org/doc/Documentation/trace/ftrace.txt

```
debug> sysrq z
[ 2319.636396] sysrq: SysRq : [ 2319.636605] systemd-journald[979]: /dev/kmsg buffer overrun, some messages lost.

[ 2319.637274] Dump ftrace buffer
[ 2319.637564] Dumping ftrace buffer:
[ 2319.637873]    (ftrace buffer empty)

```



