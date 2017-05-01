---
layout: default
title:  "shadowshocks配置说明"
date:   2016-11-09
categories: main
---


1.准备条件
--
一台境外的虚拟机（买便宜的vps，或者用境外的docker）  
shadowsocks

2.shadowsocks介绍  
--
"A fast tunnel proxy that helps you bypass firewalls"  
(一个可穿透防火墙的快速代理)  
github link--- https://github.com/shadowsocks/shadowsocks-libev  

3.vm上运行shadowsocks  
--
apt-get update 
apt-get install python-pip 
pip install shadowsocks 
pip 是 python 下的方便安装的工具，类似 apt-get。  
写一个配置文件保存为etc/shadowsocks.json，作为服务启动配置   
  
    {
       "server":"my_server_ip",
       "server_port":8388,
       "local_address": "127.0.0.1",
       "local_port":1080,
       "password":"mypassword",
       "timeout":300,
       "method":"aes-256-cfb",
       "fast_open": false
    }
启动ssserver -c /etc/shadowsocks.json -d start   

`坑：`这里有个比较坑的地方，aws默认的安全组只开了22端口的ssh权限，http无法连接，需要去设置一下aws ec2的安全组。

4.启动客户端  
--  
下载windows客户端（mac，android类似）可以去github下载  
设置时候对照着2中的json填，也支持直接导入json  
接下来享受google，facebook，youtube~~

5.将shadowsocks跑在容器中，脱离vm限制  
--
可以去境外买个docker服务，run在容器中，便宜点  
灵雀云有海外版，以前会送优惠券，现在没了  
daocloud`海外机房似乎还没有上线  
时速云有海外机房提供容器服务，需要充值  
  
`小坑：`容器run时候需要设置一下command参数（json的配置），否则无法正常启动  

6.我搭建的代理信息
-  
最后搭建了一个跑在aws中的容器，在daocloud中托管。  
如果你想使用，请到github下载客户端（pc or Android），配置按照下面的来即可  

       "server_port":2016,
       "local_address": "52.35.90.109",
       "local_port":any,
       "password":"suzhen336",
       "timeout":300,
       "method":"aes-256-cfb",

## shadowshocks转HTTP

- Linux：
[Linux 命令行下使用 Shadowsocks 代理](https://mritd.me/2016/07/22/Linux-%E5%91%BD%E4%BB%A4%E8%A1%8C%E4%B8%8B%E4%BD%BF%E7%94%A8-Shadowsocks-%E4%BB%A3%E7%90%86/)

- MAC：
[为终端设置Shadowsocks代理](http://droidyue.com/blog/2016/04/04/set-shadowsocks-proxy-for-terminal/index.html)

- 配置MAC的host OS与virtual box的ubuntu数据互通
[MAC与ubuntu挂同一个目录](http://askubuntu.com/questions/456400/why-cant-i-access-a-shared-folder-from-within-my-virtualbox-machine)

- [把MAC配置成代理服务器](http://www.akmumu.com/2015/07/07/367.html)，这样MAC上的VM都能通过这个代理访问国外网站啦]

- [Virtual Box安装CoreOS](https://coreos.com/os/docs/latest/booting-on-virtualbox.html)

从此世界畅通啦。。