# Redis Lua脚本

## 1 Redis为什么引入Lua脚本？

## 1.1 Lua脚本的好处

1. 减少网络开销，可以将多条命令放在同一Lua脚本中执行。

2. 原子性操作，Redis会把整个Lua脚本当作一个整体去执行，保证不会插入其他命令。

3. 复用性，客户端发送的脚本会被Redis永久存储，其它客户端也可以复用。

## 2 什么是Lua脚本？

    Lua脚本是基于标准C语言编写的，是一种轻量小巧的语言。常用于游戏开发，独立应用脚本，数据库插件等。

### 2.1 Redis中Lua常用命令

- EVAL命令 `EVAL script numkeys key [key...] arg [arg...]`
  
  用于执行一段Lua缓存，每次执行EVAL命令redis都会将脚本的SHA1摘要加入脚本缓存中，下次可以直接使用EVALSHA命令调用脚本。
  
  - script：Lua5.1版本脚本程序（script不用被定义为一个Lua函数）
  
  - numkeys：script中要使用的key个数
  
  - key [key...]：script使用Redis的key具体键值，在脚本中通过KEYS[index]访问
  
  - arg [arg...]：附加参数，在脚本中通过ARGV[index]访问

- SCRIPT LOAD命令 `SCRIPT LOAD script`
  
  该命令用于将脚本加入脚本缓存且不执行脚本，该命令会返回脚本的SHA1摘要。
  
  - script：脚本内容

```shell
127.0.0.1:16390> script load "return 1"
"e0e1f9fabfc9d4800c877a703b823ac0578ff8db"
```

- SCRIPT EXISTS命令 `SCRIPT EXISTS sha1`
  
  该命令用于判断sha1脚本是否被缓存。
  
  - sha1：脚本的sha1摘要

- SCRIPT FLUSH命令

        清空脚本缓存

- SCRIPT KILL命令

        强制终止当前脚本执行

# Redis管道技术

    Redis管道技术可以在服务端未响应时，客户端可以继续向服务端发送请求，最终一次性读取所有服务端的响应。

    管道技术最大的优势是可以减少网络开销。单条命令执行时间 = 客户端发送时间+redis处理和返回时间+一个来回网络时间。网络的时间是不稳定的，redis每秒可以执行10条左右的时间如果网络花费250ms那么每秒只能执行4个请求。

    需要注意的是管道中命令的数量不能太多，如果太多也会造成I/O和网络传输压力，并且redis会将管道中的命令存入队列顺序执行，会消耗额外的内存。

# 
