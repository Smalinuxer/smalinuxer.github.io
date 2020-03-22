---
layout: post
title: virtio spec v1.1 中文翻译(1) 1-2章 前言、Virtio设备的基本设施
category: store
keywords: store,virtio,virtualization
---


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

### 2.2 功能位(Feature Bits)
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



### 2.3 通知(Notifications)
“发送通知”（驱动到设备，设备到驱动）的概念是非常重要的。通知（Notifications）是一种特殊的通信手段。
以下有三种通知属性：
- configuration change notification 配置更改通知
- available buffer notification 可用buffer通知
- used buffer notification 用掉了buffer通知 (笔者注：我并不明白这个是标记成用过了还是重用

配置更改通知(configuration change notification)和复用buffer通知（used buffer notification）都是设备发送给驱动的。配置更改通知表示设备的配置空间(Device Configuration space）已经改变了，用掉了buffer通知表示这条消息里面指定的virtqueue中的buffer可能是用掉了的。

可用buffer通知(available buffer notification)是驱动发送给设备的，这个通知类型代表了这条消息里面指定的virtqueue中的buffer是可以用的

在以下各章中详细说明了不同通知的语义，特定于传输的实现以及其他重要方面。
大多数传输会使用中断来实现设备发送给驱动程序的通知。因此，在文档的先前版本中，这些通知通常称为中断。在本文档中，仍保留该中断术语。有时，“中断”用于指代通知或收到通知。

### 2.4 设备配置空间(Device Configuration Space)

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

## 2.6 Split Virtqueues

Split virtqueue 格式只在标准1.0之后支持

Split virtqueue格式将virtqueue分割了好多部分，每一个部分对于设备和驱动来说都是可写的， 但是只有一个能写。当缓冲区改变的时候（available or used buffer notification）在Split virtqueue里面的多个parts且或在parts中的locations都得更新。

每一个队列有16位大小的队列参数，这个参数设置了条目数且表示了队列总大小。

每一个队列由三个部分组成：

- Descriptor Table(描述表) : 描述符的区域
- Available Ring(可用环形区) : 驱动区域
- Used Ring (已用的环形区) ：设备区域

每一个部分都是物理连续的在虚拟机内存，并且有不同的对齐要求。

virtqueue内存对齐和大小要求如下：

| Virtqueue Part  |	Alignment   |	Size               |
| --------        |  :-----:    |   :----:             |
| Descriptor Table| 	16 		|	16 ∗ (Queue Size)  | 
| Available Ring  |	    2 		|	6 + 2∗(Queue Size) |
| Used Ring       |	    4 		|	6 + 8∗(Queue Size) |

表描述：
- Alignment列给出virtqueue的每个部分的最小对齐方式。
- Size列给出virtqueue每个部分的总字节数。
- Queue Size对应于virtquee中的最大缓冲区数。Queue Size的值总是2的幂，最大Queue Size值为32768。此值是以bus-spec的方式指定的。

当驱动想要发送一个buffer到设备，它将填充描述表(或多个表连在一起)，并且将描述索引(Descriptor index)放入可用环形区。之后驱动会通知设备。当设备完成了buffer的处理，设备会将描述索引写到已用的环形区。并且发送出一个used buffer notification。


### 2.6.1 驱动要求：Virtqueues

驱动必须保证每一个virtqueue第一个字节的物理地址是上表指定对齐值的倍数。

### 2.6.2 遗留接口：virtqueues的内存结构

对于遗留的接口，对virtqueues的内存结构有了一些限制。

略，老的virtuqueues内存分布

### 2.6.3 遗留接口：virtqueues的Endianness

略

### 2.6.4 消息框架

带有描述符的消息框架是和buffer内容无关的。举个例子，一个网络传输的buffer由12个字节的头通过网络包组成，这将可以简单地放在描述符表中，作为12 + 1514字节的输出描述符。但是同样的，它也可以头和包内容相邻，组成一个1526大小的输出描述符。或也可以由三个或者更多描述符组成（可能会性能降低）。

对于描述符长度，有些设备的实现附带了很多且有来由的限制（比如虚拟机设置的IOV_MAX）。这点是没啥问题的：对于那些创建不合理大小的描述符的驱动程序，比如将一个网络数据包分成1500个单字节的描述符，我们不会给予多少同情！（翻译者:你吼那么大声干什么啦？）

### 2.6.4.1 驱动要求：消息框架

设备不得对描述符的特定排列做出假设。The device MAY have a reasonable limit of descriptors it will allow in a chain.

### 2.6.4.2 驱动要求：消息框架

驱动必须要防止任意设备可写描述符(device-writable descriptor)元素在设备可读描述符(device-readable descriptor)元素之后。

驱动不应该用太多的描述符来描述buffer。（翻译者注：上文说性能会下降）

### 2.6.4.3 遗留接口：消息框架

略

### 2.6.5 Virtqueue的描述表

描述表记录了设备已用的buffer。addr是物理地址，并且buffer是可以用next串联到下一个的，描述符描述的buffer只能是只读或者只写的。但是串联起来的描述符可以同时包含可读或可写buffer。

提供给设备的内存的实际内容取决于设备类型。 最常见的方法是在数据的开头加上一个头（包含一些小端数据）以便设备读取。并在数据尾部加入一个状态字段以便设备写入。

```
struct virtq_desc {
	/* Address (guest-physical). */
	le64 addr;
	/* Length. */
	le32 len;
	/* This marks a buffer as continuing via the next field. */
	#define VIRTQ_DESC_F_NEXT 1
	/* This marks a buffer as device write-only (otherwise device read-only). */
	#define VIRTQ_DESC_F_WRITE 2
	/* This means the buffer contains a list of buffer descriptors. */
	#define VIRTQ_DESC_F_INDIRECT 4
	/* The flags as indicated above. */
	le16 flags;
	/* Next field if flags & NEXT */
	le16 next;
};
```

由virtqueue的队列长度来定义描述表的大小：描述表容量的最大值为virtqueue的大小

如果用了VIRTIO_F_IN_ORDER这个功能位，驱动使用描述符(mark:不是在描述表吗，为什么又是in ring,wrng)的顺序：从下标0到表的末尾。


### 2.6.5.1 设备要求：virtqueue的描述表

设备不能写只读，设备不应该去读只写去（可能是debug会这么做）。

### 2.6.5.2 驱动要求：virtqueue的描述表

设备的描述符链总长一定不能长过2^32字节。这意味着描述符链循环是被禁止的!

如果用了VIRTIO_F_IN_ORDER功能位，然后下标为x的描述符中有VRING_DESC_F_NEXT标，当x = queue_size - 1的时候，驱动必须在next中设置0，并且x + 1为其他的描述符。

### 2.6.5.3 间接描述符

有些设备通过并发分发大量的大请求获得收益。VIRTIO_F_INDIRECT_DESC功能位允许这一点（可以参考virtio_queue.h），为了增加环的容量，驱动可以存一个内存中的间接描述符表(table
of indirect descriptors)，并插入一个描述符在主virtqueue(flags&VIRTQ_DESC_F_INDIRECT要为true)，这样内存就可以包含非直接描述表。addr和lenrefer对应间接描述符表中的地址和长度。

间接描述符表内存分布如下(len为指向的描述符的长度，是一个变量，所以这个并不能编译）：

```
struct indirect_descriptor_table {
	/* The actual descriptors (16 bytes each) */
	struct virtq_desc desc[len / 16];
};
```

第一个间接描述符在间接描述表的第一个（index 0 ），其他的间接描述符用next属性相连，间接描述符在尾部的话，它是没有有效的next(flags&VIRTQ_DESC_F_NEXT为false)信号。单个间接描述表可以包括可读、可写两种描述符。

如果有VIRTIO_F_IN_ORDER功能位，间接描述符是顺序索引：index 0后面是index 1、index 2.... 。

### 2.6.5.3.1 驱动要求：间接描述表

- 在VIRTIO_F_INDIRECT_DESC功能位未被设置时，驱动必须不不设置VIRTQ_DESC_F_INDIRECT功能位。驱动不能在间接描述符中设置VIRTQ_DESC_F_INDIRECT功能位（即 每个描述符只有一个表）。
- 驱动创建的描述符链不能超过设备的队列大小。
- 驱动不能同时设置VIRTQ_DESC_F_INDIRECT和VIRTQ_DESC_F_NEXT功能位。
- 如果设置了VIRTIO_F_IN_ORDER功能位，间接描述符必须是顺序的。

### 2.6.5.3.2 设备要求：间接描述表

- 设备必须忽略在间接描述符中的只读标识(flags&VIRTQ_DESC_F_WRITE)
- 设备必须处理零个或多个正常且带有flag&VIRTQ_DESC_F_INDIRECT描述符。

### 2.6.6 Virtqueue可用环形区

可用环形区结构：
```
struct virtq_avail {
	#define VIRTQ_AVAIL_F_NO_INTERRUPT 1
	le16 flags;
	le16 idx;
	le16 ring[ /* Queue Size */ ];
	le16 used_event; /* Only if VIRTIO_F_EVENT_IDX */
};
```

驱动使用可用环形区为设备提供缓冲区：每个环形区中的元素都指向描述符链的头部。它只由驱动写入，由设备读取。

idx字段表示驱动将下一个描述符项放入环中的位置（将队列大小moudulo化）。从0开始，然后增加。

### 2.6.6.1 驱动要求：Virtqueue可用环形区

驱动一定不能减少idx在virtqueue中（即不能“unexpose”buffer）

### 2.6.7 禁用已用buffer通知

如果未设置VIRTIO_F_EVENT_IDX功能位，可用环形区中的flags字段为驱动提供了一种粗略的机制在驱动想要给到设备信息的时候，他不会被通知当buffer被使用。

否则的话,used_event是一种性能更高的替代方法，其中驱动将指定设备在被通知之前可以运行多久。

这两种通知的方法都不可靠，因为它们与设备并不同步，但它们都是有用的优化。

### 2.6.7.1 驱动要求：禁用已用buffer通知

如果未设置VIRTIO_F_EVENT_IDX：（翻译注：原文是is not negotiated ，这块翻译不准确）
- 驱动需要设置flag为0或1.
- 驱动可能设置flag为1，通知设备不需要通知。

否则，已设置VIRTIO_F_EVENT_IDX：
- 驱动flag必须设置为0
- 驱动可能使用used_event告诉设备不需要获取通知，直到设备将used_event指定的索引项写入已用环形区（等效地，直到所用环中的idx达到used_event的值+1）

设备必须处理驱动给出的spurious的通知

### 2.6.7.2 设备要求:禁用已用buffer通知

如果未设置VIRTIO_F_EVENT_IDX：（翻译注：感觉写反）
- 设备必须忽略used_event
- 再设备写入一个描述符索引到已用环形区中后：
	- 如果flag设置为1，设备不应该发送通知
	- 如果flag设置为0，设备必须发送通知

否则，已协商VIRTIO_F_EVENT_IDX：
- 设备必须忽略flags的低位
- 再设备写入一个描述符索引到已用环形区中后：
	- 如果idx属性在已用环形区（索引决定了描述符的位置）被放入了used_event。这个设备必须发送通知
	- 否则设备不需要发送通知

例子：如果used_event是0，那么设备使用VIRTIO_F_EVENT_IDX将发送一个used buffer notification到驱动在第一个buffer被使用了（一直循环到第65536个buffer）。

### 2.6.8 Virtqueue已用环形区

已用环形区内存结构：
```
struct virtq_used {
	#define VIRTQ_USED_F_NO_NOTIFY 1
	le16 flags;
	le16 idx;
	struct virtq_used_elem ring[ /* Queue Size */];
		le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */
	};
	/* le32 is used here for ids for padding reasons. */
	struct virtq_used_elem {
	/* Index of start of used descriptor chain. */
	le32 id;
	/* Total length of the descriptor chain which was used (written to) */
	le32 len;
};
```
已用环形区是设备在使用完buffer后返回buffer的地方：它只由设备写入，由驱动程序读取。

每一个元素在环形区都是一对：id表示描述符链的头部元素（这与之前虚拟机放置在可用环形区中的条目相匹配）。 并且len代表了写入buffer所有字节数。

注：len 对于使用不受信任的缓冲区的驱动特别有用：如果驱动不知道设备到底写了多少，驱动必须事先将缓冲区归零，以确保不会发生数据泄漏。 举个例子，一个网络驱动可能处理接收的buffer到未授权的用户空间应用中，如果网络设备不去覆盖的写那块buffer，这可能会将已释放内存的内容泄漏到别的应用程序。

idx字段代表了设备将在环形区中放置下一个描述符的位置（modulo队列大小）。开始是0，之后增加。

### 2.6.8.1 遗留接口：Virtqueue已用环形区

略

### 2.6.8.2 设备要求：Virtqueue已用环形区

设备必须再更新已用idx之前设置len。

设备必须写入len字节数到描述符，在更新使用的idx之前，从第一个设备可写缓冲区开始。

设备可能会写入大于len的字节数到描述符

注：在某些潜在的错误情况下，设备可能不知道缓冲区的哪些部分被写入了。这就是为什么len被允许低估的原因：这比驱动认为未初始化的内存在没有被重写时被重写要好。

### 2.6.8.3 驱动要求：Virtqueue已用环形区

驱动必须不能假设设备可写缓冲区中的数据超过第一个len字节，并且应该忽略这些数据。

### 2.6.9 顺序使用描述符

有些设备总是按照它们可用的顺序使用描述符。这些设备可以提供VIRTIO_F_IN_ORDER功能位。如果设置了这个功能位，它会允许设备仅通过写入一个已用环形区条目（id最后一个缓冲区的描述符链的头条目相对应）来通知驱动使用这一个缓冲区。

然后，设备根据批次的大小在环中向前跳跃。因此，它会增加按批次大小使用idx。

驱动需要查找使用的id并计算批次处理的大小，以便能够前进到设备将写入下一个已用环形区的位置。

这样可用环形的偏移和已用环形区相匹配，下一个也是同样匹配的。

被跳过的缓冲区(没有使用过得已用环形区)会被假设为已经被设备使用过了(读或写)。

### 2.6.10 禁用可用Buffer通知

设备可以不接收available buffer notification，其方式类似于驱动不接收used buffer notification一样，再2.6.7所述。

### 2.6.10.1 驱动要求：禁用可用Buffer通知

当开始分配内存给已用环形区时，驱动必须初始化标志位为0再已用环形区中。

如果未设置VIRTIO_F_EVENT_IDX：
- 驱动必须忽略avail_event值
- 在驱动写入设备描述符index到可用环形区后：
	- 如果flag是1，驱动不应该发送通知
	- 如果flag是0，启动应该发送通知。

否则，如果设置了VIRTIO_F_EVENT_IDX：
- 驱动必须忽略flags低位。
- 在驱动写入设备描述符index到可用环形区后：
	- 如果idx属性在可用环形区（索引决定了描述符的位置）被放入了avail_event。这个驱动必须发送通知
	- 否则驱动不应该发送通知。

### 2.6.10.1 设备要求：禁用可用Buffer通知

如果未设置VIRTIO_F_EVENT_IDX：
- 设备需要设置flag为0或1.
- 设备可能设置flag为1，通知设备不需要通知。

否则，已设置VIRTIO_F_EVENT_IDX：
- 设备flag必须设置为0
- 设备可能使用avail_event告诉驱动不需要获取通知，直到驱动将avail_event指定的索引项写入可用环形区（等效地，直到所用环中的idx达到avail_event的值+1）

设备必须处理驱动给出的spurious的通知

### 2.6.11 Virtqueues操作助手

Linux kernal源码已更可用的形式包含上述定义，内容在include/uapi/linux/virtio_ring.h中。这是IBM和Red Hat根据（3-clause）BSD许可证显式授权的，以便它可以被所有其他项目自由使用。并且也在virtio_queue.h有定义。

### 2.6.12 Virtqueue操作

virtqueue操作有两个部分：
- 向设备提供新的可用缓冲区
- 处理设备中的已用缓冲区

注：举个例子，最简单的virtio网络设备有两个virtqueue：传输virtqueue（the transmit virtqueue）和接收virtqueue（the receive virtqueue）。驱动将向外(设备可读)网络包放到传输virtqueue,并且释放他们当他们被用掉了。最简单的，向内网络(设备可写)缓存将被放到接收virtqueue，并且处理它们，当他们被用掉。

下面是在更详细地使用split virtqueue格式时，对于这两个操作部分的需求。

### 2.6.13 提供buffer到设备

驱动提供buffer到一个设备的virtqueues步骤如下：
1. 驱动将buffer放入描述符表中的空闲描述符中（请参阅2.6.5 Virtqueue描述符表）。
2. 驱动将描述符链头部的索引插入可用环形区的下一个环条目中。
3. 步骤1，2可能重复多次，如果是批次数据的话。
4. 驱动分配内存(performs a suitable memory barrier)，以确保设备在下一步之前看到更新的描述符表和可用的环。
5. 可用的idx自增，依据于添加到可用环形区的描述符链头的数量。
6. 驱动分配内存，以确保它在检查禁用通知操作之前更新idx字段。
7. 驱动发送available buffer notification到设备，如果需要的话(翻译注：看之前设置过了的话)。

注意，上面的代码没有对可用的环缓冲区wrapping around采取预防措施：这是不可能的，因为环形区区的大小与描述符表的大小相同，所以步骤（1）将防止出现这种情况。

此外，最大队列大小为32768（16位最大值），因此16位idx值始终可以区分满缓冲区和空缓冲区。

以下是每个阶段的详细要求。

### 2.6.13.1 buffer放入描述表

一个buffer由1到多个设备可写且物理连续的元素组成(最少一个)，且其后面跟着1到多个设备可读且物理连续的元素(最少一个)。这个算法将映射它到描述表，来形成描述链。

对于每一个buffer元素，简称b:

1. 获得下一个空闲的描述表实体，简称d
2. 设置d.addr为b的物理地址开头
3. 设置d.len为b的长度
4. 如果b是设备可写的，设置d.flags为VIRTQ_DESC_F_WRITE，否则为0
5. 如果在b后面插入了元素
	1. 设置d.next为新插入元素
	2. 设置VIRTQ_DESC_F_NEXT到d.flags里面去

在实践中，d.next通常被用于连接空闲的描述符，并在开始映射之前单独计数以检查是否有足够的空闲描述符。

### 2.6.13.2 更新可用环形区

描述链的head是上面算法的第一个d，即描述表的下标映射到buffer的第一部分(翻译注：意思就是head是上面描述链的头)，缺乏经验的驱动实现可能会这样写：
```
avail->ring[avail->idx % qsz] = head;
```

然而，通常驱动可能会添加多个描述链在它更新idx(在指针对于设备可见)之前。所以它通常会保存一个计数器来记录驱动这会添加了多少描述链：
```
avail->ring[(avail->idx + added++) % qsz] = head;
```

### 2.6.13.3 更新idx

idx经常增加，并且小于65536：
```
avail->idx += added;
```

一旦驱动更新了可用的idx，就会公开描述符及其内容。设备可以立即访问驱动创建的描述符链和它们引用的内存。

### 2.6.13.3.1 驱动要求：更新idx

驱动必须在idx更新之前分配好相应的内存，以确保设备看到最新的副本。

### 2.6.13.4 通知设备

设备通知方法是总线限定的(bus-specific)，但是通常这种方法代价很高，因此，如2.6.10节所述，如果设备不需要这些通知，它可能会禁止这些通知。

### 2.6.13.4.1 驱动要求：通知设备

驱动必须在读flags或读avail_event之前分配好相应的内存，以避免丢失通知。

### 2.6.14 从设备接收可用buffer

一旦设备使用了描述符所指的缓冲区（根据virtqueue和设备的功能，读取或写入或读写）。它就会向驱动发送一个used buffer notification ，如第2.6.7节所述。

注：为了获得最佳性能，驱动可能会在处理已用的环形区时禁用used buffer notification，但请注意在清空环形区和重新启用通知时缺少通知的问题。这通常是通过在重新启用通知后重新检查已用缓冲区来处理的。

```
virtq_disable_used_buffer_notifications(vq);
for (;;) {
	if (vq->last_seen_used != le16_to_cpu(virtq->used.idx)) {
		virtq_enable_used_buffer_notifications(vq);
		mb();
		if (vq->last_seen_used != le16_to_cpu(virtq->used.idx))
			break;
		virtq_disable_used_buffer_notifications(vq);
	}
	struct virtq_used_elem *e = virtq.used->ring[vq->last_seen_used%vsz];
	process_buffer(e);
	vq->last_seen_used++;
}
```

### 2.7 packed virtqueues

Packed virtqueues是另一种可读写且压缩的virtqueue布局。即由host和虚机读写的内存。

VIRTIO_F_RING_PACKED功能位开启，Packed virtqueues才能使用。

每一个Packed virtqueues支持2^15实体。

对于当前传输，virtqueues位于由驱动分配的虚机内存中。每个Packed virtqueue由三部分组成：

- 描述环(Descriptor Ring) : 描述符的区域
- 驱动事件抑制器(Driver Event Suppression) : 驱动区域
- 设备事件抑制器(Device Event Suppression) : 设备区域

其中描述符环依次由描述符组成，并且每个描述符可以包含以下部分：

- buffer的id(Buffer ID)
- 元素地址(Element Address)
- 元素长度(Element Length)
- 标识位(Flags)

（翻译注：和split virtioqueues区别貌似在于一个两个队列，一个是环形）

一个buffer由0到多个设备可写且物理连续的元素组成(最少一个)，且其后面跟着0到多个设备可读且物理连续的元素(最少一个)。

当驱动想要向设备发送这样的缓冲区时，它会将至少一个描述缓冲区元素的可用描述符写入描述符环中。描述符通过存储在描述符中的buffer ID与buffer区相关联。

然后驱动程序通知设备，当设备处理完缓冲区后，它将一个已用设备描述符（包括缓冲区ID）写入描述符环（覆盖以前可用的驱动描述符），并发送一个used event notifion。

描述符环是循环使用的：驱动按顺序将描述符写入环中。之后到达环的末端，下一个描述符被放置在环的头部。一旦描述环中满了，驱动将停止发送新的请求。并等待设备开始处理描述符，并在产生新的驱动描述符之前写出(write out)一些已用描述符。

类似地，设备按顺序从环中读取描述符，并检测驱动提供的描述符是否可用。当描述符的处理完成时，设备会将已用描述符写回环中。

注：在读取驱动描述符并顺序启动它们之后，设备可能按顺序来处理它们，已用设备描述符按其处理完成的顺序写入（翻译注：应该是按顺序写入描述环）。

设备事件抑制器(Device Event Suppression)数据结构是设备只写的。它包括用于减少设备事件数量的信息，即，向设备发送较少的available buffer notifications。

驱动事件抑制器(Driver Event Suppression)数据结构是驱动只读的。它包括用于减少驱动事件数量的信息，即，向驱动发送较少的used buffer notifications。

### 2.7.1 驱动和设备warp计数器

每个驱动和设备都需要在内部维护一个初始化为1的warp计数器。

由驱动维护的计数器称为Driver Ring Wrap Counter，驱动每次使环中的最后一个描述符变为可用时都会更改此计数器的值（在最后一个描述符变为可用之后才会去改该值）。

由设备维护的计数器称为Divice Ring Wrap Counter，设备每次使环中的最后一个描述符变为已用时都会更改此计数器的值（在最后一个描述符变为已用之后才会去改该值）。

很容易看出，在驱动中的Driver Ring Wrap Counter与设备中的Divice Ring Wrap Counter处理相同的描述符，或者当使用了所有可用的描述符时匹配。

要将描述符标记为可用和已用，驱动和设备都使用以下两个标志：

```
#define VIRTQ_DESC_F_AVAIL (1 << 7)
#define VIRTQ_DESC_F_USED (1 << 15)
```

为了标记一个描述符是可用的，驱动设置VIRTQ_DESC_F_AVAIL在flag中来匹配Driver Ring Wrap Counter。它也设置VIRTQ_DESC_F_USED来匹配内部值（即不匹配内部的 Driver Ring Wrap Counter）

为了标记一个描述符是已用的，设备设置VIRTQ_DESC_F_USED在flag中来匹配Divice Ring Wrap Counter。它也设置VIRTQ_DESC_F_AVAIL来匹配内部值（即不匹配内部的 Divice Ring Wrap Counter）

所以对于可用描述符和已用描述符来说，VIRTQ_DESC_F_AVAIL and VIRTQ_DESC_F_USED是不同的。

注：this observation is mostly useful for sanity-checking as these，且是必要不充分条件。 举个例子：所有描述符都是赋0初始化的，为了检测已用和可用描述符，驱动和设备可以跟踪最后观察到的VIRTQ_DESC_F_USED/VIRTQ_DESC_F_AVAIL。用代码来检查VIRTQ_DESC_F_AVAIL/VIRTQ_DESC_F_USED设置位变化也是可能的。

### 2.7.2 轮询可用和已用描述符

设备和驱动描述符的写入通常可以重新排序。但是每一方（驱动和设备）只需要轮询（或测试）内存中的一个位置：下一个设备描述符，在他们之前处理的设备描述符之后，按循环的顺序。

有时，设备只需要在处理一批多个可用描述符后写出一个已用的描述符。如下面更详细描述的，当使用描述符链或按顺序使用描述符时，可能会发生这种情况。在这种情况下，设备写出已使用的描述符，其buffer id为组中最后一个描述符。在处理已用描述符之后，设备和驱动随后在环中向前跳过组中剩余描述符的数量（翻译注：应该是按批处理的话，跳过批次的数量），直到处理（读取驱动和写入设备）下一个使用的描述符。


### 2.7.3 写标志位（Write Flags）

在可用描述符中，在Flags中的VIRTQ_DESC_F_WRITE标识用于将描述符标记为对应于缓冲区的只读或写元素。

```
/* This marks a descriptor as device write-only (otherwise device read-only). */
#define VIRTQ_DESC_F_WRITE 2
```

在已用描述符中，此位用于指定设备是否已将数据写入缓冲区的部分。

### 2.7.4 元素地址和长度（Element Address and Length）

在可用描述符中，Element Address对应于缓冲区元素的物理地址。Length 假定是物理连续的，它存了元素的长度。

在已用描述符中，Element Address是用不到的，Element Length指定了被设备初始化的buffer的长度。

Element Length是为没有设置VIRTQ_DESC_F_WRITE标志的已用描述符保留的，并且是被驱动忽略的。

### 2.7.5 散列支持(Scatter-Gather Support)

有些驱动需要一个东西来为请求提供一个包含多个缓冲区元素的列表（也称为SGL,散列表）。有两个特性支持这一点：描述符链和间接描述符。

如果两个特性都没有被驱动使用，那么每个缓冲区在物理上是连续的，是只读的还是只写的完全由一个描述符描述。

虽然不常见（大多数实现要么使用非间接描述符创建所有列表，要么始终使用单个间接元素）。但如果两个特性都已被设置，只要每个列表只包含给定类型的描述符，则在环中混合间接和非间接描述符是有效的。

散列表只能应用于可用描述符。单个的已用描述符对应整个表。

设备通过传输特定值和/或设备特定值限制表中描述符数量。如果不受限制，则列表中描述符的最大数量是virtqueue大小。


### 2.7.6 下一个Flag: 描述链

Packed ring格式允许驱动使用多个描述符向设备提供散列表。 并为除最后一个可用描述符外的所有描述符设置VIRTQ_DESC_F_NEXT在标志中。

```
/* This marks a buffer as continuing. */
#define VIRTQ_DESC_F_NEXT 1
```

Buffer ID包含在表中的最后一个描述符中。

驱动总是在表的其余部分被写入环之后将表中的第一个描述符设置可用。这保证了设备永远不会看到环中的散列表。（翻译注：这样设备应该只能看到第一个描述符）

注：所有的标志位，包括VIRTQ_DESC_F_AVAIL, VIRTQ_DESC_F_USED, VIRTQ_DESC_F_WRITE必须被设置/清除在链中的每一个描述符，不只是第一个。

设备只写出一个已用描述符来表示整个描述链，然后根据表中描述符的数量向前跳。驱动需要跟踪对应于每个buffer ID的表大小，以便能够跳到设备写入下一个使用的描述符的位置。

举个例子，如果描述符的使用顺序与它们可用的顺序相同，这将导致已用描述符覆盖列表中的第一个可用描述符，下一个列表的已用描述符覆盖下一个列表中的第一个可用描述符，etc。

VIRTQ_DESC_F_NEXT保留再已用描述符中，并且是被驱动忽略的。

### 2.7.7 间接标识位：散列表

一些设备通过并发地发送大量的大请求而受益。VIRTIO_F_INDIRECT_DESC功能位允许这一点。为了增加环的容量，驱动可以存一个内存中的间接描述符表(table of indirect descriptors)，并插入一个描述符在主virtqueue(flags&VIRTQ_DESC_F_INDIRECT要为true)，这样内存就可以包含非直接描述表。addr和lenrefer对应间接描述符表中的地址和长度。

```
/* This means the element contains a table of descriptors. */
#define VIRTQ_DESC_F_INDIRECT 4

```


间接描述符表内存分布如下(len为指向的描述符的长度，是一个变量）：

```
struct pvirtq_indirect_descriptor_table {
	/* The actual descriptor structures (struct pvirtq_desc each) */
	struct pvirtq_desc desc[len / sizeof(struct pvirtq_desc)];
};
```

第一个描述符位于间接描述符表的开头，其他间接描述符紧随其后。VIRTQ_DESC_F_WRITE flags位是间接表中描述符的唯一有效标志。其他的则被保留，并被设备忽略。buffer ID也被保留，并被设备忽略。

在具有VIRTQ_DESC_F_INDIRECT设置的描述符中，VIRTQ_DESC_F_WRITE被保留，并被设备忽略。

### 2.7.8 顺序使用描述符

有些设备总是按照它们可用的顺序使用描述符。这些设备可以提供VIRTIO_F_IN_ORDER功能位。如果设置了这个功能位，它会允许设备仅通过写出一个已用的描述符（其buffer ID与批中的最后一个描述符对应）来通知驱动使用一批缓冲区。

然后根据表中描述符的数量向前跳。驱动需要跟踪对应于每个buffer ID的表大小，以便能够跳到设备写入下一个使用的描述符的位置。

这将导致已用描述符覆盖批中的第一个可用描述符，下一批的已用描述符覆盖下一批中的第一个可用描述符，等等。

### 2.7.9 多buffer请求

有些设备将多个缓冲区组合起来作为处理单个请求的一部分。


### 2.7.10 驱动和设备禁止事件

在很多使用used and available buffer notifications都会有大量开销。为了减少这种开销，每个virtqueue包含两个相同的结构，用于控制设备和驱动之间的通知。

驱动禁止事件通知的数据结构是对于设备只读的，并且控制设备发送给驱动的已用缓冲区通知。

设备禁止事件通知的数据结构是对于驱动只读的，并且控制驱动发送给设备的可用缓冲区通知。

Event Suppression结构如下：

1. 描述环事件改变flags:

```
/* Enable events */
#define RING_EVENT_FLAGS_ENABLE 0x0
/* Disable events */
#define RING_EVENT_FLAGS_DISABLE 0x1
/*
 * Enable events for a specific descriptor
 * (as specified by Descriptor Ring Change Event Offset/Wrap Counter).
 * Only valid if VIRTIO_F_RING_EVENT_IDX has been negotiated.
 */
#define RING_EVENT_FLAGS_DESC 0x2
/* The value 0x3 is reserved */
```

2. 描述环事件改变offset:当事件flags设置位描述符特定事件，环内的偏移量（以描述符大小为单位）。事件将仅在分别可用/已用此描述符时触发。
3. 描述环事件改变counter:当事件flags设置位描述符特定事件，环内的偏移量（以描述符大小为单位）。事件仅在计数器与此值匹配且描述符分别可用/已用时触发。

在写出一些描述符之后，设备和驱动都需要参考相关的结构，以确定是否应该分别使用一个available buffer notification。

### 2.7.10.1 数据结构大小与内存对齐

virtqueue的每个部分在guest的内存中物理上是连续的，并且具有不同的对齐要求。

字节上的内存对齐和大小要求，virtqueue的每个部分的摘要如下表所示：

Virtqueue Part	 Alignment 	Size
Descriptor Ring 	16 		16∗(Queue Size)
Device Event Suppression 4 4
Driver Event Suppression 4 4

- Alignment列给出virtqueue的每个部分的最小对齐方式。
- Size列给出virtqueue每个部分的总字节数
- 队列大小对应于virtqueue中的最大描述符数,队列大小值不必是2的幂。

### 2.7.11 驱动要求：Virtqueues

驱动必须确保每个virtqueue部分的第一个字节的物理地址，为上表中指定对齐值的倍数。

### 2.7.12 设备要求：Virtqueues

设备必须按照描述符在环中出现的顺序开始处理它们。设备必须开始按完成顺序将描述符写入环中。一旦描述符写入启动，设备可能会重新排序。

### 2.7.13 Virtqueue描述符格式

可用描述符指的是驱动发送到设备的buffer。adder是物理地址，描述符用id字段来辨别buffer。

```
struct pvirtq_desc {
	/* Buffer Address. */
	le64 addr;
	/* Buffer Length. */
	le32 len;
	/* Buffer ID. */
	le16 id;
	/* The flags depending on descriptor type. */
	le16 flags;
};

```

描述符环是zero-initialized。

### 2.7.14 禁止通知数据结构格式

以下结构用于减少驱动和设备之间发送的通知数。

```
struct pvirtq_event_suppress {
	le16 {
		desc_event_off : 15; /* Descriptor Ring Change Event Offset */
		desc_event_wrap : 1; /* Descriptor Ring Change Event Wrap Counter */
	} desc; /* If desc_event_flags set to RING_EVENT_FLAGS_DESC */
	le16 {
		desc_event_flags : 2, /* Descriptor Ring Change Event Flags */
		reserved : 14; /* Reserved, set to 0 */
	} flags;
}
```

### 2.7.15 设备要求：Virtqueue描述表

设备不得写device-readable buffer，设备不应该读device-writable buffer。设备不能使用描述符，除非它观察到描述符标志中的VIRTQ_DESC_F_AVAIL位被更改。设备在更改其标志中的VIRTQ_DESC_F_USED后，不能再对它操作。

### 2.7.16 驱动要求：Virtqueue描述表

驱动不能更改描述符，除非它观察到描述符标志中的VIRTQ_DESC_F_USED位被更改。驱动在更改其标志中的VIRTQ_DESC_F_AVAIL位后，不能再对它操作。当通知设备时，驱动必须设置next_off和next_wrap以匹配下一个尚未提供给设备的描述符。驱动可以发送多个 available buffer notifications，而不向设备提供任何新的描述符。

### 2.7.17 驱动要求：散列表

- 驱动创建的描述符列表不能超过设备允许的长度。
- 驱动创建的描述符列表不能超过队列大小。
- 这意味着描述符列表中的循环是被禁止的！
- 驱动必须要先放置device-readable的描述符，后面再放置device-writable描述符
- 驱动不能依赖于设备使用更多的描述符来写出列表中的所有描述符。驱动必须确保环中有足够的空间容纳整个列表，然后才能使列表中的第一个描述符对设备可用。
- 在构成列表的所有后续描述符可用之前，驱动不得使列表中的第一个描述符可用。

### 2.7.18 设备要求：散列表

- 设备必须按VIRTQ_DESC_F_NEXT标识顺序来使用描述符，顺序与驱动提供的顺序相同。
- 设备可能会限制它在列表中允许的buffer数量。

### 2.7.19 驱动要求：间接描述符

- 除非已协商VIRTIO F_INDIRECT，否则驱动不得设置VIRTIO_F_INDIRECT_DESC标记。驱动不能在间接描述符中设置任何描述符除了 DESC_F_WRITE。
- 驱动创建的描述符链不能超过设备允许的长度。
- 驱动不能写在散列表中的有VIRTQ_DESC_F_NEXT的直接描述符，当DESC_F_INDIRECT被设置。

### 2.7.20 Virtqueue操作

virtqueue操作分为两部分：向设备提供新的可用buffer和处理设备中已用buffer。

下面是在更详细地使用Packet virtqueue格式时这两个部分的要求。

### 2.7.21 提供buffer到设备

驱动向设备的virtqueues提供buffer，如下步骤：

1. 驱动将buffer放入描述符环中的空闲描述符中。
2. 驱动准备memory barrier，以确保在检查禁止通知之前更新描述符。
3. 如果通知没被禁止，驱动通知设备新的可用buffer。

以下是每个阶段的详细要求。

### 2.7.21.1 将可用buffer放入描述符环

对于每个Buffer,b:

1. 获取描述表中的下一个描述符,d
2. 获取下一个可用缓冲区id
3. 设置d.addr为b的物理地址开头
4. 设置d.len为b的len
5. 设置d.id为buffer id
6. 计算flags：
	1. 如果b是device-writable的，设置VIRTQ_DESC_F_WRITE为1，否则0
	2. 设置VIRTQ_DESC_F_AVAIL为当前的Driver Ring计数器。
	3. 设置VIRTQ_DESC_F_USED在保留位中
7. 执行memory barrier以确保描述符已初始化
8. 设置d.flags去被计算的flags中。
9. 如果d是最后一个描述符在描述环中，拨动Driver Ring计数器
10. 否则，递增d以指向下一个描述符

这使得单个描述符buffer可用。但是，一般来说，驱动可以使用一批描述符作为单个请求的一部分。在这种情况下，它会延迟更新第一个描述符的flags，直到其他描述符初始化之后。

一旦驱动更新了描述符flags字段，就会公开描述符及其内容。设备可以访问描述符和驱动创建的任何以下描述符以及它们所引用的内存。

### 2.7.21.1.1 驱动要求：更新flags

在标记更新之前，驱动程序必须准备memory barrier，以确保设备看到最新的副本。

### 2.7.21.2 发送Available Buffer Notifications

设备通知的实际方法是bus-specific，但通常会很昂贵。因此，如果设备不需要这些通知，它可能会禁止这些通知。使用禁止通知结构，包括设备区域在2.7.14中。

### 2.7.14.3 实现例子

下面这是驱动代码例子，它不会尝试减少available buffer notifications的数量，它也不支持VIRTIO_F_RING_EVENT_IDX功能。
```
/* Note: vq->avail_wrap_count is initialized to 1 */
/* Note: vq->sgs is an array same size as the ring */
id = alloc_id(vq);

first = vq->next_avail;
sgs = 0;
for (each buffer element b) {
	sgs++;

	vq->ids[vq->next_avail] = -1;
	vq->desc[vq->next_avail].address = get_addr(b);
	vq->desc[vq->next_avail].len = get_len(b);

	avail = vq->avail_wrap_count ? VIRTQ_DESC_F_AVAIL : 0;
	used = !vq->avail_wrap_count ? VIRTQ_DESC_F_USED : 0;
	f = get_flags(b) | avail | used;
	if (b is not the last buffer element) {
		f |= VIRTQ_DESC_F_NEXT;
	}

	/* Don't mark the 1st descriptor available until all of them are ready. */
	if (vq->next_avail == first) {
		flags = f;
	} else {
		vq->desc[vq->next_avail].flags = f;
	}

	last = vq->next_avail;
	vq->next_avail++;
	if (vq->next_avail >= vq->size) {
		vq->next_avail = 0;
		vq->avail_wrap_count \^= 1;
	}
}

vq->sgs[id] = sgs;
/* ID included in the last descriptor in the list */
vq->desc[last].id = id;
write_memory_barrier();
vq->desc[first].flags = flags;
memory_barrier();

if (vq->device_event.flags != RING_EVENT_FLAGS_DISABLE) {
	notify_device(vq);
}

```

### 2.7.21.3.1 驱动要求：发送Available Buffer Notifications

在读取占据设备区域的事件抑制结构之前，驱动程序必须准备memory barrier。否则，可能导致无法发送available buffer notifications。

### 2.7.22 从设备接收已用buffer

一旦设备有已用buffer指向描述符（根据virtqueue和设备的性质，可以向它们读或写，或读写），它向驱动发送used buffer notification，详见第2.7.14节。

注：为了获得最佳性能，驱动在处理已用buffer时可能会禁用used buffer notification,但请注意清空环和重新启用used buffer notification之间缺少通知的问题。这通常通过在重新启用通知后重新检查更多已用缓冲区来处理（翻译注：应该是每次清空的时候重启下就行）：


```
/* Note: vq->used_wrap_count is initialized to 1 */
vq->driver_event.flags = RING_EVENT_FLAGS_DISABLE;

for (;;) {
	struct pvirtq_desc *d = vq->desc[vq->next_used];
	/*
	 * Check that
	 * 1. Descriptor has been made available. This check is necessary
	 * if the driver is making new descriptors available in parallel
	 * with this processing of used descriptors (e.g. from another thread).
	 * Note: there are many other ways to check this, e.g.
	 * track the number of outstanding available descriptors or buffers
	 * and check that it's not 0.
	 * 2. Descriptor has been used by the device.
	 */
	flags = d->flags;
	bool avail = flags & VIRTQ_DESC_F_AVAIL;
	bool used = flags & VIRTQ_DESC_F_USED;
	if (avail != vq->used_wrap_count || used != vq->used_wrap_count) {
		vq->driver_event.flags = RING_EVENT_FLAGS_ENABLE;
		memory_barrier();
		/*
		 * Re-test in case the driver made more descriptors available in
		 * parallel with the used descriptor processing (e.g. from another
		 * thread) and/or the device used more descriptors before the driver
		 * enabled events.
		 */
		flags = d->flags;
		bool avail = flags & VIRTQ_DESC_F_AVAIL;
		bool used = flags & VIRTQ_DESC_F_USED;
		if (avail != vq->used_wrap_count || used != vq->used_wrap_count) {
			break;
		}

		vq->driver_event.flags = RING_EVENT_FLAGS_DISABLE;
	}

	read_memory_barrier();
	/* skip descriptors until the next buffer */
	id = d->id;
	assert(id < vq->size);
	sgs = vq->sgs[id];
	vq->next_used += sgs;
	if (vq->next_used >= vq->size) {
		vq->next_used -= vq->size;
		vq->used_wrap_count \^= 1;
	}

	free_id(vq, id);
	process_buffer(d);
}
``` 

### 2.7.23 驱动通知

有时需要驱动向设备发送available buffer notification。

当未协商VIRTIO_F_NOTIFICATION_DATA时，此通知涉及将virtqueue数目发送到设备（方法取决于传输）。

为了帮助进行这些优化，在协商VIRTIO F_NOTIFICATION_DATA时，设备的驱动通知包括以下信息：

vqn ： 需要被通知的VQ号
next_off： 将写入下一个可用环条目的环内的偏移量。当VIRTIO_F_RING_PACKED未被协商，这个值指可用索引的15个最低有效位。当When VIRTIO_F_RING_PACKED被协商，这个值指得是offset（以描述符项为单位）在将写入下一个可用描述符的描述符环中。
next_wrap：Wrap计数器，当VIRTIO_F_RING_PACKED被协商，这个warp计数器指向了下一个可用描述符，当VIRTIO_F_RING_PACKED未被协商，这个值是可用索引的最高有效位（15位）。

注：驱动可以发送多个通知，即使不再提供任何可用的缓冲区。当VIRTIO_F_NOTIFICATION_DATA被协商，这些通知将具有相同的next-off和next-wrap值。


1-2章完