---
title: macos高版本安装php扩展报错
description: 安装php扩展xdebug, mongodb  
categories:
 - php
tags:
 - macos
 - xdebug
---

# mac安装php扩展报错问题分析

## 背景

1. 机器：mac os catalina
2. 版本: 10.15.2
3. PHP: 7.3.9( 系统默认安装)

php的一个项目依赖mongodb扩展， `composer update`的提示，提示需要通过pecl安装mongodb扩展, 通过pecl安装扩展时，提示 找不到PHP头文件错误， 如下报错。

```
phpize 
grep: /usr/include/php/main/php.h: No such file or directory
grep: /usr/include/php/Zend/zend_modules.h: No such file or directory
grep: /usr/include/php/Zend/zend_extensions.h: No such file or directory
Configuring for:
PHP Api Version:        
Zend Module Api No:     
Zend Extension Api No:
```
最终顺利解决，这里记录下问题分析


## 问题分析

中文资料非常少，并且内容过时， 大部分是针对 版本`10.14`, `10.13`版本下的解决方案：
```
sudo installer -pkg /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg -target /
```

对于我们的`10.15`版本并不适用。

有的是直接修改phpize，这里改变了系统的phpize，不可取(https://bbqsoftwares.com/blog/xdebug-catalina-issue)

这里找到一篇思路清晰的解决方案针对xdebug，适用于所以需要编译的pecl扩展。把这篇文章翻译过来， 做记录。


## 怎么在MacOS 10.15 Catalina上安装xdebug

在 macOS Catalina上当你尝试构建xdebug时， 会报如下错误:

```
phpize 
grep: /usr/include/php/main/php.h: No such file or directory
grep: /usr/include/php/Zend/zend_modules.h: No such file or directory
grep: /usr/include/php/Zend/zend_extensions.h: No such file or directory
Configuring for:
PHP Api Version:        
Zend Module Api No:     
Zend Extension Api No:
```

```
xdebug-2.9.0/xdebug.c:25:10: fatal error: 'php.h' file not found
#include "php.h"
         ^~~~~~~
1 error generated.
make: *** [xdebug.lo] Error 1
```

```
/usr/include/php/main/php.h:33:10: fatal error: 'zend.h' file
      not found
#include "zend.h"
         ^~~~~~~~
1 error generated.
make: *** [xdebug.lo] Error 1
```

````
/usr/include/php/Zend/../TSRM/TSRM.h:20:11: error: 
      'tsrm_config.h' file not found with <angled> include; use "quotes" instead
# include <tsrm_config.h>
          ^~~~~~~~~~~~~~~
          "tsrm_config.h"
....
/usr/include/php/Zend/zend_virtual_cwd.h:24:10: fatal error: 
      'TSRM.h' file not found
#include "TSRM.h"
         ^~~~~~~~
2 errors generated.
make: *** [xdebug.lo] Error 1
````

### 在macOS Catalina上配置，构建xdebug

上述错误的原因主要是 在macos上发布最新的`xcode11`时，把目录`/usr/include`给去掉了。

我们可以绕过这个问题，继续编译xdebug

首先， 我们需要确认 `Xcode`和命令行工具(command line tools) 已经正确安装，打开命令行终端，运行以下命令，打印sdk路径

```
xcrun --show-sdk-path
```

理论上会输出

```
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk
```

如果输出跟上面不吻合，在命令行执行以下命令

```
xcode-select --install
```
等安装完毕，打开xcode，确保正确安装，重新打印sdk路径

```
xcrun --show-sdk-path
```

编译PHP扩展需要被删除的 `include`目录，  因此我们可以对`phpize`和`php-config` 程序进行修改，使其能使用 mac os sdk的include目录。 我们首先复制出一份 `phpize`和`php-config`， 我们通过打补丁(patch)的方式，去修改这个复制品，（主要不影响原有功能 ）

在我们个人目录新建 `php-private`

```
mkdir ~/php-private/
```

复制程序`phpize`和`php-config`到 `php-private`目录。

```
cp /usr/bin/phpize ~/php-private/
cp /usr/bin/php-config ~/php-private/
```

查看PHP版本

```
grep version= ~/php-private/php-config
```

可能会有以下输出 

```
version="7.3.11
```

我已经提前做好程序的补丁，将这两个文件下载下来。

下载`phpize`[补丁](https://profilingviewer.com/catalina-php-patches/phpize-catalina.patch.zip)，补丁详细内容如下：

```
--- /usr/bin/phpize	2019-09-11 02:46:18.000000000 +0200
+++ ./phpize	2019-12-26 23:10:32.000000000 +0100
@@ -1,11 +1,12 @@
 #!/bin/sh
 
 # Variable declaration
+XCODE_SDK_ROOT=$(/usr/bin/xcrun --show-sdk-path)
 prefix='/usr'
 datarootdir='/usr/php'
 exec_prefix="`eval echo ${prefix}`"
 phpdir="`eval echo ${exec_prefix}/lib/php`/build"
-includedir="`eval echo ${prefix}/include`/php"
+includedir="`eval echo ${XCODE_SDK_ROOT}${prefix}/include`/php"
 builddir="`pwd`"
 SED="/usr/bin/sed"
```

对 php 7.3.9,下载 `php-config`[补丁](https://profilingviewer.com/catalina-php-patches/php-config-7.3.9-catalina.patch.zip)

对于7.3.11, 下载[补丁](https://profilingviewer.com/catalina-php-patches/php-config-7.3.11-catalina.patch.zip)

php-confg的补丁如下

```
--- /usr/bin/php-config	2019-09-11 02:48:43.000000000 +0200
+++ ./php-config	2019-12-26 23:10:19.000000000 +0100
@@ -1,12 +1,14 @@
 #! /bin/sh
 
+XCODE_SDK_ROOT=$(/usr/bin/xcrun --show-sdk-path)
+
 SED="/usr/bin/sed"
 prefix="/usr"
 datarootdir="/usr/php"
 exec_prefix="${prefix}"
 version="7.3.9"
 vernum="70309"
-include_dir="${prefix}/include/php"
+include_dir="${XCODE_SDK_ROOT}${prefix}/include/php"
 includes="-I$include_dir -I$include_dir/main -I$include_dir/TSRM -I$include_dir/Zend -I$include_dir/ext -I$include_dir/ext/date/lib"
 ldflags=" -L$SDKROOT/usr/lib -L$SDKROOT/usr/local/libressl/lib  -L/usr/local/lib"
 libs="-lresolv  -lcrypto -lssl -lcrypto -lexslt -ltidy -lresolv -ledit -lncurses -lpq -lpq -lldap -llber -liconv -liconv -lpng -lz -ljpeg -lcrypto -lssl -lcrypto -lbz2 -lz -lcrypto -lssl -lcrypto -lm  -lxml2 -lz -licucore -lm -lkrb5 -lcurl -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lnetsnmp -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxslt -lxml2 -lz -licucore -lm "
	
```

解压压缩包

PHP 7.3.9
```
unzip ~/Downloads/phpize-catalina.patch.zip -d ~/Downloads/
unzip ~/Downloads/php-config-7.3.9-catalina.patch.zip -d ~/Downloads/
```

PHP 7.3.11
```
unzip ~/Downloads/phpize-catalina.patch.zip -d ~/Downloads/
unzip ~/Downloads/php-config-7.3.11-catalina.patch.zip -d ~/Downloads/
```
现在我们对我们的副本 `phpize`和`php-config`打补丁

PHP 7.3.9

```
patch ~/php-private/phpize < ~/Downloads/phpize-catalina.patch
patch ~/php-private/php-config < ~/Downloads/php-config-7.3.9-catalina.patch
```

PHP 7.3.11
```
patch ~/php-private/phpize < ~/Downloads/phpize-catalina.patch
patch ~/php-private/php-config < ~/Downloads/php-config-7.3.11-catalina.patch
```

到这里，我们已经把编译xdebug的前期工作准备就绪。

在`home`目录，常见xdebug的build文件夹

```
mkdir ~/xdebug-install
```

从[Xdebug官网](http://xdebug.org/)下载xdebug到 `~/Downloads`

```
cp ~/Downloads/xdebug-2.9.0.tgz ~/xdebug-install
cd ~/xdebug-install
tar -xvzf xdebug-2.9.0.tgz
cd xdebug-2.9.0
```

现在我们在xdebug目录运行我们补丁版的phpize

```
~/php-private/phpize
```

输出内容，一切正常。

```
Configuring for:
PHP Api Version:         20180731
Zend Module Api No:      20180731
Zend Extension Api No:   320180731
```

执行过程中，如果没有其他报错， 你可以跳过下面张杰，直接进入到 `配置安装xdebug`章节

再执行过程中， 你可能遇到以下错误，提示你需要安装一些程序

```
Configuring for:
PHP Api Version:         20180731
Zend Module Api No:      20180731
Zend Extension Api No:   320180731
Cannot find autoconf. Please check your autoconf installation and the
$PHP_AUTOCONF environment variable. Then, rerun this script.
```

如果 phpize 输出以下内容， 就代表我们需要安装`autoconf`

```

Cannot find autoconf. Please check your autoconf installation and the
$PHP_AUTOCONF environment variable. Then, rerun this script.
```

执行以下命令，对autoconf就行安装

```
#change back to our working directory
cd ~/xdebug-install

#dowload autoconf
curl -OL http://ftpmirror.gnu.org/autoconf/autoconf-2.69.tar.gz

#extract it, and change into the folder
tar xzf autoconf-2.69.tar.gz
cd autoconf-2.69

#now build and install it
./configure --prefix=/usr/local
make
sudo make install

#change back to the xdebug folder
cd ..
cd xdebug-2.9.0
```

重新在xdebug文件夹运行补丁 phpize

```
~/php-private/phpize
```

重新查看输出， 如果一切正常， 我们继续安装流程

```
Configuring for:
PHP Api Version:         20180731
Zend Module Api No:      20180731
Zend Extension Api No:   320180731
```

### 配置，build xdebug程序

我们可以看下补丁`php-config`的绝对路径

```
echo ~/php-private/php-config
```
输出如下：
```
/Users/YOUR-USERNAME/php-private/php-config
```

将上述路径 配置到以下命令中

```
./configure --enable-xdebug --with-php-config=/Users/YOUR-USERNAME/php-private/php-config
```

命令输出结果，你会发现 sdk path已经被正确使用了

```
...
checking for PHP prefix... /usr
checking for PHP includes... -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php/main -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php/TSRM -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php/Zend -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php/ext -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php/ext/date/lib
checking for PHP extension directory... /usr/lib/php/extensions/no-debug-non-zts-20180731
checking for PHP installed headers prefix... /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/php
checking if debug is enabled... no
...
```

现在，我们可以build扩展

```
make
```

我们不能 macos系统上 直接进行 `make install`操作来安装xdebug.so， 主要是因为 macos  sip保护系统不允许我们把 xdebug安装到 `/usr/lib/extensions`目录， 因此我们把扩展安装到 `/usr/local`目录

```
sudo mkdir -p /usr/local/php/extensions
sudo cp modules/xdebug.so /usr/local/php/extensions
```

现在我们去编译 `php.ini`(一般会在/etc/php.ini) 去加载我们打包好的xdebug, PHP扩展默认去专门的扩展目录搜寻需要安装的扩展， 因为我们的xdebug在该目录外面， 因此我们需要指定绝对路径。

```
zend_extension=/usr/local/php/extensions/xdebug.so
```
我们做个简单测试， 执行
```
php -i | grep xdebug
```
预期输出状态：
```
xdebug
xdebug support => enabled
....
```

然后我们重新apache -web服务， 使我们的修改生效

```
sudo apachectl restart
```


## 参考文章

1. https://profilingviewer.com/installing-xdebug-on-catalina.html
2. https://stackoverflow.com/questions/52592548/unable-to-use-phpize-after-update-to-macos-mojave
3. [patch命令](https://www.jianshu.com/p/6ee74c2790b1)
4. https://bbqsoftwares.com/blog/xdebug-catalina-issue


## 其他收获

1. 学习了`diff``patch`命令


