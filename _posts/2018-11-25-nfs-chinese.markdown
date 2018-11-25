---
layout: default
title:  "NFS中文乱码问题分析"
date:   2018-11-25
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

- 检查windows NFS支持的编码格式如下，可见并不支持UTF-8

![]({{ site.baseurl }}/assets/nfs-chinese-2.png)

- 网上有个方案，建议下载新版本的windows NFS客户端，http://www.nihao001.com/archives/1574.html，安装之后发现无法解决问题，应该是需要NFS server支持4.1才能解决乱码问题

![]({{ site.baseurl }}/assets/nfs-chinese-3.png)

- 又找到另一个方案,https://zhuanlan.zhihu.com/p/46254792,新版本的windows客户端可以支持UTF8了，可惜NFS server都还不支持，走不下去

### NFS乱码问题解决方案

- 出SMB协议
- 出NFSv4.1协议
- 等windows NFS客户端升级支撑UTF8