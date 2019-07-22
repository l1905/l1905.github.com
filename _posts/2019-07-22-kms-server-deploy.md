---
title: kms-server搭建记录
description: kms-server搭建记录
categories:
 - 教程
tags:
 - KMS
 - 激活工具
---

## KMS服务器搭建流水


## 背景

最近重装系统， 上次重装系统还是3，4年前，还是用的知名`小马跑跑`激活工具。 目前看，已经废掉， 经过调研，用第三方PE系统装怕广告各种飞， 因此选择安装纯净版系统。

建议下载纯净版镜像网址： https://msdn.itellyou.cn/

国内访问较快的纯净版镜像地址： http://www.win10com.com/win10msdn/

安装之后，面临激活问题， 上次选择淘宝买激活码， 感觉不是长久办法， 于是搭建`kms服务器`用于激活后期重装的win10系统

虚拟机环境
> centos 7

## 搭建流程



```
https://github.com/Wind4/vlmcsd/releases

#下载激活工具 vlmcsd
wget https://github.com/Wind4/vlmcsd/releases/download/svn1112/binaries.tar.gz

#解压到当前目录
tar zxvf binaries.tar.gz

#进入目录， 将对应的bin文件复制到`/user/local/bin`下面
cp /root/binaries/Linux/intel/static/vlmcsd-x64-musl-static /usr/local/bin/vlmcsd-x64-musl-static  

#尝试启动
> vlmcsd-x64-musl-static
> 

#防火墙放行1688端口
iptables -I INPUT -p tcp --dport 1688 -j ACCEPT


#会自动监听 1688接口， 我们可以验证端口使用情况
lsof -i:1688


```

## 验证服务器

```
windows系统下

#载激活工具 vlmcsd
wget https://github.com/Wind4/vlmcsd/releases/download/svn1112/binaries.tar.gz

# 进入windows/intel/static

# 找到vlmcs-Windows-x64.exe 激活文件

# 获取版本号
./vlmcs-Windows-x64.exe -x

# 从上一步拿到版本号，进行验证操作

> ./vlmcs-Windows-x64.exe -v -l {版本号} {你的IP}

```

上一步输出如下：

```
Request Parameters
==================

Protocol version                : 6.0
Client is a virtual machine     : No
Licensing status                : 2 (OOB grace)
Remaining time (0 = forever)    : 43200 minutes
Application ID                  : 55c92734-d682-4d71-983e-d6ec3f16059f (Windows)
SKU ID (aka Activation ID)      : 8de8eb62-bbe0-40ac-ac17-f75595071ea3 (Windows Server 2019 ARM64)
KMS ID (aka KMS counted ID)     : 8449b1fb-f0ea-497a-99ab-66ca96e9a0f5 (Windows Server 2019)
Client machine ID               : 4fb75b7c-bfa0-4b96-8528-dea579c04ce1
Previous client machine ID      : 00000000-0000-0000-0000-000000000000
Client request timestamp (UTC)  : 2019-07-22 07:38:37
Workstation name                : we-love.yahoo.es
N count policy (minimum clients): 5

Connecting to 68.168.129.114:1688 ... successful

Performing RPC bind ...
... NDR64 ... BTFN ... NDR32 ... successful
Sending activation request (KMS V6) 1 of 1

Response from KMS server
========================

Size of KMS Response            : 260 (0x104)
Protocol version                : 6.0
KMS host extended PID           : 06401-00206-551-838903-03-4122-9600.0000-1572019
KMS host Hardware ID            : 3A1C049600B60076
Client machine ID               : 4fb75b7c-bfa0-4b96-8528-dea579c04ce1
Client request timestamp (UTC)  : 2019-07-22 07:38:37
KMS host current active clients : 50
Renewal interval policy         : 10080
Activation interval policy      : 120
```

到此，我们验证程序安装成功。


## 将程序以守护进程方式运行

新建`vlmcsd-daemon.service`

```
[Unit]
Description=vlmcsd-daemon
After=network.target
Wants=network-online.target

[Service]
Type=forking #这里切记是要forking, 要不然会自动退出守护进程
User=root
Group=root
WorkingDirectory=/root/litong/
ExecStart=/usr/local/bin/vlmcsd-x64-musl-static
Restart=always


[Install]
WantedBy=multi-user.target
```

### 软链接到systemd目录

```
#软连接一定要绝对路径
ln -s /root/litong/systemd-service/vlmcsd-daemon.service /usr/lib/systemd/system/vlmcsd-daemon.service
```

### 重新载入

```
systemctl daemon-reload
```

### 查看状态

```
> systemctl status  vlmcsd-daemon.service


● vlmcsd-daemon.service - vlmcsd-daemon
   Loaded: loaded (/root/litong/systemd-service/vlmcsd-daemon.service; enabled; vendor preset: disabled)
   Active: active (running) since 一 2019-07-22 06:04:17 EDT; 37min ago
  Process: 881 ExecStart=/usr/local/bin/vlmcsd-x64-musl-static (code=exited, status=0/SUCCESS)
 Main PID: 892 (vlmcsd-x64-musl)
   CGroup: /system.slice/vlmcsd-daemon.service
           └─892 /usr/local/bin/vlmcsd-x64-musl-static

7月 22 06:04:17 ideal-text-2.localdomain systemd[1]: Starting vlmcsd-daemon...
7月 22 06:04:17 ideal-text-2.localdomain systemd[1]: Started vlmcsd-daemon.
```

### 加入到开机启动

```
systemctl enable  vlmcsd-daemon.service
```



## window激活操作(只能激活VL版！！！)


目前未实践， 后续再重装系统时，再实践

以管理员身份运行命令提示符或者PowerShell，然后输入以下命令：

```
cd /d "%SystemRoot%\system32"
slmgr /skms 你的KMS服务端主机的IP或者域名
slmgr /ato
slmgr /xpr
```


## office激活操作(只能激活VL版！！！)

接下来以管理员身份运行命令提示符或者PowerShell，然后输入以下命令：

```
#以 64 位的 Office2016 为例，进入 Office 目录
cd "C:\Program Files\Microsoft Office\Office16"
cscript ospp.vbs /sethst:你的KMS服务端主机的IP或者域名
cscript ospp.vbs /act
```


## 参考资料

1. https://moe.best/tutorial/linux-kms-server.html
2. https://github.com/dakkidaze/one-key-kms
3. https://zilongshijia.top/index.php/2019/03/16/%E8%87%AA%E5%BB%BAkms%E6%9C%8D%E5%8A%A1%E5%99%A8/
4. https://mogeko.me/2018/017/
5. http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html
6. https://npect.com/2018/03/15/create-service-in-systemd/
7. https://butui.me/post/vlmcsd/

































