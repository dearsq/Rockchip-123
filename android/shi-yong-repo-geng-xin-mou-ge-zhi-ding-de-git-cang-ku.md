# Android repo 工具的基本用法

## repo 工具是何方神圣
相信从事软件开发的大家都了解 git 版本控制工具。它是当之无愧的版本管理神器。<br>但是同样，它也有一些局限性。比如当我们的项目（比如 Android）过于庞大，需要构建很多个子仓库分别交给不同的开发者维护时。那用 git 便显得力不从心了。<br>于是 google 就琢磨着，咱能不能写个脚本来管理这么多个 git 仓库呢？<br>**repo** 于是应运而生。<br>它的**本质**便是 Python 脚本。<br>它的**作用**主要是用来下载、管理 Android 项目的软件仓库。

那么在这篇文章中，我们会学习了解到 repo 的以下用法：
* init 初始化仓库
* status 显示状态
* sync 同步整个项目
* sync 同步单个项目
* diff 显示版本差异
* download 下载指定版本修改
* start 创建新分支
* prune 删除已经合并的项目


## init 初始化仓库
`repo init -u URL`
在当前目录安装 repository ，会在当前目录创建一个目录 ".repo"。<br>**使用 -u 参数**指定一个URL， 从这个URL 中取得repository 的 manifest 文件。
```
repo init -u git://android.git.kernel.org/platform/manifest.git
```
**使用 -m 参数**来选择 repository 中的某一个特定的 manifest 文件，如果不具体指定，那么表示为默认的 namifest 文件 (default.xml)
```
repo init -u git://android.git.kernel.org/platform/manifest.git -m dalvik-plus.xml
```
**使用 -b 参数**来指定某个manifest 分支。
```
repo init -u git://android.git.kernel.org/platform/manifest.git -b release-1.0
```

## status 显示状态
显示当前 project 中各个子 git 仓库的状态。
```
repo status
```

## sync 同步整个项目
```
repo sync
```
下载新更改,并更新本地环境中的工作文件,如果不带参数,将同步所有的项目的所有文件。<br>当运行repo sync时,会执行以下操作:<br>如果之前未进行过同步,那么等同于git clone, 所有远程的分支都被复制到本地。<br>如果之前同步过,那么等同于
```
git remote update 
git rebase origin/<BRANCH>
```
其中`<BRANCH>`指当前本地目录所有的分支, 如果本地分支没有track远程分支,则不会同步.

## sync 同步单个项目
同步单个 项目 / 文件夹 的方式就是
```
repo sync <project>
```
**< project>** 为 manifest 中的 name 或者 path 字段。

我们打开 .repo/manifest.xml ，有类似如下代码
```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
<remote fetch="." name="aosp" review="https://10.10.10.29"/>
<remote fetch="." name="rk" review="https://10.10.10.29"/>
<default remote="aosp" revision="refs/tags/android-6.0.1_r63" sync-j="4"/>
<project name="android/RKDocs" path="RKDocs" remote="rk" revision="rk3399/box/android-6.0"/>
<project name="android/RKTools" path="RKTools" remote="rk" revision="ce0d44915d3b13cc556ff607dde745f71965ccc3" upstream="rk3399/box/android-6.0"/>
<project name="rk/RKNanoC" path="RKNanoC" remote="rk" revision="master"/>
<project groups="pdk-cw-fs,pdk-fs" name="android/device/common" path="device/common" remote="rk" revision="6482b0e4219edbbb3da2c6e77ce7c2ba22e55ba9" upstream="rk32/mid/6.0/develop"/>
<project groups="pdk" name="android/device/generic/arm64" path="device/generic/arm64" revision="39249053f37b7f9633eb406af3dbedfea7bf8b3e" upstream="refs/tags/android-6.0.1_r63"/>
```
比如我想要同步 RKDocs 项目中的内容。<br>可以看到其 name 或者 path 字段为 android/RKDocs 和 RKDocs。

那么我们可以采用如下命令：
```
repo sync RKDocs
```
或者
```
repo sync android/RKDocs
```

## diff 显示版本差异
```
repo diff <project>
```
显示提交的代码和当前工作目录代码之间的差异。

## download 下载指定版本修改
```
repo download <target> <VersionID>
```
下载特定的修改版本到本地。<br>比如 下载修改版本为 1234 的代码。
```
repo download platform/frameworks/base 1234 
```

## start 创建新分支
```
repo start <branchname> <project>
```

## prune 删除已经合并的项目
删除已经 merge 的 project。
```
repo prune <project>
```



## 欢迎关注微信公众号
黑羊爱学习<br>**blacksheepgogogo**

定期分享 嵌入式 Android/Linux 学习资料。<br>不定期分享 个人职场心得 / 鸡汤 / 游记 / 书籍读后感。

扫描以下二维码：
![](http://ww1.sinaimg.cn/large/ba061518gy1fjskczerf6j20p00f0jx7.jpg)
