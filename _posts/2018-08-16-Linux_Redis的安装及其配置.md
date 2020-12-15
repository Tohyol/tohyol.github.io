---
layout:     post
title:      "【Linux】Redis的安装及其配置"
tags:
    - Linux
    - Redis
---

## 安装
```
# 安装依赖
yum -y install wget gcc

# 下载文件
cd /home
wget http://download.redis.io/releases/redis-5.0.3.tar.gz

# 解压文件
tar -zxvf redis-5.0.3.tar.gz

# 删除文件
rm -rf redis-5.0.3.tar.gz

# 安装
cd redis-5.0.3
make(make MALLOC=libc)
cd src
make install
```

## 配置
```
# 设置密码
cd /home/redis-5.0.3
vi redis.conf
    port 6666
    daemonize yes
    requirepass 123456

# 启动
/home/redis-5.0.3/src/redis-server /home/redis-5.0.3/redis.conf

# 关闭
pkill -9 redis

# 清除缓存
# 进入redis目录
cd redis

# 连接redis
redis-cli -h 127.0.0.1 -p 6379

# 输入密码，无密码跳过
auth 123456

# 查看所有key值
keys *

# 删除指定索引的值
del key

# 清空整个 Redis 服务器的数据
flushall

# 清空当前库中的所有 key
flushdb
```
