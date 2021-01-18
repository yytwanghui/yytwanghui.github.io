---
title: Ubuntu Install Oracle
date: 2019-05-29 16:54:48
categories:
- Oracle
tags: 
- ubuntu
- install
- oracle
---

<center><font size=4 color="red">Ubuntu 18.04 Docker 安装 Oracle Database</font></center>

<!--more-->

# 使用docker安装oracle

## 登录docker

```shell
docker login
```

相应的输入注册的docker用户名和密码，如果没有注册，先去docker官网注册

https://hub.docker.com

## 下载oracle镜像

拉取oracle镜像

```shell
docker pull deepdiver/docker-oracle-xe-11g
```

运行oracle

```shell
docker run -d -p 49160:22 -p 49161:1521 deepdiver/docker-oracle-xe-11g
```

查询oracle镜像是否已经存在

```shell
docker ps
```

显示如下就说明镜像已经存在了

```shell
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS              PORTS                                                      NAMES
cc6bc8d8aa17        deepdiver/docker-oracle-xe-11g   "/bin/sh -c 'sed -i …"   16 seconds ago      Up 11 seconds       8080/tcp, 0.0.0.0:49160->22/tcp, 0.0.0.0:49161->1521/tcp   hardcore_engelbart
```

将容器的名称重新命名为oracle

```shell
docker rename hardcore_engelbart oracle
```

进入oracle的容器

```shell
docker exec -it oracle bash
```

进入oracle

```shell
sqlplus /nolog
```

使用system进入oracle

```shell
connect system
```

会提示输入密码，密码为：oracle

然后出现以下信息

```shell
ERROR:
ORA-28002: the password will expire within 7 days


Connected.
```

这个密码只能适用7天，因此需要修改密码,退出oracle的system，在oracle容器里：

先给oracle权限

```shell
su oracle
```

以sysdba的方式进入oracle

```shell
sqlplus / as sysdba
```

修改密码时间权限

```sql
alter profile default limit password_life_time unlimited;
```

查询修改后的密码配置情况

```sql
select * from dba_profiles s where s.profile='DEFAULT' and resource_name='PASSWORD_LIFE_TIME';
```
如果DEFAULT为unlimit就是配置成功了

## 远程登录方式

```shell
Connection Type:Basic
port:49161
Service Name:xe
Service Name的类型:SID
Role:Default
User name:system
password:oracle
```

## Mac下载SQL*Plus和Basic

默认Mac下已经下载了连接oracle的工具，我下载的是Navicat Premium

下载SQL*Plus和Basic链接：

http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html

下载内容：

instantclient-basic-macos.x64–11.2.0.4.0.zip
instantclient-sqlplus-macos.x64–11.2.0.4.0.zip

cd到下载内容所在的文件夹下

把下载好的文件放到~/Library/Caches/Homebrew下

```shell
mv instantclient-basic-macos.x64-18.1.0.0.0.zip ~/Library/Caches/Homebrew
mv instantclient-sqlplus-macos.x64-18.1.0.0.0.zip ~/Library/Caches/Homebrew
```

执行以下命令

```shell
$ brew tap InstantClientTap/instantclient
$ brew install instantclient-basic
$ brew install instantclient-sqlplus
```

在运行brew install instantclient-basic和brew install instantclient-sqlplus命令时，也许会出现错误提示信息，要求对instantclient-basic-macos.x64–11.2.0.4.0.zip和instantclient-sqlplus-macos.x64–11.2.0.4.0.zip改名并移动到~/Library/Caches/Homebrew/Download下。复制要修改的名称改名并复制就可

例如：

```shell
To this location (a specific filename in homebrew cache directory):

  /Users/wanghui/Library/Caches/Homebrew/downloads/1ace9ca784e431112e837a769fc89eae38ad1489165c38aa698139d25d8fd96b--instantclient-basic-macos.x64-18.1.0.0.0.zip
```

进入~/Library/Caches/Homebrew，将instantclient-basic-macos.x64–11.2.0.4.0.zip修改为1ace9ca784e431112e837a769fc89eae38ad1489165c38aa698139d25d8fd96b--instantclient-basic-macos.x64-18.1.0.0.0.zip，然后移动到~/Library/Caches/Homebrew/downloads

## 配置Navicat Premium

Navicat Premium --> Preferences --> Environment

默认的安装路径

`/usr/local/Cellar/instantclient-sqlplus/18.1.0.0.0`

将其填入

![](oracle.png)

这个是我本人的

个人oracle账号：

Connect database with following setting:

    hostname: localhost
    port: 1521
    sid: EE
    service name: EE.oracle.docker
    username: system
    password: oracle

To connect using sqlplus:

<pre>
sqlplus system/oracle@//localhost:1521/EE.oracle.docker
</pre>

Password for SYS & SYSTEM:

    oracle


