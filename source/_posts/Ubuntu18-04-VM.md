---
title: Ubuntu18.04 VM
date: 2019-05-28 11:12:24
categories:
- 虚拟机
tags: 
- ubuntu18.04
- install
- VM
---

<center><font size=4 color="red">配置Ubuntu 18.04基础镜像</font></center>

<!--more-->

# 在Windows上安装Ubuntu18.04虚拟机

## 下载Ubuntu18.04的镜像

下载链接
https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/18.04/

下载的版本
ubuntu-18.04.2-live-server-amd64.iso

## 在VMware里安装虚拟机

安装过程省略,这里主要讲解开启虚拟机后的一些设置

修改镜像地址,改成阿里云的.
https://mirrors.aliyun.com/ubuntu/

![](aaa.jpg)

选择：Use An Entire Disk And Set Up LVM

![](03.jpg)

![](05.jpg)

![](07.jpg)

选择安装openSSH,如果这个没有安装,进入到系统后,要执行以下命令安装
`$ apt install openssh-server`

其它的都不用配置,直接done

## 开启虚拟机root用户权限

设置root的密码并开启root权限

```shell
$ sudo passwd
$ su root
```

修改root的权限配置文件
`vi /etc/ssh/sshd_config`

修改如下

![](01.jpg)

重启ssh服务
`service ssh restart`

在Xshell里使用root用户登录

![](04.jpg)

![](06.jpg)

## 设置静态ip和DNS

编辑配置文件

`/etc/netplan/50-cloud-init.yaml`

```shell
network:
    ethernets:
        ens33:
         addresses: [192.168.10.129/24]
         gateway4: 192.168.10.2
         nameservers:
           addresses:
           - 114.114.114.114
           - 114.114.115.115
    version: 2
```

使其生效
`netplan apply`

配置文件里的ens33是和`ip a`里得到的一致

![](02.jpg)

> 注意这个配置文件的格式,一个空格都可能导致配置出错

启动service
`systemctl start systemd-resolved.service`

设置为开机自启动
`systemctl enable systemd-resolved.service`

重启
`reboot`

试验网络
`ping www.baidu.com`

检查ip地址是否是自己设置的静态ip
`ip a`

做个快照
`鼠标右键虚拟机-->快照-->拍摄快照`

## 交换空间的设置

关闭交换空间
`swapoff -a`

避免开机启动交换空间
`vi /etc/fstab`
修饰掉带swap的一行

关闭防火墙
`ufw disable`

关机
`shotdown -h now`

开机做个快照
`鼠标右键虚拟机-->快照-->拍摄快照`

## 安装docker

使用APT安装docker,方法来源:

https://funtl.com/zh/service-mesh-kubernetes/%E5%AE%89%E8%A3%85%E5%89%8D%E7%9A%84%E5%87%86%E5%A4%87.html#%E4%BD%BF%E7%94%A8-apt-%E5%AE%89%E8%A3%85-docker

```shell
# 更新软件源
sudo apt-get update
# 安装所需依赖
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# 安装 GPG 证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 新增软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 再次更新软件源
sudo apt-get -y update
# 安装 Docker CE 版
sudo apt-get -y install docker-ce
```

验证
```shell
root@yytubuntu:~# docker version
Client:
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77
 Built:             Sat May  4 02:35:57 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.6
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.8
  Git commit:       481bc77
  Built:            Sat May  4 01:59:36 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

## 配置加速

方法来源地址:

https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors?accounttraceid=9b8e4567-a62c-4723-95e0-6fcb5b44c87f

逐条输入以下命令

```shell
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://veoukc4z.mirror.aliyuncs.com"]
}
EOF

sudo systemctl daemon-reload

sudo systemctl restart docker
```

查看是否配置成功

输入`docker info`,出现以下信息就是配置成功了

```shell
Registry Mirrors:
https://veoukc4z.mirror.aliyuncs.com/
```

## 使用pip安装docker-compose

获取get-pip.py方法来源,安装这个是为了安装pip:

https://pip.pypa.io/en/stable/installing/

获取get-pip.py命令

```shell
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
```

要安装get-pip.py需要python,get-pip.py不能和本机自带的python相和谐,所以要自己安装python

使用ppa安装python

安装ppa

`sudo add-apt-repository ppa:deadsnakes/ppa`

安装python

`apt install python3.7`

安装get-pip.py

`python3.7 get-pip.py`

更改pip的镜像源

更改方式的地址来源:

https://mirrors.tuna.tsinghua.edu.cn/help/pypi

使用临时镜像升级pip,提高网速

```shell
$ pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pip -U
```

升级pip到最新版本后进行配置

```shell
$ pip install pip -U
$ pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

安装docker-compose

`$ pip install -U docker-compose`

删除get-pip.py

```shell
$ ls
$ rm -fr get-pip.py
```

关机

`$ reboot`

开机做个快照

## 克隆

克隆不是新创建一个虚拟机,而是在原有虚拟机上做改变,不会影响最初的base虚拟机

`鼠标右键-->管理-->克隆`

## 导出OVF格式文件

导出的文件删除了DVD驱动,导入到新的VMware后要添加DVD驱动和镜像


