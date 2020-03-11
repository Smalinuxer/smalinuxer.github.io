---
layout: post
title: virtio spec v1.1 中文翻译(1) 1-2章 前言、Virtio设备的基本设施
category: store
keywords: store,virtio,virtualization
---

## 0. 写在前面的话

virtio这个东西，本人其实并不是非常熟悉，项目需要，开始学习spec。翻译可能并不尽善尽美，也会加入我自己的一些理解。希望您能多多体谅。

如果有翻译出错的地方可以联系我的邮箱。如果您有更多更好的建议，也可以联系本人。或有任何问题，您可以联系我的私人邮箱 [smalinuxer@gmail.com](mailto://smalinuxer@gmail.com)

翻译内容中，带有mark字段的为翻译者所著，表示自身不理解的地方

原文档地址为：[virtio-v1.1.pdf](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf)
 
版权归属于OASIS所有，翻译版本只为学习交流，请勿商务用途，翻译版本版权归属于[jiaqizho.github.com](http://jiaqizho.github.com)


## 1. 前言
该文档描述了能使用“virtio”的设备家族的技术规范，这些设备将在虚拟环境中可视化。通过这种设计，对于虚拟机看来，这些虚拟设备就像是物理设备一样。 并且在本文档中也将虚拟设备视为物理设备。这将允许虚拟机（guest os)使用标准的驱动程序(driver)和驱动扫描程序(driver discovery)。
virtio和此规范的目的是，虚拟环境和虚拟机应具有用于虚拟设备的简单，高效，标准和可扩展的机制。而不是对于不同的系统需要定制不同的环境。

>* Straightforward： 略 
>* Efficient：略
>* Standard：略
>* Extensible ： 略

### 1.1 Normative References
### 1.2 Non-Normative References
### 1.3 Terminology
不重要，略
### 1.4 结构规范


在本文档中，大多数设备和驱动的内存数据分布将使用C的结构体语法来表示。所有结构体都假定没有额外填充。强调一下，通常C编译器都会加入一些额外的填充在使用了GNU C __attribute__((packed)) 的语法（mark:我怎么感觉他说反了）

- u8, u16, u32, u64 带长度无符号整数
- le16, le32, le64 带长度无符号整数, 小端模式
- be16, be32, be64 带长度无符号整数, 大端模式

在本文档中，定义的某些字段不在字节边界处开始或结束，这些字段被称为bit-fields, 这么一组bit-fields始终是整数类型字段的细分。在整数字段中的bit-fields始终按从低到高的顺序列出。bit-fields被定义为是指定宽度的无符号整数，下一个是和其相关连的保留位。

举个例子：
```$c
struct S {
	be16 {
		A : 15;
		B : 1;
	} x;
	be16 y;
};
```
这里例子里面，A存在x的低15位中，B存在x的高位中，整数x依次使用big-endian字节序存储在结构S的开头，并随后紧贴着big-endian字节序存储的无符号整数y从偏移2字节（16位）。
结构的开始。

这个语法有点类似于C语言的bitfield语法，但是不要natively的以为这个是可以移植的（意思是这两者并不相同）。它在小端的打包方式上一样，在大端的方式上不同。
假设CPU_TO_BE16将16位整数从本地CPU转换为大端字节序，则以下是等效的可移植C代码，用于生成要存储到x中的值：

```$c
CPU_TO_BE16(B << 15 | A)
```

## 1. virtio设备的基础设备

一个virtio设备时会被通过bus-specific方法（见原文4.1，4.2，4.3）辨认发现。每一个virtio设备由以下几个部分组成
 - 设备状态属性(Device status field)
 - 功能位(Feature bits)
 - 通知(Notifications)
 - 设备配置空间(Device Configuration space)
 - 一个或多个virtqueues(One or more virtqueues)

### 2.1 设备状态属性(Device status field)

在驱动进行初始化时，驱动将文档3.1中指定的步骤。

设备状态字段提供了一些已完成步骤的简单低级指示,想象一下它作为交通信号灯挂在控制台上指示每个设备状态。


- **ACKNOWLEDGE(1)** 表示虚拟机已找到该设备并将其识别为有效的virtio设备。
- **DRIVER(2)** 表示虚拟机(或驱动)已经知道如何使用virtio设备。注：设置该位之前可能会有大的（或无限）的延迟。例如，在Linux下，驱动程序可以是可加载模块，意思就是可能一直准备着不加载。
- **FAILED(128)** 表示虚拟机出现了一些错误并且虚机放弃了该virtio设备。这可能是一个虚机内部错误，或者是驱动和virtio设备并不适配。或者当在操作virtio设备时出的错。
- **FEATURES_OK(8)** 表示驱动适配设备，驱动已经完全了解且适配设备的功能。
- **DRIVER_OK(4)** 表示驱动已经设置好，准备通过驱动使用设备。
- **DEVICE_NEEDS_RESET(64)** 表示设备出现了错误并且它不能恢复

### 2.1.1 驱动要求：设备状态属性

- 驱动必须得更新设备状态，设置状态属性位来表示驱动初始化的程度。驱动一定不能清除设备状态属性位。如果驱动设置了设备状态为FAILED，那么驱动必须先尝试重新初始化不行再去重置设备。
- The device MUST initialize device status to 0 upon reset（mark:没看懂这句话）
- 当状态不是DRIVER_OK时候，设备一定不要消费或发送任何bufeer notification去驱动
- 当设备发生了错误，并且需要重置的时候，设备最好设置为DEVICE_NEEDS_RESET。如果当前设备时DRIVER_OK状态的，那么它在被设置为DEVICE_NEEDS_RESET时候，该设备一定要发送一个configuration change notification通知(2.3中专用通知)给驱动。

### 2.2 功能位
每一个virtio设备提供了它的所有的功能。在当设备初始化的时候，驱动读到它的功能位，并且告诉设备的子集（mark:原文tells the device the subset）它接受了，如果想重新获取的话那只能重置了。

这样将提供向前和向后的兼容性：如果设备有了一个新的功能，那么旧的驱动不会将该功能位写回设备。同样，如果驱动有一个新特性但是设备不支持，这个功能也是不会对外提供的。功能位字节分布如下：

- 0 to 23 特殊的设备属性的功能位
- 24 to 37 保留用于扩展队列和功能协商机制的功能位
- 38 and above 保留功能位以供将来扩展。

举个例子，网络设备的功能位0（即设备ID 1）表示这个设备能对网络包进行校验。


### 2.2.1 Driver Requirements: Feature Bits
### 2.2.2 Device Requirements: Feature Bits
### 2.2.3 Legacy Interface: A Note on Feature Bits
略



### 2.3 通知
“发送通知”（驱动到设备，设备到驱动）的概念是非常重要的。通知（Notifications）是一种特殊的通信手段。
以下有三种通知属性：
- configuration change notification 配置更改通知
- available buffer notification 可用buffer通知
- used buffer notification 用掉了buffer通知 (笔者注：我并不明白这个是标记成用过了还是重用

配置更改通知(configuration change notification)和复用buffer通知（used buffer notification）都是设备发送给驱动的。配置更改通知表示设备的配置空间(Device Configuration space）已经改变了，用掉了buffer通知表示这条消息里面指定的virtqueue中的buffer可能是用掉了的。

可用buffer通知(available buffer notification)是驱动发送给设备的，这个通知类型代表了这条消息里面指定的virtqueue中的buffer是可以用的

在以下各章中详细说明了不同通知的语义，特定于传输的实现以及其他重要方面。
大多数传输会使用中断来实现设备发送给驱动程序的通知。因此，在文档的先前版本中，这些通知通常称为中断。在本文档中，仍保留该中断术语。有时，“中断”用于指代通知或收到通知。

### 2.4 设备配置空间

设备配置空间通常用于rarely-changing或initialization-time参数。配置字段是可选的，功能位会标识配置空间存在与否：该文档的未来版本可能会在尾部添加额外的字段来扩展设备配置空间。

注：设备配置空间使用小端的格式表示多个字节属性

在每一个传输（transport，mark不知道这里传输指的啥，指的是session？）提供了设备配置空间的生成计数。那么对设备配置空间的两次访问都可以看到不同版本的配置空间。


### 2.4.1 驱动要求：设备配置空间

驱动不能假定从大于32位宽的字段进行的读取是原子的。驱动应该读取设备配置空间属性如下：
```
u32 before, after;
do {
	before = get_config_generation(device);
	// read config entry/entries.
	after = get_config_generation(device);
} while (after != before);

```
我注:以前我想如果非原子资源的话，加把锁或者尽量保证数据原子性。 看起来遇到驱动不好加锁的场景，这个办法也是非常 trick 的。

对于可选的配置空间字段，驱动必须检查是否提供了相应的功能，在访问配置空间的那一部分之前。

驱动不能限制结构大小和设备配置空间大小。相反，驱动应仅检查设备配置空间是否足够大，以包含设备操作所需的字段。

例如，如果文档指出设备配置空间必须“包括一个8位字段”，驱动应了解这意味着设备配置空间可能还包括一个任意数量的尾部填充，并且驱动接受的设备配置空间
必须等于或大于8位。

### 2.4.2 驱动要求：设备配置空间

在设置FEATURES_OK之前，设备必须允许驱动读取任意设备特定的配置字段。这包括以功能位为条件的字段，只要这些功能位由设备有提供。

### 2.4.3 驱动要求：关于设备配置空间字节序的说明

对于旧版界面，设备配置空间通常是虚拟机的本地字节序，而不是PCI的小端。The correct endian-ness is documented for each device。

### 2.4.4 Legacy Interface: Device Configuration Space

也是讲的老版本，略

## 2.5 Virtqueues

在virtio设备上进行批量数据传输的机制被称为virtqueue。 每个设备可以具有零个或多个virtqueue。

驱动通过向队列添加可用buffer区，即添加一个有buffer描述的请求到virtqueue。并可选地触发驱动事件，即发送available buffer notification通知设备。设备计算使用的每个缓冲区已写入内存的字节数。这被称为“used length”。

设备通常不会使用已经被标记为available buffer的buffer区(mark:我不知道理解成连续的buffer还是能复用的Buffer)。

有些驱动会使用同样序列的缓存区（mark:连续?），他们会提供VIRTIO_F_IN_ORDER功能标记。如果驱动也同意的话，这将会简化设备和驱动的代码。

virtqueue由三个部分组成：
 - 描述区域（Descriptor Area）： 用于描述缓冲区
 - 驱动区域（Driver Area）：驱动提供给设备的额外数据
 - 设备区域（Device Area）：设备提供给驱动的额外数据
 
 
两种格式的virtqueue时支持的：Split Virtqueues（2.6章描述）和Packed Virtqueues（2.7章描述）

任何驱动和设备支持Split Virtqueues和Packed Virtqueues的其中一种或者两种。

## 2.6 分割类型Virtqueues

未完待续

