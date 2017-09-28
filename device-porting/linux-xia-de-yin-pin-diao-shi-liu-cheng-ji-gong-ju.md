# Linux 下的音频调试流程及工具

| Author | Younix |
| ------------- |:-------------:|
| Platform | RK3399 |
| Android Version| Android6.0 |
| Kernel Version| Linux4.4.70 |



## 一、基本概念

**DAI**：音频设备硬件接口 Digital Audio Interfaces ，包括 PCM、I2S、AC97。利用它来实现音频数据在 CPU 和 CodeC 之间的通信。

**PCM**：脉冲编码调制接口。由 BCLK 脉冲时钟、FS 帧同步信号、DR 接受数据、DX 发送数据组成。

**I2S**：Inter-IC Sound。在 LRCLK（Left/Right CLOCK）的信号机制中经过多路转换，将两路音频信号变成单一的数据队列。LRCLK 为高，左声道传输，LRCLK 为低，右声道传输。

**ALSA**：Advanced Sound Linux Architecture，是 音频子系统 在 Linux 中的实现框架。

**OSS**：曾经的 linux 音频体系结构，现在已经被 Alsa 取代并兼容。

linux 平台的音频调试工具有许多，如果你的 linux 系统比较老旧采用的是 OSS 驱动（或者利用 Alsa 对 OSS 兼容层）播放音频文件，那么你可以使用 play / rec 工具进行播放和录音。如果你的 linux 系统采用的是 Alsa 架构，你可以采用 aplay / arecord 进行播放 / 录音。

另外这些工具均只能播放 voc, wav, raw or au 格式的音频文件，并不能正确处理 MP3、OGG 等其他格式。

本文以 aplay、arecord、amixer 系列工具为例谈谈如何进行音频驱动的调试。

## 二、aplay 播放

aplay 用于播放音频

### 2.1 aplay -l 列出声卡和数字音频设备

```
root@linaro-alip:~# aplay -l 
**** List of PLAYBACK Hardware Devices ****
card 0: realtekrt5651co [realtek,rt5651-codec], device 0: ff880000.i2s-rt5651-aif1 rt5651-aif1-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: ROCKCHIPSPDIF [ROCKCHIP,SPDIF], device 0: ff870000.spdif-dit-hifi dit-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 2: HDMICODEC [HDMI-CODEC], device 0: ff8a0000.i2s-i2s-hifi i2s-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

### 2.2 播放
```
root@linaro-alip:~# aplay -D hw:0,0 -r8000 -f cd ./dukou.wav

# -D 参数用于指定音频设备 PCM
# hw:0,0 表示 硬件 0 号声卡下的 0 号设备
# -r 指定采样频率
# -f 指定采样格式 cd/cdr/dat/S16_LE/S32_LE
```

## 三、arecord 录音
arecord 用于录音
### 3.1 arecord -l 列出声卡和数字音频设备
```
arecord -l 
# 用法同 aplay 
```
### 3.2 录音
```
arecord -Dhw:0,1 -r8000 -f cd ./dukou.wmv
```

## 四、amixer 混音器
alsamixer 是 Linux 音频架构 ALSA 工具中的一个。
它是基于文本下的图形界面的，**用于配置音频的各个参数**。
amixer 是 alsamixer 的命令行模式。

值得一提的是当你使用 ALSA 的时候。按照默认设置，ALSA 启动时所有的输出频道都是静音的。因此，你可能能播放一个声音文件，但是却什么都听不到（播放程序可能暂时“冻住”了，但过了一会儿当文件静悄悄地播放完，又“解冻”了）。

所以我们在调试时应该确保所调试的设备解除了静音，并且音量设置在合适的大小。

### 4.1 amixer cmd
```
root@linaro-alip:~# amixer -h
Usage: amixer <options> [command]

Available options:
  -h,--help       this help
  -c,--card N     select the card
  -D,--device N   select the device, default 'default'
  -d,--debug      debug mode
  -n,--nocheck    do not perform range checking
  -v,--version    print version of this program
  -q,--quiet      be quiet
  -i,--inactive   show also inactive controls
  -a,--abstract L select abstraction level (none or basic)
  -s,--stdin      Read and execute commands from stdin sequentially
  -R,--raw-volume Use the raw value (default)
  -M,--mapped-volume Use the mapped volume

Available commands:
  scontrols       show all mixer simple controls
  scontents       show contents of all mixer simple controls (default command)
  sset sID P      set contents for one mixer simple control
  sget sID        get contents for one mixer simple control
  controls        show all controls for given card
  contents        show contents of all controls for given card
  cset cID P      set control contents for one control
  cget cID        get control contents for one control
