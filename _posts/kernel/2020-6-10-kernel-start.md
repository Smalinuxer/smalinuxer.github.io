---
layout: post
title: Kernel之旅(一) 编译&调试环境搭建
category: kernel
keywords: kernel
---


## 0. 前言

好久没认真的看开源项目了，给波兰人当小弟当久了，想着是时候开始看看kernel代码了。

主要是想看看fs、block device、kernel driver相关的内容。

这次绝对不弃坑，顺带看到block device后，把fio的坑填了。之前不填坑，主要是因为被spdk的沙雕iodepth设计震惊了。 

## 1. 编译kernel

首先，下载源码。目前kernel版本已经到了5.x和6.x，我觉着也没必要用那么高，就随便选了一个4.5。该版本略稳定

推荐使用清华的源，国内会快一些。
```$shell
$ wget https://mirror.tuna.tsinghua.edu.cn/kernel/v4.x/linux-4.5.tar.xz
$ xz -d linux-4.5.tar.xz
$ tar -xvf linux-4.5.tar
```

接着可能需要安装以下工具(可能不全)，不同发行版使用不同方式安装即可。如果编译报错也不要慌，说明你装的工具不全。

```$shell
libncurses5-dev libssl-dev bison flex libelf-dev gcc make openssl libc6-dev
```

接着设置一下编译选项
```$shell
$ cd linux-4.5
$ make menuconfig 
```

我这里设置了开启了fs和block device相关的debug选项。同时最好开启下

```$shell

Kernel hacking  ---> 
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        [*]   Provide GDB scripts for kernel debugging

```

这样可以打开GDB的python扩展。

接着直接编译

```$shell
$ make -j24
```

kernel 4.5版本你可能会遇到如下错误
```$shell
kernel/built-in.o: In function `update_wall_time':
/home/ubuntu/linux-4.10.4/kernel/time/timekeeping.c:2088: undefined reference to `____ilog2_NaN'
Makefile:969: recipe for target 'vmlinux' failed
make: *** [vmlinux] Error 1
```

即有关于____ilog2_NaN找不到的的错误。

这时，创建一个kernel-log.patch的文件，写入如下内容
```c
diff --git a/include/linux/log2.h b/include/linux/log2.h
index ef3d4f67118c..c373295f359f 100644
--- a/include/linux/log2.h
+++ b/include/linux/log2.h
@@ -16,12 +16,6 @@
 #include <linux/bitops.h>
 
 /*
- * deal with unrepresentable constant logarithms
- */
-extern __attribute__((const, noreturn))
-int ____ilog2_NaN(void);
-
-/*
  * non-constant log of base 2 calculators
  * - the arch may override these in asm/bitops.h if they can be implemented
  *   more efficiently than using fls() and fls64()
@@ -85,7 +79,7 @@ unsigned long __rounddown_pow_of_two(unsigned long n)
 #define ilog2(n)				\
 (						\
 	__builtin_constant_p(n) ? (		\
-		(n) < 1 ? ____ilog2_NaN() :	\
+		(n) < 2 ? 0 :			\
 		(n) & (1ULL << 63) ? 63 :	\
 		(n) & (1ULL << 62) ? 62 :	\
 		(n) & (1ULL << 61) ? 61 :	\
@@ -148,10 +142,7 @@ unsigned long __rounddown_pow_of_two(unsigned long n)
 		(n) & (1ULL <<  4) ?  4 :	\
 		(n) & (1ULL <<  3) ?  3 :	\
 		(n) & (1ULL <<  2) ?  2 :	\
-		(n) & (1ULL <<  1) ?  1 :	\
-		(n) & (1ULL <<  0) ?  0 :	\
-		____ilog2_NaN()			\
-				   ) :		\
+		1 ) :				\
 	(sizeof(n) <= 4) ?			\
 	__ilog2_u32(n) :			\
 	__ilog2_u64(n)				\
diff --git a/tools/include/linux/log2.h b/tools/include/linux/log2.h
index 41446668ccce..d5677d39c1e4 100644
--- a/tools/include/linux/log2.h
+++ b/tools/include/linux/log2.h
@@ -13,12 +13,6 @@
 #define _TOOLS_LINUX_LOG2_H
 
 /*
- * deal with unrepresentable constant logarithms
- */
-extern __attribute__((const, noreturn))
-int ____ilog2_NaN(void);
-
-/*
  * non-constant log of base 2 calculators
  * - the arch may override these in asm/bitops.h if they can be implemented
  *   more efficiently than using fls() and fls64()
@@ -78,7 +72,7 @@ unsigned long __rounddown_pow_of_two(unsigned long n)
 #define ilog2(n)				\
 (						\
 	__builtin_constant_p(n) ? (		\
-		(n) < 1 ? ____ilog2_NaN() :	\
+		(n) < 2 ? 0 :			\
 		(n) & (1ULL << 63) ? 63 :	\
 		(n) & (1ULL << 62) ? 62 :	\
 		(n) & (1ULL << 61) ? 61 :	\
@@ -141,10 +135,7 @@ unsigned long __rounddown_pow_of_two(unsigned long n)
 		(n) & (1ULL <<  4) ?  4 :	\
 		(n) & (1ULL <<  3) ?  3 :	\
 		(n) & (1ULL <<  2) ?  2 :	\
-		(n) & (1ULL <<  1) ?  1 :	\
-		(n) & (1ULL <<  0) ?  0 :	\
-		____ilog2_NaN()			\
-				   ) :		\
+		1 ) :				\
 	(sizeof(n) <= 4) ?			\
 	__ilog2_u32(n) :			\
 	__ilog2_u64(n)				\
```

