---
layout: post
title: android 4.0.3源码编译小结
category: android
keywords: android,aosp,source,编译
---

## 0x01. 前言
android6.0发布了,删除了n年之前编译的2.3.5_r1的镜像,打算编译4.0.3_r1

为什么是4.0呢?主要有两点原因:

> * 体积小,包括4.0迁入的各种外设驱动,4.4以上迁入chromium都是硬盘大户.
> * 我想研究的core并没有因为版本变化而改动太多,4.x版本主要改动都是硬件外设等,而5.0改动主要是art的引入以及ui等.所以4.0的代码够研究了


## 0x02. 准备阶段

### 0x021.环境准备

环境准备:

> * **ubuntu** 10.04以上,笔者用14.04.3 kylin lts,原因只是因为kylin挺美的,当然其他的发行版也可以,只不过又要替换system lib,又要换取
> * **虚拟机(可选)** ,笔者是mac 15inch低配(i7 16g 256ssd),装了vmware fu8,这个一堆坑,子标题细说.
> * **jdk** , oracle jdk 6 ,注意一定不能用Openjdk,落坑了
> * **环境库依赖** , 一堆坑,子标题细说
> * **源码下载** , 更换清华源下载

### 0x022.ubuntu与虚拟机

ubuntu请在官方源站下载.别随便百度一个阉割版.

ubuntu虚拟机基本硬件设置

> * 2核 4g 配置不能太低,太低编译慢,很容易出问题
> * 硬盘最少40g+,下载+编译35g
> * 硬盘最好挂载在同一个分区,不是一个分区挺麻烦

