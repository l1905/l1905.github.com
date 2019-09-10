---
title: 使用pac文件，配合手机抓包
description: 使用pac文件，配合手机抓包
categories:
 - 日常开发
tags:
 - pac
---


### 遇到的问题

使用charles或者findder抓包，配合客户端调试接口请求时， 
常用的使用方式是：

手机客户端设置 wifi代理到 个人电脑----->个人电脑打开charles----> 设置本机为内网host---> 抓包查看。


手机上的全部请求都会通过代理，请求到个人电脑上。 像虎扑，或者下厨房等应用， 检测到使用代理后，应用提示网络报错。


所以我们核心问题是： 怎么才能只代理 我们自己的域名请求，而不影响其他应用请求。


### 使用pac


我们使用酸酸乳时， 会看到一个选项是「PAC自动模式」，这就是我们要用的pac模式。

摘要自 维基百科：

>
一个PAC文件包含一个JavaScript形式的函数“FindProxyForURL(url, host)”。这个函数返回一个包含一个或多个访问规则的字符串。用户代理根据这些规则适用一个特定的代理器或者直接访问。当一个代理服务器无法响应的时候，多个访问规则提供了其他的后备访问方法。浏览器在访问其他页面以前，首先访问这个PAC文件。PAC文件中的URL可能是手工配置的，也可能是是通过网页的网络代理自动发现协议（WPAD）自动配置的。


所以我们只需要定义PAC文件，指定如果域名是 xxx.com , 就走代理，其他是直接请求。


所以我们定义自己的pac文件即可

```
function FindProxyForURL(url, host) {

// If the hostname matches, send direct.
    if (dnsDomainIs(host, "domain.com") ||
        shExpMatch(host, "(*.smz.com|smz.com)"))
        return "PROXY 172.26.201.43:8888"; //修改这里的IP端口， 为自己本机的IP端口
}
```

命名为 xxx.pac.js, 上传到我们自己的服务器上。 比如地址是

```
http://106.75.92.63:8080/litong_pac.js
```


客户端设置网络为 PAC代理方式。


正常上网即可。



### 参考文章

1. https://gist.github.com/sheng168/6f5e32825f68e1a326eb61ce73df8a47
2. https://github.com/petronny/gfwlist2pac/blob/master/gfwlist.pac
3. https://www.barretlee.com/blog/2016/08/25/pac-file/