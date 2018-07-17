---
layout: post
title: art和dalvik的区别
category: android
keywords: android,art,dalvik
---

# 0x01.前言

这几天一直在翻aosp的文档,正好看到了知乎上有人问“art和dalvik的区别”就来答一波

[知乎回答](https://www.zhihu.com/question/29406156/answer/83413563)

# 0x02.区别

**主要如下**:
> * **Ahead-of-time (AOT) compilation instead of Just-in-time (JIT)** 
> * **Improved garbage collection**
> * **Improved memory usage and reduce fragmentation**

# 0x021.AOT代替了JIT

**Ahead-of-time (AOT) compilation instead of Just-in-time (JIT)**
 
在dalvik中(实际为android2.2以上引入的技术),如同其他大多数jvm一样,都采用的是jit来做及时翻译(动态翻译),将dex或odex中并排的dalvik code(或者叫smali指令集)运行态翻译成native code去执行.jit的引入使得dalvik提升了3~6倍的性能

而在art中,完全抛弃了dalvik的jit,使用了aot直接在安装时用dex2oat将其完全翻译成native code.这一技术的引入,使得虚拟机执行指令的速度又一重大提升


# 0x022.更高的垃圾回收效率

**Improved garbage collection**


首先介绍下dalvik的gc的过程.主要有有四个过程:

1. 当gc被触发时候,其会去查找所有活动的对象,这个时候整个程序与虚拟机内部的所有线程就会挂起,这样目的是在较少的堆栈里找到所引用的对象.需要注意的是这个回收动作是和应用程序**同时执行(非并发)**.
2. gc对符合条件的对象进行标记
3. gc对标记的对象进行回收
4. 恢复所有线程的执行现场继续运行


**dalvik这么做的好处是,当pause了之后,gc势必是相当快速的.但是如果出现gc频繁并且内存吃紧势必会导致ui卡顿,掉帧.操作不流畅等.**

后来art改善了这种gc方式(也是想对ui流畅度做贡献,当然关于ui流畅,5.0以上了新的并行ui线程),**主要的改善点在将其非并发过程改变成了部分并发.还有就是对内存的重新分配管理**

当art gc发生时:

1. gc将会锁住java堆,扫描并进行标记
2. 标记完毕释放掉java堆的锁,并且挂起所有线程
3. gc对标记的对象进行回收
4. 恢复所有线程的执行现场继续运行
5. 重复2-4直到结束

可以看出整个过程做到了部分并发使得时间缩短.据官方测试数据说gc效率提高2倍


# 0x023.增加内存使用,减少内存碎片

**Improved memory usage and reduce fragmentation**

官方把这一点合并到了**Improved garbage collection**这个主题中讲,原因也是和gc有很大关系
可以对比一下两个虚拟机的内存分配的规则,首先是dalvik.他的内存管理特点是:内存碎片化严重,当然这也是Mark and Sweep算法带来的弊端

该算法如图(图片来自《深入理解jvm》):

![1](http://smali.test.upcdn.net/dvm-art-diff-img-1.png)

可以看出每次gc后内存千疮百孔,本来连续分配的内存块变得碎片化严重,之后再分配进入的对象再进行内存寻址变得困难

art的解决:在art中,它将java分了一块空间命名为**Large-Object-Space**,这块内存空间的引入用来专门存放large object.同时art又引入了moving collector的技术,即将不连续的物理内存块进行对齐.对齐了后内存碎片化就得到了很好的解决.Large-Object-Space的引入一是因为moving collector对大块内存的位移时间成本太高,而且提高内存的利用率

根官方统计，art的内存利用率提高10倍了左右


参考链接:[ART and Dalvik](http://source.android.com/devices/tech/dalvik/index.htmll)