---
title: 针对git之对revert的revert操作记录
description: git对revert进行revert操作
categories:
 - 开发常见问题解决
tags:
 - git
 - revert
---


### 背景

公司开发环境主干dev分支，突然发现少了几天的提交记录，从git log中也排查不出问题， 根据经验， 是被revert掉代码了， 丢失代码比较多， 需要找回， 所以我们本次面对的问题是 怎么找回被revert的代码， 即revert--更早revert操作代码


### 确定问题

先切到dev分支，找到是哪次commit合并提交，导致的代码丢失，但是非常不容易追踪，因为merge-log中没有详细的提交log, 

我们这根据 距离上次commit时间差来来推测， 比如我们今天31号， 我们上次提交是25号，时间差较大， 猜测大概率是 这次提交出的问题。


### 解决问题


我们查看本次commit， 对应的has_id为 c6d6bf323， 并且有两个父级提交， 一个是本地的commit的节点， 一个是远端仓库的commit的节点.

我们可以通过分别查看两个父级hash_id, 来确认， 到底是要恢复哪个节点

```
> git log ab168a530d
> git log cc5274deeb
```

这里我们确认 

ab168a530d 是我们将要恢复的分支， 并且排序是第二个，我们执行如下命令进行恢复

 
```

git revert c6d6bf323 -m 2

#pull代码
git pull
 
#push到远端
git push

```


## 经验总结

revert操作会丢失提交记录， 切记要小心


## 参考资料

1. https://stackoverflow.com/questions/7099833/how-to-revert-a-merge-commit-thats-already-pushed-to-remote-branch
