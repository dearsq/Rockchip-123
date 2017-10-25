# PCIe

## 历史
PCI Bus 由 ISA Bus 发展而来。

ISA 有 8bit 和 16bit 两种模式。时钟频率为 8 MHz，总线带宽为 `8bit*8MHz=64Mbps=8MB/s` 或 `16bit*8MHz=128Mbps=16MB/s`

然后 IBM 推出了 32bit 的 MAC Bus，总线带宽达到了 40MB/s。

所以为了与其竞争，诞生了 EISA Bus，`32bit*8MHz=256Mbps =  32MB/s`。

后来又由 Intel 提出了 PCI Bus。
PCI总线支持32位和64位两种位宽，时钟频率为33MHz，总线带宽：`32bit*33MHz= 1056Mbps =132MB/s` 或 `64bit*33MHz=2112Mbps=264MB/s`

后来就是不断升级时钟频率的 PCI-X 时代。
PCI-X 1.0 的时钟频率有 66MHZ/100MHz/ 133MHz，总线带宽分别为：264MB/s, 400MB/s和532MB/s(32位)，528MB/s, 800MB/s和1064MB/s(64位)；   
PCI-X 2.0的时钟频率有266MHz/533MHz/1066MHz，总线带宽分别为：1064MB/s, 2132MB/s和4264MB/s(32位)，2128MB/s, 4264MB/s和8512MB/s(64位)，PCI-X与PCI总线在硬件结构上完全兼容。

现在被称为 PCI-Express 时代，串行高速总线。
分为X1,X2,X4,X8,X12,X16和X32七种模式。
PCI-E 1.0 的速率为 2.5Gbps，
PCI-E 2.0 的速率为 5.0Gbps，
PCI-E 3.0 的速率可达 8.0Gbps。

## 总线结构

树型结构，并且独立于CPU总线，可以和CPU总线并行操作。
PCI总线上可以挂接PCI设备和PCI桥片，PCI总线上只允许有一个PCI主设备，其他的均为PCI 从设备，而且读写操作只能在主从设备之间进行，从设备之间的数据交换需要通过主设备中转。

### 管脚功能 
PCI主设备最少需要49根线，从设备最少需要47根线，剩下的线可选。在介绍PCI管脚功能前，先来说明下PCI管脚信号的类型。

**in**：输入信号；
**out**：输出信号；
**t/s**：双向三态信号(Tri-state)，无效时为高组态；     
**s/t/s**：持续三态信号(Sustained Tri-state)，每次由且只由一个单元拥有并驱动的低有效、双向、三态信号。驱动一个s/t/s信号到低的单元在释放该信号浮空之前必须要将它驱动到高电平至少一个周期。这个特点很重要，在后面我们分析PCI信号质量案例的时候会用到；    
**o/d**：漏极开路输出(Open Drain)；    
**#**：此符号代表该信号在低电平时有效。

![](http://ww1.sinaimg.cn/large/ba061518ly1fkk42lhrisj20de0f8ae8.jpg)

实际使用中需要上拉的信号有: FRAME#, TRDY#, IRDY#, DEVSEL#, STOP#, PERR#, SERR#, LOCK#, REQ64#, ACK64#, REQ#, GNT#, AD[63:32], C/BE[7:4],PAR64 等,上拉电阻一般为 10kohm , 未使用的 PCI 管脚也要处理,避免悬空。

不需要上拉的信号有 AD[31:0], C/BE[3:0], PAR, IDSEL, CLK

## 其他
### mini PCIe 接口
mini PCIe 是基于 PCIe 协议的。
主设备兼容 PCIe 和 USB2.0 连接，可以使用两种标准。

![](http://ww1.sinaimg.cn/large/ba061518ly1fkk4txhkt9j206404n0sv.jpg)

### mini SATA
它还有 miniSATA(mSATA) 的变体。

尽管和 mini PCIe 外形尺寸相同，但是不一定和 mini PCIe 电气特性兼容。
这种变体通过 保留和未保留某些引脚 实现 SATA 和 IDE 接口直通，只保留 USB、接地线，有时候仍然保留 PCIe X1 总线。

### 固态硬盘的两种接口
固态硬盘 SSD 有两种类型的物理接口规格（插座标准），原来的 SATA 接口和 现在的 M.2 接口。

M.2 也叫作 NGFF，有两种接口模式，socket2 和 socket3;
socket2 硬件接口对应的是 bkey，总线标准是 sata。传输标准也是 AHCI。
socket3 硬件接口对应的是 mkey，总线标准是 pcie。传输标准有两种 NVME 和 AHCI。