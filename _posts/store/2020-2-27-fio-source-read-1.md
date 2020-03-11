---
layout: post
title: fio源码阅读(一) fio的启动
category: store
keywords: store,virtio,virtualization
---


## 1. 前言

fio(flexible IO tester)一直是存储领域一个非常好用且重要的工具。它是由大佬Jens Axboe编写的io测试工具。这位大佬不但是kernel中block layer的maintainer。还是一位有着自己维基百科的程序员。

整体fio有350+文件，行数早已突破10万行。各个模块都已经做得非常完善，从最基本io统计、lantency统计，再到对于ZBD，对于盘的steadystate自有计算，保存任务进度，再到gfio可视化图形。都可以说非常的强大。

所以博主只是盲人摸象一般，从其启动开始分析脉络，梳理出运行时候发生什么，之后再写一篇lantency计算相关的代码分析。目前我使用的版本是3.15。

相关资源
- fio github : [https://github.com/axboe/fio](https://github.com/axboe/fio)
- fio 官方文档 ： [FIO’s documentation!](https://fio.readthedocs.io/en/latest/)

## 2. fio的启动

一般运行fio，都不用gfio，而是使用fio.c编译的fio

```$c

int main(int argc, char *argv[], char *envp[])
{
	// 检查下fio运行的环境
	if (initialize_fio(envp))
		return 1;
    ... 
	// 创建主线程的Thread Specific Key Data
	if (fio_server_create_sk_key())
		goto done;
 	
 	// 解析传入的参数，大部分参数会变成init.c中的全局变量
	if (parse_options(argc, argv))
		goto done_key;

	fio_time_init();
  

	// 检查是以server-client运行的模式，还是本地跑
	if (nr_clients) {
		set_genesis_time();

		if (fio_start_all_clients())
			goto done_key;
		ret = fio_handle_clients(&fio_client_ops);
	} else
		ret = fio_backend(NULL);
    ...
}

```

可以看到fio运行是可以使用server-client的模式进行运行的，如果使用该模式，fio的后端和前端可以分别运行，即fio服务器可以在“被测设备”上生成I / O工作负载，同时由另一台计算机上的客户端控制。

在参数可以这样写，支持TCP/IP和socket运行
```$xslt
fio --server=ip:hostname,4444
或
fio --server=sock:/tmp/fio.sock
```
这里我们跳过server-client模式，直接分析本地运行的情况，那么他会调用fio_backend(NULL);

## 2 fio_backend的启动

 

