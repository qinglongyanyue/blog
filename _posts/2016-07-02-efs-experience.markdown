---
layout: default
title:  "EFS正式版体验"
date:   2016-07-02
categories: filesystem
---

## 1. 背景

EFS preview一年多，终于发布了正式版，必须赶紧体验下。

目前发布的地点主要在三个地方提供服务：
1. EU (Ireland)
2. US East (N. Virginia)
3. US West (Oregon)

我先择了US West区域，首先申请EFS，流程非常简单清晰。

follow流程，默认先择在3个AZ都启动挂载点，可以测试下EFS的3个AZ是全AA还是主备。

第2步，在每个AZ申请一个EC2 VM，规格m3.medium，ubuntu 14.04（其实2个不同的VM，2个AZ就够了）. 为了省钱，选择了带4GB SSD，3.7G内存，中等带宽的网络。

## 2. AZ内部EFS性能体验

默认挂载参数如下：

```
ip-172-31-19-0% sudo grep /efs/az1 /proc/mounts
us-west-2a.fs-ae1bef07.efs.us-west-2.amazonaws.com:/ /efs/az1 nfs4 rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=172.31.19.0,local_lock=none,addr=172.31.19.28 0 0
```

默认参数似乎看不到太多的信息，不过它暂时只支持nfs4.1，不知是哪种考虑？

### 2.1 AZ内部DirectIO性能

从AZ1的VM连接AZ1的挂载点

- 最小写延迟
```
ip-172-31-19-0% sudo fio -filename=/efs/az1/test -direct=1 -iodepth 1 -thread -rw=write -ioengine=libaio -bs=4k -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=100M

IOPS:110
latency:9ms
```
- 最小读延迟
```
ip-172-31-19-0% sudo fio -filename=/efs/az1/test -direct=1 -iodepth 1 -thread -rw=read -ioengine=libaio -bs=4k -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=100M

IOPS:402
latency:2.5ms
```

- 最大写带宽
```
sudo fio -filename=/efs/az1/test -direct=1 -iodepth 16 -thread -rw=write -ioengine=libaio -bs=512K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

带宽：40MB/s
```

- 最大读带宽
```
ip-172-31-19-0% sudo fio -filename=/efs/az1/test -direct=1 -iodepth 16 -thread -rw=read -ioengine=libaio -bs=512K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

带宽：40MB/s
```

- 最大写IOPS
```
ip-172-31-19-0% sudo fio -filename=/efs/az1/test -direct=1 -iodepth 16 -thread -rw=randwrite -ioengine=libaio -bs=4K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

IOPS：310
```

- 最大读IOPS
```
ip-172-31-19-0% sudo fio -filename=/efs/az1/test -direct=1 -iodepth 1024 -thread -rw=randread -ioengine=libaio -bs=4K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

IOPS：7449
```

### 2.2 跨AZ DirectIO体验
从AZ1的VM连接AZ2的挂载点，同样的测试模型

- 最小写延迟
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 1 -thread -rw=write -ioengine=libaio -bs=4k -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=100M

IOPS:110
lantency:9ms
```
- 最小读延迟
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 1 -thread -rw=read -ioengine=libaio -bs=4k -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=100M

IOPS:389
lantency:2.56ms
```

- 最大写带宽
```
sudo fio -filename=/efs/az2/test -direct=1 -iodepth 16 -thread -rw=write -ioengine=libaio -bs=512K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

带宽：40MB/s
```

- 最大读带宽
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 16 -thread -rw=read -ioengine=libaio -bs=512K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

带宽：40MB/s
```

- 最大写IOPS
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 128 -thread -rw=randwrite -ioengine=libaio -bs=4K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

IOPS：310
```

- 最大读IOPS
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 1024 -thread -rw=randread -ioengine=libaio -bs=4K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

IOPS：6671
```

### 2.2 跨AZ DirectIO体验
从AZ2的VM连接AZ2的挂载点，同样的测试模型（测试下EFS是否多个AZ可以同时写的？是否AA？）

- 最小写延迟
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 1 -thread -rw=write -ioengine=libaio -bs=4k -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=100M

IOPS:110
latency:9ms
```
- 最小读延迟
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 1 -thread -rw=read -ioengine=libaio -bs=4k -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=100M

IOPS:365
lantency:2.7ms
```

- 最大写带宽
```
sudo fio -filename=/efs/az2/test -direct=1 -iodepth 16 -thread -rw=write -ioengine=libaio -bs=512K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

带宽：40MB/s
```

- 最大读带宽
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 16 -thread -rw=read -ioengine=libaio -bs=512K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

带宽：40MB/s
```

- 最大写IOPS
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 128 -thread -rw=randwrite -ioengine=libaio -bs=4K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

IOPS：310
```

- 最大读IOPS
```
ip-172-31-19-0% sudo fio -filename=/efs/az2/test -direct=1 -iodepth 16 -thread -rw=randread -ioengine=libaio -bs=4K -numjobs=1 -runtime=30 -group_reporting -name=mytest  -size=1000M

IOPS：7351
```

## 小节

|client VM| EFS target |write latency |read latency | write bandwith|read bandwith | write IOPS| read iops|
| ---- | ----| ---- | ---- | ---- | ---- | ---- | ---- |
|AZ1|AZ1|8.72ms|2.27ms|40MB/s|40MB/s|310|7460|
|AZ1|AZ2|8.7ms|2.75ms|40MB/s|40MB/s|310|7421|
|AZ2|AZ2|8.78ms|2.54ms|40MB/s|40MB/s|310|7351|
|AZ2|AZ1|8.81ms|2.58ms|40MB/s|40MB/s|310|7460|


发现2个现象：

- 随机读的性能要比顺序读要好。（随机的时候，后端多盘一起服务，顺序时候落到一个盘）
- 由于QoS的限制，很多数据不太好测试出来，上限的差异非常小

## 同时访问
都是同一个EFS，主要看client端的IOPS

- 测试1:AZ1访问AZ1，同时AZ2访问AZ2
- 测试2:AZ1访问AZ2，同时AZ2访问AZ1
- 测试3:AZ1访问AZ1，同时AZ1访问AZ2

|test| AZ1 latency|AZ2 latency|AZ1 IOPS | AZ2 IOPS|
| ---- | ----| ---- | ---- | ---- |
|1read|2.26ms|2.56ms|4005|2934|
|2read|2.55ms|2.69ms|2702|4165|
|3read|2.3ms|2.66ms|4308|2441|
|1write|8.9ms|9ms|160|156|
|2write|9ms|9ms|159|156|
|3write|8.9ms|9ms|160|156|

所有操作都针对同一个FS的同一个文件。

这个数据非常纠结，差距很小，不过几个点比较清楚：

- 无论怎么玩，写IOPS性能都非常非常搓，说明EFS确实定位大带宽场景
- 无论怎么写，写的延迟，IOPS，带宽都完全一样，似乎可以说明EFS的跨AZ不是AP模式（不过由于写延迟太大，就算是AP模式我们也难以感知到）
- 读似乎有一定的不对称，多个点同时读的时候，AZ1那个点提供出来的读性能似乎要高点，看起来似乎读归属到啦AZ1（难道有读Cache？）