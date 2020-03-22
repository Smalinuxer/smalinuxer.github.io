---
layout: post
title: virtio spec v1.1 中文翻译(0) 大纲
category: store
keywords: store,virtio,virtualization
---



## 1. 写在前面的话

virtio这个东西，本人其实并不是非常熟悉，项目需要，开始学习spec。翻译可能并不尽善尽美，也会加入我自己的一些理解。希望您能多多体谅。

如果有翻译出错的地方可以联系我的邮箱。如果您有更多更好的建议，也可以联系本人。或有任何问题，您可以联系我的私人邮箱 [smalinuxer@gmail.com](mailto://smalinuxer@gmail.com)

翻译内容中，带有mark字段的为翻译者所著，表示自身不理解的地方

原文档地址为：[virtio-v1.1.pdf](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf)
 
版权归属于OASIS所有，翻译版本只为学习交流，请勿商务用途，翻译版本版权归属于[jiaqizho.github.com](http://jiaqizho.github.com)


## 2.翻译导读
1-2章链接 : [virtio spec v1.1 中文翻译(1) 1-2章 前言、Virtio设备的基本设施](http://jiaqizho.github.io/2020/03/11/virtio-spec-translation-1.html)

## 3. 翻译进度

- [x] 第一章：前言(Introduction)
- [x] 第二章：Virtio设备的基本设施(Basic Facilities of a Virtio Device)
    - [x] 2.1 设备状态属性(Device status field)
    - [x] 2.2 功能位(Feature Bits)
    - [x] 2.3 通知(Notifications)
    - [x] 2.4 设备配置空间(Device Configuration Space)
    - [x] 2.5 Virtqueues
    - [x] 2.6 Split Virtqueues
    - [x] 2.7 Packed Virtqueues
- [x] 第三章：初始化和设备操作(General Initialization And Device Operation)
    - [ ] 3.1 设备初始化(Device Initialization)
    - [ ] 3.2 设备操作(Device Operation)
    - [ ] 3.3 设备清空(Device Cleanup)
- [ ] 第四章：Virtio Transport Options
    - [ ] 4.1 Virtio Over PCI Bus
    - [ ] 4.2 Virtio Over MMIO
    - [ ] 4.3 Virtio Over Channel I/O
- [ ] 第五章：Device Types
    - [ ] 5.1 Network Device
    - [ ] 5.2 Block Device
    - [ ] 5.3 Console Device
    - [ ] 5.4 Entropy Device
    - [ ] 5.5 Traditional Memory Balloon Device
    - [ ] 5.6 SCSI Host Device
    - [ ] 5.7 GPU Device
    - [ ] 5.8 Input Device
    - [ ] 5.9 Crypto Device    
    - [ ] 5.10 Socket Device
- [ ] 第六章：Reserved Feature Bits
    - [ ] 6.1 Driver Requirements: Reserved Feature Bits
    - [ ] 6.2 Device Requirements: Reserved Feature Bits
    - [ ] 6.3 Legacy Interface: Reserved Feature Bits
- [ ] 第七章：Conformance
    - [ ] 7.1 Conformance Targets
    - [ ] 7.2 Clause 1: Driver Conformance
    - [ ] 7.3 Clause 14: Device Conformance
    - [ ] 7.3 Clause 27: Legacy Interface: Transitional Device and Transitional Driver Conformance
- [ ] 其他
    - [ ] Appendix A. virtio_queue.h
    - [ ] Appendix B. Creating New Device Types

    