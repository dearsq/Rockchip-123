# 添加自定义开机动画后出现黑屏5s的问题

| Author | Younix |
| ------------- |:-------------:|
| Platform | RK3288 |
| Android Version| Android5.1 |


---
跟踪开机信息
发现是由于等待电池的后台服务启动导致的，屏蔽如下代码。

 frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp

  ![](https://ws3.sinaimg.cn/large/ba061518gw1f7kstpdro9j20mx0btjuz.jpg)

屏蔽后黑屏时间减为 1s 左右。