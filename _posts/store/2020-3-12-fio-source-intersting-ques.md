---
layout: post
title: fio源码-lantency的最大有效数据
category: store
keywords: store,fio
---

## 1. 前言

当有人问你，你知道fio默认能接受最大的io延迟是多长时间吗？

我可以明确地告诉你：**34359738368纳秒**，也就是**34.3秒**左右。

当单次IO超过这个大小的时候，fio的lantency统计将还是会记录这个数。

![奇怪的知识增加了.jpg](https://s1.ax1x.com/2020/03/12/8miWqI.jpg)

那么这个结论是如何得出来的呢

## 2. lantency细则

## 2.1 fio面临的问题

不要急，在讲lantency统计相关的内容之前，先看看这么一个问题：**给你两个区间，一个小区间[0,N],一个大区间[0,M]，要求小区间映射到大区间范围内所有数。比如，小区间的1映射倒了大区间的[1,10]。写出两个函数，分别是大区间的数转化为小区间的数，和小区间的数转化为大区间的范围。**

这么一看这个问题其实很简单，只需要用大区间除以小区间（M/N），在[0,N]的小区间内，每一个数都可以映射M/N个大区间的数。这一步叫做**等比映射**。

那么把题目变一下，加上这么一个条件：**要求小区间映射范围曲线增加**。比如小区间[0,N]，数字0映射了大区间[0,2]，数字10可能就需要映射大区间[50,200]，其中要映射150个数而不是3个数。

这个题目其实就是fio所面临的问题，在测试场景中，fio这个工具需要统计lantency的分布，并且io lantency总是趋近于稳定且高速的。所以如果按等比映射的话，那么就会变成以下情况：

* 要么增大内存，整一个非常大的数组存精度足够小的lantency数据，这样计算起来会非常麻烦。
* 要么降低精度，节省内存，映射的lantency范围变大，这样统计出来的lantency percentiles会变得不准确。

## 2.2 fio的处理方式

首先我们可以看下fio lantency的小区间是多大的
```
#define FIO_IO_U_PLAT_BITS 6
#define FIO_IO_U_PLAT_VAL (1 << FIO_IO_U_PLAT_BITS)
#define FIO_IO_U_PLAT_GROUP_NR 29
#define FIO_IO_U_PLAT_NR (FIO_IO_U_PLAT_GROUP_NR * FIO_IO_U_PLAT_VAL)

// 存放lantency的二维数组
uint64_t io_u_plat[DDIR_RWDIR_CNT][FIO_IO_U_PLAT_NR];
```
对于io_u_plat的第一维我们可以不用在意，因为这个是存放不同统计type的，其实也就三种，SLAT,CLAT和LAT。

第二维的长度是 **29 * (1 << 6)** ，看起来这数字非常奇怪，29是怎么来的，为什么需要1 << 6。之后会详细的讲到这个长度。

有了数组的长度（也就是问题中小区间的长度），首先我们来看看小区间映射大区间的方法。
```c
static unsigned int plat_val_to_idx(unsigned long long val)
{
	unsigned int msb, error_bits, base, offset, idx;

	/* Find MSB starting from bit 0 */
	if (val == 0)
		msb = 0;
	else
		msb = (sizeof(val)*8) - __builtin_clzll(val) - 1;

	/*
	 * MSB <= (FIO_IO_U_PLAT_BITS-1), cannot be rounded off. Use
	 * all bits of the sample as index
	 */
	if (msb <= FIO_IO_U_PLAT_BITS)
		return val;

	/* Compute the number of error bits to discard*/
	error_bits = msb - FIO_IO_U_PLAT_BITS;

	/* Compute the number of buckets before the group */
	base = (error_bits + 1) << FIO_IO_U_PLAT_BITS;

	/*
	 * Discard the error bits and apply the mask to find the
	 * index for the buckets in the group
	 */
	offset = (FIO_IO_U_PLAT_VAL - 1) & (val >> error_bits);

	/* Make sure the index does not exceed (array size - 1) */
	idx = (base + offset) < (FIO_IO_U_PLAT_NR - 1) ?
		(base + offset) : (FIO_IO_U_PLAT_NR - 1);

	return idx;
}
```

输入的参数是纳秒，算出来的idx是小区间的idx。它具体做了这么几件事
1. 计算MSB（最高有效位）,但其实是数字二进制长度减1。假如输入是300000纳秒，即18。
2. 当MSB小于等于FIO_IO_U_PLAT_BITS时，那么直接映射，这里FIO_IO_U_PLAT_BITS为6。那么[0,127]纳秒都是直接映射
3. 计算error_bits，error_bits是一个中间值
4. 计算base值，base值我理解为“组偏移”，把base展开是(msb - FIO_IO_U_PLAT_BITS + 1) << FIO_IO_U_PLAT_BITS。得到的是这个输入的数，将会在哪个小区间的组里面。
    1. 之前提到数组是 29 * (1 << 6) 可以理解为29个组
    2. 前两个组，除了第二个数都是直接映射即[0,2 * (1 << 6)]都为直接映射
    3. 那么(MSB - FIO_IO_U_PLAT_BITS + 1) << FIO_IO_U_PLAT_BITS可以理解成，我把这个数放在第(msb - FIO_IO_U_PLAT_BITS + 1) 组里面
5. 计算offset，这个offset是将数安插再组里面后，计算在当前组的offset。(val >> error_bits)指的是其最后6位（二进制的）。(FIO_IO_U_PLAT_VAL - 1)则是111111(二进制)也正是一个组的大小
6. 好有了第几组也有了组中的第几位就开始计算是否大于最大值，如果不大于最大组数那么直接相加。

画一个图来表示其的内存分布：
![fio-calc.png](https://s1.ax1x.com/2020/03/12/8mJ9fg.png)

那么回到开头问题：fio默认能接受最大的io延迟是多长时间？

可以通过结尾的三元表达式反推，当(base + offset) >= 29 * (1 << 6) - 1 时，那么将会存在最后一个位置。展开式可得

* (error_bits + 1) << (1 << 6) + offset > >= 29 * (1 << 6) - 1;
* offset最大为(1 << 6) - 1 ，那么可得 (error_bits + 1) << (1 << 6)  >= 29 * (1 << 6) .注这里减一最后再减
* 再把error_bits替换掉得到(MSB - 6 + 1) << (1 << 6)  >= 29 * (1 << 6)
* 之前又说道MSB是二进制长度减1。那么结果就是二进制长度为 (29 + 6) = 35的数。即**1 << 35 = 34359738368**

再来看看小区间反映射大区间的过程
```c
static unsigned long long plat_idx_to_val(unsigned int idx)
{
	unsigned int error_bits;
	unsigned long long k, base;

	assert(idx < FIO_IO_U_PLAT_NR);

	/* MSB <= (FIO_IO_U_PLAT_BITS-1), cannot be rounded off. Use
	 * all bits of the sample as index */
	if (idx < (FIO_IO_U_PLAT_VAL << 1))
		return idx;

	/* Find the group and compute the minimum value of that group */
	error_bits = (idx >> FIO_IO_U_PLAT_BITS) - 1;
	base = ((unsigned long long) 1) << (error_bits + FIO_IO_U_PLAT_BITS);

	/* Find its bucket number of the group */
	k = idx % FIO_IO_U_PLAT_VAL;

	/* Return the mean of the range of the bucket */
	return base + ((k + 0.5) * (1 << error_bits));
}

```

这个过程就简单的多，只需要解开当前所在的组，再计算出他的偏移，**偏移会作为最后六位(二进制)保留下来**。越大的数，丢失的精度将会越高。

但是现在的盘一般io延迟都非常低，六位(二进制)保留其实已经很准确啦。

再回到之前的问题，为什么是29和6呢？因为6位作为精度保存，29作为分组数。当然这个数可以根据不同硬件情况自由组合啦。只不过需要再代码里面改，重新编译即可~

## 3. 结尾

虽然增加了奇怪的知识点，但是这一切值得吗？

[![但这一切值得吗？.jpg](https://s1.ax1x.com/2020/03/12/8makdI.jpg)](https://imgchr.com/i/8makdI)