然后执行
```shell
$ patch -i patch.diff
```

在提示输入后，依次输入**include/linux/log2.h**  ， **tools/include/linux/log2.h**即可。

再重新编译。编译完成后，会在目录下生成一个**vmlinux**文件



## 2. initramfs 构建 

initramfs(initram file system)是提供给pid为1的进程(init进程)使用的文件系统，init进程负责启动系统后续的工作，包括定位、挂载“真正的”根文件系统设备（如果有的话）。然后执行 /sbin/init程序完成系统的后续初始化工作。

initramfs处理流程如下

1. boot loader 把内核以及 Initramfs 文件加载到内存的特定位置。
2. 内核判断Initramfs的文件格式，如果是cpio格式。
3. 将Initramfs的内容释放到rootfs中。
4. 执行Initramfs中的sbin/init文件，执行到这一点，内核的工作全部结束，完全交给init文件处理。

这里推荐用busybox来制作一个initramfs

```shell 
$ wget https://busybox.net/downloads/busybox-1.31.1.tar.bz2
$ tar -jxvf  busybox-1.31.1.tar.bz2
$ cd busybox-1.31.1/
```

编译
```shell 
$ make menuconfig
```

然后记得把Settings中的Build static binary (no shared libs)选上
```shell
$ make -j24
$ make install
```

install生成在 **_install**目录下。这时候开始构建initramfs。initramfs的所有文件系统操作都是在内存中，因为没有挂载实际的盘，所以之后我可能还需要改。
```shell
$ mkdir initramfs
$ cd initramfs
$ cp ../_install/* -rf ./
$ mkdir dev proc sys
$ sudo cp -a /dev/{null, console, tty, tty1, tty2, tty3, tty4} dev/
$ rm linuxrc
```

再编辑一下init进程
```shell
$ vim init
$ chmod a+x init
```

具体内容如下
```shell script
#!/bin/busybox sh         
mount -t proc none /proc  
mount -t sysfs none /sys  

exec /sbin/init
```

打包
```shell
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

## 3. qemu调试 

qemu的安装我就不多说了，这个软件比内核还复杂。功能十分强大。

当你装好qemu之后，执行如下（更改下相应文件路径）
```shell 
$ qemu-system-x86_64 -s -kernel ./linux-4.5/arch/x86/boot/bzImage -initrd busybox-1.31.1/initramfs.cpio.gz -S
```

之后gdb挂载远端target
```shell
$ gdb vmlinux
(gdb) target remote localhost:1234
```

挂载了之后，就可以设置断点开始调试了。

```shell
(gdb) b start_kernel
(gdb) c
```

另外有人推荐重新编译安装以下gdb，需要注释一下gdb/remote.c，否则弹错误。高版本目前我倒是没出现过。

## 4. 后续

后续还会详细的讲讲挂载一块ssd，发一个文件io请求，然后跟踪调试过程~