```

### 4.2 amixer contents 查看可以操作的接口
```
root@linaro-alip:~# amixer contents
numid=11,iface=MIXER,name='Mono ADC Capture Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=127,step=0
  : values=47,47
  | dBscale-min=-176.25dB,step=3.75dB,mute=0
numid=5,iface=MIXER,name='Mono DAC Playback Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=175,step=0
  : values=175,175
  | dBscale-min=-656.25dB,step=3.75dB,mute=0
numid=12,iface=MIXER,name='ADC Boost Gain'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=3,step=0
  : values=0,0
  | dBscale-min=0.00dB,step=12.00dB,mute=0
numid=17,iface=MIXER,name='ADC IF2 Data Switch'
  ; type=ENUMERATED,access=rw------,values=1,items=4
  ; Item #0 'Normal'
  ; Item #1 'Swap'
  ; Item #2 'left copy to right'
  ; Item #3 'right copy to left'
  : values=0
numid=9,iface=MIXER,name='ADC Capture Switch'
  ; type=BOOLEAN,access=rw------,values=2
  : values=on,off
numid=10,iface=MIXER,name='ADC Capture Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=127,step=0
  : values=47,47
  | dBscale-min=-176.25dB,step=3.75dB,mute=0
numid=18,iface=MIXER,name='DAC IF2 Data Switch'
  ; type=ENUMERATED,access=rw------,values=1,items=4
  ; Item #0 'Normal'
  ; Item #1 'Swap'
  ; Item #2 'left copy to right'
  ; Item #3 'right copy to left'
  : values=0
numid=50,iface=MIXER,name='DAC L2 Mux'
  ; type=ENUMERATED,access=rw------,values=1,items=2
  ; Item #0 'IF1'
  ; Item #1 'IF2'
  : values=1

```


### 4.3 amixer 设置某个参数
amixer cget + 控制参数
amixer cset + 控制参数 + " " + 参数

#### 1）设置音量 value
比如，如果要设置主音量：
```
numid=5,iface=MIXER,name=’PCM Volume’
```
可以先看其当前的值
```
~# amixer cget numid=5,iface=MIXER,name=’PCM Volume’
numid=5,iface=MIXER,name=’PCM Volume’
; type=INTEGER,access=rw—R–,values=2,min=0,max=27,step=0
: values=27,27
| dBscale-min=-40.50dB,step=1.50dB,mute=0
```
最大值为 27 ，假设要想设置为 25, 用 cset 去设置：
```
~# amixer cset numid=5,iface=MIXER,name=’PCM Volume’ 25
numid=5,iface=MIXER,name=’PCM Volume’
; type=INTEGER,access=rw—R–,values=2,min=0,max=27,step=0
: values=25,25
| dBscale-min=-40.50dB,step=1.50dB,mute=0
```

#### 2）打开/关闭 Mic 的 switch 
```
~# amixer cset numid=12,iface=MIXER,name=’Mic Supply Switch’ Off
numid=12,iface=MIXER,name=’Mic Supply Switch’
; type=ENUMERATED,access=rw——,values=1,items=2
; Item #0 ‘On’
; Item #1 ‘Off’
: values=1
```

### 4.4 alsactl 将混音器配置保存/读取
这个程序能把当前的混音器设置存到一个文件中或者从文件中读出来。<br>当你用自己喜欢的混音程序调节满意之后，用 root 身份输入 `alsactl store` 。<br>这个命令将把混音设置储存到 `/var/lib/alsa/asound.state` 中。<br>此后，你就能在一个启动脚本中调用 `alsactl restore` 来恢复设置。

保存的格式如下
```
state.realtekrt5651co {
        control.1 {
                iface MIXER
                name 'HP Playback Volume'
                value.0 31
                value.1 31
                comment {
                        access 'read write'
                        type INTEGER
                        count 2
                        range '0 - 39'
                        dbmin -4650
                        dbmax 1200
                        dbvalue.0 0
                        dbvalue.1 0
                }
        }
        control.2 {
                iface MIXER
                name 'OUT Playback Volume'
                value.0 31
                value.1 31
                comment {
                        access 'read write'
                        type INTEGER
                        count 2
                        range '0 - 39'
                        dbmin -4650
                        dbmax 1200
                        dbvalue.0 0
                        dbvalue.1 0
                }
        }
```
