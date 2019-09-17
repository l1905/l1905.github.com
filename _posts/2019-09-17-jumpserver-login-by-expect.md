---
title: 通过expect实现登录跳板机jumpserver的指定机器
description: 通过expect实现登录跳板机jumpserver的指定机器
categories:
 - jumpserver
tags:
 - jumpserver
---


### 背景


目前公司测试环境服务器登录 不能通过ssh登录， 必须通过跳板机jumpserver实现集中ssh登录， 主要方便管理监控开发命令执行。

目前jumpserver 开启MFA认证， 每次登录，

1. 都需要输入「谷歌验证器」提供的6位 验证码， 
2. 并且每次都需要进入到堡垒机菜单页面， 选择需要连接的实际机器。才能实际操作。

过程非常繁琐， 严重影响开发效率， 所以是希望能简化登录服务器流程， 在windows平台，可以使用securecrt, 但是mac平台没有好用的工具。


### mac实现方案

查资料，发现有一种 linux下的小语言`expect`程序(类似awk），可以实现执行脚本时自动交互。即能简化我们的登录步骤(2), 并且该程序在macos下是默认安装。常用命令有以下：

* send：用于向进程发送字符串
* send_user：向当前用户打印消息
* expect：从进程接收字符串， 即必须接手spawn进程的命令
* exp_continue：循环继续进行
* spawn：启动新的进程
* interact：允许用户交互


脚本实现方式为：

```
#!/usr/bin/expect -f
 
set timeout 30

#获取第一个参数，作为用户名
set username [lindex $argv 0]

#启动新的进程，实现ssh操作
spawn ssh  -p 2222 $username@jumpserver.ddddd.com

#for循环操作
expect {
        "*MFA*" {
            # send_user "输入MFA\n"
            set AUTH [gets stdin]
            send "$AUTH\n"
            # 继续循环执行
            exp_continue
        }
        "*Opt*" {
            send "10.4.255.255\n"
        }
}

expect "*root@10-4-255-255*"

# 登录到指定用户账号
send "su $username\n"
send "cd \n"

#进行交互操作
interact
```

使用方式为

```
chmod a+x demo.sh
./demo username
```


### 参考资料：

1. https://blog.csdn.net/yzl11/article/details/52838449
2. https://www.cnblogs.com/lzrabbit/p/4298794.html
3. http://xstarcd.github.io/wiki/shell/expect.html


