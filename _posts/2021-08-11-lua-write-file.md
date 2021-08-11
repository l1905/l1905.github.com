---
title: openresty中使用lua输出json日志到磁盘文件串行问题
description: openresty中使用lua输出json日志到磁盘文件串行问题
categories:
 - openresty
tags:
 - json
 - openresty
---

# openresty中使用lua输出json日志到磁盘文件串行问题


## 背景

公司客户端日志上报模块，使用openresty+lua 先将json日志文件写到磁盘，然后flume采集走。

最近发现， 落到磁盘的文件日志，存在串行情况， 即第二个json覆盖了第一个json的部分内容， 导致这两个json串都无法正常解析。

测试文本内容如下：

```
{"system_v":"14.7.1","f":"iphone","sdk_v":"0.1.19","network_data":{"operator":"2","network_type":"4G","dns_server":"61.166.150.123","domain":"dingyue-api.test.com","s_duration_s":"0.029","local_proxy":"","x_zhi_request_id":"4j7g7nfQ-B7bPtkGg-njS-4uhe","remote_address":"116.211.155.218","local_address":"106.61.154.214","duration":"404.4849872589111","http_status_code":"200","path":"\/dy\/util\/api\/user_action"},"domain":"applog.debug","cookie":{"network":"4","device_s":"kQNQVpcdsbKrLmDZAtcvy\/Ikhpn5kJ38blitlZUKU0G3fy6bJSGNe"version":"1.0","ip":"101.84.10.123, 0.0.0.0, 101.84.10.123, 117.23.61.146","device_id":"9bDHqR8u4BIVs3QYyWEGgnJgjwuhG4xT438QwkSZJFZply3uUdnJlg==","slt":1628645228,"event_time":"1628634470003","user_id":"3741591312","app_key":"com.test.client.ios","type":"app"}
```

## 问题分析

因为json文本不合法，且openresty输出的是合法json文本。 只能有一个原因。  数据被覆盖。 重点关注日志写入逻辑

```
local _M = {}

local function build_file_path(dirname)
    local now = os.time()
    return dirname .. "/collect." .. os.date("%Y%m%d%H", now) .. ".log"
end

-- 批量输出日志
function _M.write(dirname, log_arr)
    -- 文件路径
    local path = build_file_path(dirname)

    -- 打开文件
    local file = io.open(path, 'a')
    if not file then
        ngx.log(ngx.ERR, '文件无法打开: ' .. file_path)
        return
    end

    -- 将日志顺序写到磁盘上
    for _, log_item in ipairs(log_arr) do
        file:write(log_item .. '\n')
    end

    -- 关闭文件
    file:close()
end
```

使用方式为

```
local log = require("log")
-- 输出日志
if ngx.ctx.log_arr ~= nil then
        log.write(ngx.var.collect_path, ngx.ctx.log_arr)
end
```

引入log包， 调用write写入方法

write写入方法逻辑有:

1. 获取待写入的文件路径
2. 打开文件
3. 将文件顺序写入到磁盘上
4. 关闭文件句柄

逻辑较清晰， 这里出问题的大概率是`io.open`, `write`上， 因为存在多个请求并发写， 是否并发写入导致的问题呢？

经过google排查，发现 确实有反馈 io.open自身缓存(lua的io层还是调用fopen）， 会导致覆盖原有数据。
给出的解决方案也比较简单， 换成直接系统调用方法(open,write方法)。

但并没有给出如何在lua代码中调用 systemcall 的open, write方法。

lua程序中，需要借助`ffi` 库，直接调用`c`的系统调用方法， ffi库允许Lua代码调用外部C函数。

### 如何在lua代码中调用外部C函数？

1. 加载ffi库
2. 调用`ffi.cdef`为使用的C函数进行声明， 在lua中，只能调用已声明的
3. 调用步骤2中声明的C函数

### 借助ffi包实现直接调用write写文件

```
local _M = {}

local function build_file_path(dirname)
    local now = os.time()
    return dirname .. "/collect." .. os.date("%Y%m%d%H", now) .. ".log"
end

-- io.open使用clib的fopen方法， 存在缓存， 所以直接采用系统调用write, 避免缓存
-- 参考文章：https://qmsheng.github.io/2016/09/21/openresty-log/
local ffi = require "ffi"

ffi.cdef[[
size_t strlen(const char *str);
int open(const char *pathname, int flags, int mode);
ssize_t write(int fd, const void *buf, size_t count);
int close(int fd);
static const int O_RDWR   = 02;
static const int O_CREAT  = 0100;
static const int O_APPEND = 02000;
static const int S_IRUSR  = 00400;
static const int S_IWUSR  = 00200;
char *strerror(int errnum);
]]

-- 批量输出日志
function _M.write(dirname, log_arr)
    -- 文件路径
    local path = build_file_path(dirname)
    -- 打开文件 O_APPEND
    local fd = ffi.C.open(path, bit.bor(ffi.C.O_RDWR, ffi.C.O_CREAT, ffi.C.O_APPEND), bit.bor(ffi.C.S_IRUSR, ffi.C.S_IWUSR))
    if fd < 0 then
        ngx.log(ngx.ERR, '文件无法打开: ' .. path..ffi.string(ffi.C.strerror(ffi.errno())))
        return
    end
    ---- 将日志顺序写到磁盘上
    for _, log_item in ipairs(log_arr) do
        local str = log_item .. '\n'
        ffi.C.write(fd, str, ffi.C.strlen(str))
    end
    ffi.C.close(fd)
end

return _M
```

重新上线后， 没有再出来新的串行问题

## 在Nginx Worker中共享变量

上述写文件中， 会持续的open， close 文件句柄， 造成性能的损耗。 我们试想， 怎么才能共用文件句柄？

这里就需要讲到nginx的worker方式， worker即工作线程， 每个worker会处理多个网络请求(epoll网络模型)，使用`require`导入模块后， 此模块下的局部变量和数据和代码都归此worker下的所有网络请求共享。

数据共享 是基于 worker下的所有请求， 而不是说所有worker共享此数据。通常建议是worker下，只共享只读数据，也可以共享可修改的数据， 但必须保证模块中没有 非阻塞IO操作， 即不主动交出Nginx 事件循环和nginx_lua模块的轻量级线程调度，要十分小心共享可修改数据，出现bug非常不容易排查。

如果想在不同worker之间，共享数据，有以下方式：
1. 使用ngx.shared.DICT 方式
2. 使用1个worker和1个server
3. 借助第三方缓存方式， 比如redis

那么这里我们就可以声明`local fd`,  1个worker下多个请求共享文件句柄



## 参考文章

* [openresty官方文档](https://github.com/openresty/lua-nginx-module#data-sharing-within-an-nginx-worker)
* [openresty最佳实践](https://wiki.jikexueyuan.com/project/openresty/)
* [ffi共享数据](https://forum.openresty.us/d/5116-5dc67692009b28a1e73ae1e5fcb3e159)
* [openresty串数据](https://qmsheng.github.io/2016/09/21/openresty-log/)
* [ffi使用](https://blog.csdn.net/weixin_40703303/article/details/104390497)
* [鱼儿老师设计过程](https://yuerblog.cc/2018/02/13/learn-openresty/)






