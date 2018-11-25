---
layout: default
title:  "NFS中文乱码问题分析"
date:   2018-08-23
categories: filesystem
---

## NFS乱码问题分析

### Linux挂载乱码问题
- 直接在linux目录挂载NFS之后，使用中文创建文件或者目录，直接显示为？？？？，如下
```
#mkdir 测试
##ls
??????
```
- 这个问题主要是Linux没有配置UTF-8编码，如何配置，并重启bash即可解决
```
vi /etc/environment

add these lines...

LANG=en_US.utf-8
LC_ALL=en_US.utf-8
```
- 测试下
```
#mkdir 测试
##ls
测试
```

### Windows挂载乱码问题

- 上文Linux中用中文创建的目录，通过NFS在windows中显示继续为乱码

![]({{ site.baseurl }}/assets/nfs-chinese-1.png)