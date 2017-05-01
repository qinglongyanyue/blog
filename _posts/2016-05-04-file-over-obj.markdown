---
layout: default
title:  "File over Object"
date:   2016-05-03
categories: filesystem
---

### 准备环境
- 用virtualbox在macbook中安装ubuntu14.04，配置好网络。

### 开通对象存储服务
- 我用的是阿里云的OSS，一键式开通。其他公有云完全类似，价格计算比较复杂，不过应该还是很便宜的。

- 买了一个40GB的存储包，半年5元。再加上流量费和API调用费，如果只是用来测试备份一些桌面数据，价格应该还算OK。

- 进入OSS管理界面，创建一个bucket就可以存放数据了。记得查看access keys，后面认证需要用到。

### 安装部署ossfs

- 直接github搜到ossfs（基于s3fs & FUSE），按照指南安装，过程非常顺利：
```
sudo apt-get update
sudo apt-get install gdebi-core
sudo gdebi your_ossfs_package
```

- 配置：
```
echo your_bucket_name:your_key_id:your_key_secret > /etc/passwd-ossfs
chmod 640 /etc/passwd-ossfs
mkdir /tmp/ossfs
ossfs your-bucket-name /tmp/ming -ourl=http://oss-cn-shenzhen.aliyuncs.com
```

- 使用：
```
cd /tmp/ming
mkdir mingliang
echo hello world > test
```

- 登录oss管理界面，立刻看到新创建的对象

- 登录浏览器，也可以直接通过url访问该对象：
`http://mingliang.oss-cn-shenzhen.aliyuncs.com/test`

- Posix语义可能有不完善，git clone直接失败；不适合用来做开发目录，适合用来存档，备份之类
```
#/mnt git clone https://github.com/aliyun/ossfs.git
Cloning into 'ossfs'...
fatal: Fetch attempted without a local repo
```
- 在阿里云的VM中体验更好（内部网络必须好啊），性能更好，git clone也能成功

### 感受
- 安装部署体验不错，非常简单快捷
- 一份数据可以用fs的方式访问，也可以http share出去，多端同步很有价值
- 默认是sync模式挂载的，ls，cd，touch，echo各个命令都很卡，不是一个FS的体验
- 折腾半天也没有把cache模式弄出来，只能忍受巨慢的速度啦
- 语义上会有不少欠缺，明确提到不支持硬link和原子rename
