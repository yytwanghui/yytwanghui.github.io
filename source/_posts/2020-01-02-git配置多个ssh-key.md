---
title: git配置多个ssh-key
date: 2020-01-02 17:37:42
categories:
- Git
tags:
- git
- ssh-key
---

<center><font size=4 color="red">git配置多个ssh-key</font></center>

<!--more-->

## git配置多个ssh-key

> 我们在工作当中，经常遇到自己有个gitlab账号，然而公司也有个gitlab账号，或者再有个github账号，或者gitee账号，这么多的账号，每次拉取项目都是个很麻烦的问题，因为需要配置多个ssh-key来管理。

#### 生成公司和个人的ssh-key

```shell
#生成公司的ssh-key
$ ssh-keygen -t rsa -C 'youremail@yourcompany.com' -f ~/.ssh/gitlab_company_rsa
#生成个人的ssh-key
$ ssh-keygen -t rsa -C 'youremail@your.com' -f ~/.ssh/gitlab_myself_rsa
```

* 生成公司的ssh-key

```shell
$ ssh-keygen -t rsa -C 'wanghui_isf@si-tech.com.cn' -f ~/.ssh/gitlab_company_rsa
Generating public/private rsa key pair.
Created directory '/c/Users/wanghui/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/wanghui/.ssh/gitlab_company_rsa.
Your public key has been saved in /c/Users/wanghui/.ssh/gitlab_company_rsa.pub.
The key fingerprint is:
SHA256:wPCl9enJ2v43bFr3ZeMQuAP9o9An8jzSiCI8/XRfnso wanghui_isf@si-tech.com.cn
The key's randomart image is:
+---[RSA 3072]----+
|    .   o        |
|     + + . .     |
|      =   o      |
|       . o...    |
|        S.+o .   |
|         oo o .  |
|  . .  .o++= B..+|
|   + o...+BoB.Xo+|
|    o o.  oE=* o.|
+----[SHA256]-----+
```

* 生成个人的ssh-key

```shell
$ ssh-keygen -t rsa -C 'emcderm2@students.solano.edu' -f ~/.ssh/gitlab_myself_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/wanghui/.ssh/gitlab_myself_rsa.
Your public key has been saved in /c/Users/wanghui/.ssh/gitlab_myself_rsa.pub.
The key fingerprint is:
SHA256:jqYu2k9TddzlwpNLzvXXkJBfhldXbaZzsPsiLz7P4JQ emcderm2@students.solano.edu
The key's randomart image is:
+---[RSA 3072]----+
|            ....B|
|         . o.=o.O|
|        . o B.=X |
|       . . + ==oo|
|      . S   +  ++|
|     . o     .. .|
|    o o .   E  . |
| ... +     o+o. .|
|...++      .o*+. |
+----[SHA256]-----+
```

* 查看公司和个人的ssh-key

```shell
#查看公司的ssh-key
$ cat /c/Users/wanghui/.ssh/gitlab_company_rsa.pub
#查看个人的ssh-key
$ cat  /c/Users/wanghui/.ssh/gitlab_myself_rsa.pub
```

#### 添加ssh-key到gitlab中

![addsshkey](addsshkey.jpg)

#### 添加私钥

```shell
$ ssh-agent bash
$ ssh-add ~/.ssh/gitlab_company_rsa
$ ssh-add ~/.ssh/gitlab_myself_rsa
$ ssh-add -l
```
> 备注：如果`ssh-add ~/.ssh/gitlab_company_rsa`不行，就改成`ssh-add ~/.ssh/gitlab_company.rsa`

具体操作

```shell
wanghui@DESKTOP-H6A9OVE MINGW64 /c/wanghui/persion/blog (master)
$ ssh-agent bash

wanghui@DESKTOP-H6A9OVE MINGW64 /c/wanghui/persion/blog (master)
$ ssh-add ~/.ssh/gitlab_company_rsa
Identity added: /c/Users/wanghui/.ssh/gitlab_company_rsa (wanghui_isf@si-tech.com.cn)

wanghui@DESKTOP-H6A9OVE MINGW64 /c/wanghui/persion/blog (master)
$ ssh-add ~/.ssh/gitlab_myself_rsa
Identity added: /c/Users/wanghui/.ssh/gitlab_myself_rsa (emcderm2@students.solano.edu)

wanghui@DESKTOP-H6A9OVE MINGW64 /c/wanghui/persion/blog (master)
$ ssh-add -l
3072 SHA256:wPCl9enJ2v43bFr3ZeMQuAP9o9An8jzSiCI8/XRfnso wanghui_isf@si-tech.com.cn (RSA)
3072 SHA256:jqYu2k9TddzlwpNLzvXXkJBfhldXbaZzsPsiLz7P4JQ emcderm2@students.solano.edu (RSA)
```

#### 创建config文件

文件所在路径 `~/.ssh`，配置信息

```config
# gitlab_company
Host git.si-tech.com.cn
    HostName git.si-tech.com.cn
    User git
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitlab_company_rsa
# gitlab_myself
Host gitlab.com
    HostName gitlab.com
    User git
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitlab_myself_rsa
```

config配置文件讲解,以公司ssh-key为例

```config
# gitlab_company
Host git.si-tech.com.cn
    HostName git.si-tech.com.cn
    User git
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitlab_company_rsa
```

例如拉取的项目是：`git@git.si-tech.com.cn:chengwei/woapp-dev.git`

配置文件`User+HostName = git@git.si-tech.com.cn`

其中

* User = git
* HostName = git@git.si-tech.com.cn

#### 验证

```shell
#验证公司的是否连接
$ ssh -T git@git.si-tech.com.cn
#验证个人的是否连接
$ ssh -T git@gitlab.com
```

如果连接，会显示出用户名

