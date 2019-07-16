---
title: 使用jekyll+github pages搭建独立博客
description: 简述搭建jekyll+github pages
categories:
 - 教程
tags:
 - 教程
 - 搭建
 - github
---

主要使用的`jekyll`的`Next`主题。

### 搭建主要流程：

1. github上新建仓库， 命名为 `username.github.io`， 本地 `git clone`下来
2. 将下载的`Next`源码复制到对应仓库代码中
3. 参考`Next`安装教程进行操作， 不需要额外安装`jekyll`， 会自动安装jekyll
4. 进入代码目录，进行本地调试 `bundle exec jekyll server`， 如果报错， 需要修改`_config.yml`文件的`description`字段。
5. 本地代码无问题，将代码提交到git仓库， 并push 到github
6. 浏览器打开`username.github.io`, 进行验收


### 后续开发主要流程：

在`_post`目录新建文件， 以`年-月-日-title.md`进行命名， 头部标识文章属性， 比如`摘要`，`具体时间`,`栏目`,`标签等`

类似如下：

```
---
title: Next Theme Tutorial
description: 简述搭建jekyll+github pages
categories:
 - 教程
tags:
 - 教程
 - 搭建
 - github
---
```


后续开发既可以本地搭建调试环境， 也可以直接在`_post`目录新增文件进行提交


### 附加功能

引入百度统计，谷歌统计，评论模块即参考Next主题教程， 不再详细描述

### 参考目录

1. [jekyll主题Next安装教程](http://theme-next.simpleyyt.com/getting-started.html)


