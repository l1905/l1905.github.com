---
title: 将mysql数据备份定时发送到email
description: 将mysql数据备份定时发送到email
categories:
 - 运维
tags:
 - mysql
 - 备份
---


## 备份mysql数据库到邮箱

### 背景

环境

* centos

备份主要分为两个步骤

+ 首先安装发送邮件需要的程序并配置自己邮箱
+  通过mysqldump备份数据库文件

### 安装对应的发送邮件程序

* yum install mutt -y
* yum install msmtp -y

### 编辑文件加入配置

vim /etc/Muttrc.local

```
# Local configuration for Mutt.

set sendmail="/usr/bin/msmtp"
set use_from=yes
set realname="126备份邮箱"
set from="xxxxxx@126.com"
set envelope_from=yes
```


继续编辑配置文件

vim .msmtprc

```
account default
host smtp.126.com
port 25
tls off
auth login
user xxxxx@126.com #邮箱账号
password xxxxx #邮箱密码
```


测试邮件是否配置配置成功

`echo "标题" | mutt -s "$DATABASE备份" xxxx@qq.com -a 附件.sql`



### 配置发送邮件脚本

```
#!/bin/bash

BACKUP_PATH=/data/backup/mysql
CURRENT_TIME=$(date +%Y%m%d_%H%M%S)

[ ! -d "$BACKUP_PATH" ] && mkdir -p "$BACKUP_PATH"

#数据库地址
HOST=localhost
#数据库用户名
DB_USER=
#数据库密码
DB_PW=
# 要备份的数据库名
DATABASE=huanghou
FILE_GZ=${BACKUP_PATH}/$CURRENT_TIME.$DATABASE.sql.gz
/usr/bin/mysqldump -u${DB_USER} -p${DB_PW} --host=$HOST -q -R --databases $DATABASE  | gzip > $FILE_GZ # 此处必须要用绝对路径

# 所有数据库
#mysqldump --all-databases -xxxxx

echo "数据库备份--$FILE_GZ" | mutt -s "$DATABASE备份-" 123456@qq.com -a $FILE_GZ

# 删除 7 天以前的备份 「注意写法」
cd $BACKUP_PATH
find $BACKUP_PATH -mtime +7 -name "*sql.gz"  -exec rm -f {} \;

```

### 解决出现问题

作为发件：网易邮箱(163,126) ，无法在阿里云上使用， 是因为网易的smtp服务端口使用25， 但是阿里云出于安全考虑，将此端口禁掉， 即无法在阿里云上使用网易邮箱作为发件方

#### 替代方案

申请新的qq邮箱，作为替代方案。 但是qq邮箱申请后， 得15天后，才能使用其smtp服务， 真的服务掉渣渣。

这里等待申请通过后，在验证备份信息。

### 执行定时任务

```
凌晨三点三十分执行一次
crontab -e
30 03 * * * /data/shells/mysqlBackup.sh
```


## 参考资料

https://www.jianshu.com/p/1f69f7003f3f

https://learnku.com/articles/13342/mysql-auto-backup-and-send-to-mailbox