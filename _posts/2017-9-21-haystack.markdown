---
layout: default
title:  "Haystack小文件优化存储分析"
date:   2017-09-21
categories: filesystem
---

haystack是facebook比较久的论文了，专业针对互联网图片的存储，对于posix小文件的存储也很具有参考价值，重点分析下他的单盘布局这块。

## 1 haystack storage

## 1.1 磁盘布局

![]({{ site.baseurl }}/assets/haystack-1.png

haystack的store file：

- 底层是XFS（建议我们也切换到XFS，大文件用它比较好）
- 创建并分配一个巨大的file，比如默认100GB
- superblock存放整体信息，比如这个file的大小，归属之类的吧，建议预留4KB
- 每个needle代表一个完成的条目，包括数据和元数据；needle详情见表格
- 所有的IO + 元数据随着时间顺序append到这个大文件中

整体逻辑比较简单清晰，关键我们要看这个表格是如何索引到的呢？内存数据结构是什么样的呢？

内存中key为（Key,Alternate Key），通过这个key唯一确定一个needle，主要是方便同一个照片存放多个版本清晰度，用alternate key来区分不同清晰度，多个不同清晰度的图片的key是同一个。

内存中的表到底存放些啥？还得看看后文。。

### 1.1.1 读流程

读请求下发的req包括volid（大文件的id），Key, Alternate Key以及cookie（一个放在URL中的随机数），流程如下：

- 拿到请求，确认是否已经删除，没删除在内存映射表中找到key对应的needle的偏移和长度
- 下盘读取数据即可

### 1.1.2 写流程

上传一个照片的流程，写请求的参数包括volid，key，alternate key，cookie，data：

- 直接将需要的信息按照布局的格式持久化写入log file的最后
- 更新内存中的映射表
- 不支持更新写入，只能用新的覆盖旧的数据，如果新旧数据都在同一个volid中，那么新数据在log的后面，旧数据在前面（没有设置标记的动作）
- 如果新的数据写入到一个新的逻辑卷（新的volid），直接修改应用的映射关系

### 1.1.3 删除流程

- 检查内存数据结构中needle信息，如果不存在，直接返回错误
- 直接设置needle的flag为删除
- 设置内存映射表的flag为删除

## 1.2 index file布局

为了优化reboot的性能，设计一个index file的结构，如果没有这个东西，每次重启都需要扫描整个log file重建内存映射，效率太低。

![]({{ site.baseurl }}/assets/haystack-2.png

![]({{ site.baseurl }}/assets/haystack-3.png

这个index file就是为了快速重建内存映射而设计的。

- 每个log file都对应一个index file
- 内存映射定期checkpoint到index file中
- index file的layout如上图所示

index file的时候动作如下：

- 写入新图片时，直接写入log file，返回，然后异步写入index file
- 删除图片时，设置log file的flag为删除状态，并不修改index file
- 此时会造成2个问题：1）needle存在，但是index中没有，此时称为孤儿needle；2）index中不知道有哪些needle被删除啦

重启流程如下：

- 直接在log file的尾部才可能存在孤儿，很容易扫出来
- 扫出来之后加入到index file
- 基于index file重建内存映射表，但是此时不知道删除状态
- 读或删除的时候下盘再check状态，如果是删除，在内存设置这个条目为删除状态

## 2. 本地FS选型

由于haystack的访问特征，需要一个支撑大文件内部的随机小IO性能好的FS，最终选型为XFS。

- XFS的extent技术使得他的大文件映射表很容易放在内存中
- XFS提供高效的文件预分配，减轻了碎片的情况
- 使用XFS之后，haystack就不需要去管理磁盘空间，很多情况下一次磁盘IO就可以把数据读上来（依赖第一条）
- haystack使用1GB的extent和256KB的RAID条带

## 3. 优化方案

### 3.1 空间回收

- 在线回收被删除的图片
- 直接创建一个新的volid，将旧的有效复制复制过来
- 在复制的过程中，2个log file同时接受删除请求，旧的接受读请求
- 复制过程中同时刷新内存结构，但是index file如何同步？

### 3.2 内存优化方案

- 为了减少内存，对于删除的needle，我们直接配置offset为0，而不使用flag字段
- 内存不放cookie
- 传一个图片生成4个照片，每个照片包括2个key，8BYTE（相同） + 4BYTE（4*4=16B），以及数据大小2ByteB(4*2=8B);然后每个图片2B（4*2=8B）的overhead（HASH的overhead）；一共40B，平均下来每个图片只需要消耗10B。