**注意** : 一定要**在ubuntu安装系统前**将vm的,切分好分区大小,要不然动态扩张盘很麻烦
![ubuntu-config](http://7xkw0v.com1.z0.glb.clouddn.com/屏幕快照%202015-11-29%20下午5.00.48.png)

当然你也可以先分配一定大小的磁盘,然后用lvm切分分区,动态扩张.当然不保证原有数据不被格式化反正很麻烦.

### 0x023.jdk

一定要去安装oracle jdk,否则通过不了编译检查

网上的方法主要有两种:

1. 添加repo源 -> 安装
2. 下载deb -> 安装

**太麻烦**,这里介绍一下直接不添加repo,下载安装的办法


	sudo apt-get install python-software-properties
	sudo add-apt-repository ppa:webupd8team/java
	sudo apt-get update
	sudo apt-get install oracle-java6-installer

简单粗暴

之后如果已经安装有openjdk,可以通过ubuntu的切换命令切换jdk

	sudo update-alternatives --config java

这样java就安装好了

### 0x024.环境库依赖

这个是最繁琐与复杂的

官方上写着14.04安装如下依赖:

	sudo apt-get install git gnupg flex bison gperf build-essential zip curl libc6-dev \		libncurses5-dev:i386 x11proto-core-dev libx11-dev:i386 libreadline6-dev:i386 \
	libgl1-mesa-glx:i386 libgl1-mesa-dev g++-multilib mingw32 tofrodos python-markdown \
	libxml2-utils xsltproc zlib1g-dev:i386 
	
总结了一下,有两个是不能直接安装上去的,可以先执行可以安装的依赖命令:

	sudo apt-get install git gnupg flex bison gperf build-essential zip curl libc6-dev \
	libncurses5-dev:i386 x11proto-core-dev libx11-dev:i386 libreadline6-dev:i386 \
	g++-multilib mingw32 tofrodos python-markdown libxml2-utils xsltproc zlib1g-dev:i386 

然后我们还需要安装:

	libgl1-mesa-glx:i386 libgl1-mesa-dev

但是这两个依赖死活都安装不上去.

后来我发现他的依赖有很多都是别的包替换掉.

所以先执行:

	sudo apt-get install libglew-dev libcheese7 libcheese-gtk23 libclutter-gst-2.0-0 libcogl15 \
	libclutter-gtk-1.0-0 libclutter-1.0-0
	
	sudo apt-get install xserver-xorg-dev-lts-utopic mesa-common-dev-lts-utopic \
	libxatracker-dev-lts-utopic libopenvg1-mesa-dev-lts-utopic libgles2-mesa-dev-lts-utopic \
	libgles1-mesa-dev-lts-utopic libgl1-mesa-dev-lts-utopic libgbm-dev-lts-utopic \
	libegl1-mesa-dev-lts-utopic
	
	sudo apt-get install --install-recommends linux-generic-lts-vivid \
	xserver-xorg-core-lts-vivid xserver-xorg-lts-vivid xserver-xorg-video-all-lts-vivid \
	xserver-xorg-input-all-lts-vivid libwayland-egl1-mesa-lts-vivid libgl1-mesa-glx-lts-vivid \
	libglapi-mesa-lts-vivid libgles1-mesa-lts-vivid libegl1-mesa-lts-vivid \
	xserver-xorg-dev-lts-vivid mesa-common-dev-lts-vivid libxatracker-dev-lts-vivid \
	libgles2-mesa-dev-lts-vivid libgles1-mesa-dev-lts-vivid libgl1-mesa-dev-lts-vivid \
	libgbm-dev-lts-vivid libegl1-mesa-dev-lts-vivid

执行

	sudo apt-get install libgl1-mesa-dev

然后装libgl1-mesa-glx:i386(![Android, setting up a Linux build environment, libgl1-mesa-glx:i386 package have unmet dependencies](http://stackoverflow.com/questions/23254439/android-setting-up-a-linux-build-environment-libgl1-mesa-glxi386-package-have))

	sudo apt-get install libgl1-mesa-dri-lts-trusty:i386 libgl1-mesa-glx-lts-trusty:i386 \
	libc6:i386

	sudo apt-get install git gnupg flex bison gperf build-essential \
	zip curl libc6-dev libncurses5-dev:i386 x11proto-core-dev \
	libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-dev g++-multilib mingw32 tofrodos \ 
	python-markdown libxml2-utils xsltproc zlib1g-dev:i386

	sudo apt-get install ubuntu-desktop xserver-xorg
	sudo apt-get install libglapi-mesa-lts-saucy:i386

执行

	sudo apt-get install libgl1-mesa-glx:i386

这样就可以全部安装完了


你以为就这样结束?native!
还要做一件事,重装g++和gcc,因为用的是老的g++,gcc编译的,不重装一定出错

	sudo apt-get install gcc-4.4 g++-4.4
	sudo rm -rf /usr/bin/gcc /usr/bin/g++
	sudo ln -s /usr/bin/gcc-4.4 /usr/bin/gcc
	sudo ln -s /usr/bin/g++-4.4 /usr/bin/g++
	sudo apt-get install g++-4.4-multilib gcc-4.4-multilib


**ps:如果期间出依赖错,请自行百度**


### 0x025.源码下载

总有人傻到去直接用googlesource下载.国内aosp的镜像挺多的.所以我们可以用清华大学的中科大的镜像站都可以用来下载

这里介绍一下清华大学镜像站下载aosp方法

**step1**

下载修改好的[repo](http://pan.baidu.com/s/1kTwDyPh)

**step2**

	mkdir ~/bin
	PATH=~/bin:$PATH

**step3**

将repo 拷贝到~/bin下面

**step4**
	
建立android源码存放的路径,比如我就建立在

	mkdir ~/android-source
	cd ~/android-source
	
**step5**

初始化git的用户信息,因为repo需要验证git

	git config --global user.name "example name"
	git config --global user.email "mail@example.com"
	
**step6**

同步repo信息

注意两点

> * 这条命令请保证在android-source路径执行
> * 可以自行选择android版本,版本选择请详见[android各个版本](https://source.android.com/source/build-numbers.html#source-code-tags-and-builds),选择了版本然后替换进下面命令

	repo init -u git://aosp.tuna.tsinghua.edu.cn/android/platform/manifest -b android-4.0.3_r1

**step7**

同步repo

注意:请在源码目录执行该命令

	repo sync -j4
	
4为并发数,清华大学的repo最大支持4并发,而且如果是单核也没有必要j4,指定j2,这个python的多进程有关.


**step8**

嘟嘟嘟,过了几个小时repo就被同步了下来.4.0.3的大概12g+,我花了2~3小时同步了下来.


## 0x03. 编译&运行

终于环境也搭建好了,源码也下载好了.可以执行编译命令了

	source build/envsetup.sh
	lunch full-eng
	make -j4 showcommands
	
	
make命令执行时务必加上showcommands,否则根本不知道他到底在不在运行(不打Log)

然后你会发现,日了狗又一堆报错.原因是google打了release之后,又会增加很多patch.

不要慌,可以参照这几篇文章来debug
[传送门1](http://blog.csdn.net/deng0zhaotai/article/details/37034105)
[传送门2](http://blog.csdn.net/dongwuming/article/details/13624319)

感觉上面的bug我都遇到了,前半个小时最多了.过了半个小时基本就不会出error了就等他安安静静的编译把.

大概编译了两个小时:
![编译成功](http://7xkw0v.com1.z0.glb.clouddn.com/1.png)
编译成功,并且有了三个镜像文件

运行

	emulator

出现
![运行成功](http://7xkw0v.com1.z0.glb.clouddn.com/2.png)


结束!



