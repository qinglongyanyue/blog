---
layout: default
title:  "PureStorage SDS存储分析"
date:   2016-05-25
categories: filesystem
---

上市存储公司Pure Storage 3月在旧金山发布了cloud的存储平台——[FlashBlade](https://www.purestorage.com/company/news-and-events/press/ps_introduce_fb.html)

一只以为Pure Storage是个全闪存阵列厂商，没有太多关注，今天女神发了link让我看看才知道。

必须要分析下。

### FlashBlade是啥
看名字，似乎是硬件的名称，但是内涵却很丰富。总结几个点：

- 主打非结构化数据（全闪存阵列主打结构化数据和VM卷）。
- 全flash，重点阐述了大数据的场景。
- 灵活的的scale－out架构，跟Qumulo打架了。

顺手鄙视了传统NAS和scale－out NAS的问题：

- 第一当然是介质，flash pk disk必须秒杀
- 元数据扩展性有限，不管是NAS还是分布式NAS，元数据管理都是大难题
- 把数据切割到各个nodes上，存在数据孤立和管理麻烦的问题（看这意思，是要把数据都放一个节点？）

号称FlashBlade能够“big, fast and simple”的解决所有的三个问题。难道有核武器？

### 主打场景
由于大数据的爆发，数据的分析将会变得非常复杂。很多人都在想办法如何搞定这个问题，包括更高的性能，更简单的client连接，当然必须要绕过复杂厚重的传统存储stack。

- 数据科学和工程：比如设计飞机，需要海量的数据来分析。
- 分析一切：IOT导致数据再一次爆发，如何分析这些数据需要更大更快的平台
- Cloud－native的应用：容器，微服务，devops。这些基于容器和微服务的新兴应用需要无限&实时的扩展性，需要run在海量，无限扩展的数据平台上。比如几乎所有的google的应用都run在GFS存储平台上。

### FlashBlade架构
标准的scale－out架构，标准硬件 ＋ 软件的组合。只不过这个标准的硬件也是Pure Storage出品，暂时应该不支持第三方硬件。

- Blade：包括NVM ＋ flash，包括8TB和52TB两种容量
- 灵活可扩展的软件：支持EC，去重，加密，NFS，S3；提到一个关键技术，所有的level都是用同一套可扩展元数据引擎，同一套GC。（这么说来，应该类似Azure或者Google的架构？）
- 灵活的网络：默认使用software-defined 40GE（SDN？）；灵活性体现在software-defined QoS，能够根据优先级发网络包。

高性能，简单易用；价格实惠，不到1$/GB的采购成本。

隔靴搔痒，没讲到啥本质上去。

### 几个核心特点

- Big
 - 横向扩展：容量，IO性能，元数据性能，带宽，支持的Client连接数全部线性扩展（blade为单位扩展，8TB／52TB）。
 - 高密度：起步可以<100TB，一个4U最多能装1.6PB
- Fast
 - 全Flash，一个4U能出15GB/s带宽；一致的低延迟。
 - 元数据可扩展：支持每秒100w的对象创建，能支撑20年。支撑绿色升级，完全不中断业务。
- Simple
 - 一个系统，巨大无比的地址空间，多协议访问（主要是file和S3），简单的UI
 - file和对象：底层是分布式对象存储，通过file和对象来访问，未来也很容易添加新的协议。

预计2016年下半年发货。

### 小节
从描述来看主要还是面向第三平台的产品，定位非常类似EMC的ECS，HDS的HCP，HP的StoreAll。

不过它是全Flash，跟Qumulo可能更像一些。

与Ceph，Hedvig，CohoData还不太一样，那些更偏重块存储一些。这个东东就完全是个非结构化海量数据的存储。
