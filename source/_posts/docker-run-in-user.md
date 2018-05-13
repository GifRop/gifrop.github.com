---
title: 使用user账号登录docker容器
date: 2018-05-13 13:30:49
tags: [Docker]
---
平时工作中，使用docker容器时，均使用的root账号登录。但是有时候，你需要使用其他的账号，部署一些需要权限限制的服务，例如nginx。<!-- more -->
使用下列步骤即可完成使用普通用户登录docker。  
第一步，启动你的容器并以root进入，创建一个work账号：  
```sh
useradd -d /home/work work
```
第二步，为用户设置登录密码：  
```sh
[root@test]# passwd work
Changing password for user work.
New UNIX password:
Retype new UNIX password:
passwd:all authentication takens updated successfully.
```
第三步，设置用户主目录，由于之前使用的root创建的/home/work，这时候登录会有问题。
```sh
#如果没创建则创建
#mkdir -p /home/work
chown work:work /home/work #设置目录拥有者和组
#如果该目录有子目录
#chown -R work:work /home/work
```
这时候就可以使用work账号进行登录。  
```sh
#退出容器进入宿主机执行
docker exec -it --user=work test bash
```

