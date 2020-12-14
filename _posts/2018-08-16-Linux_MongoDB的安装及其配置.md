---
layout:     post
title:      "【Linux】MongoDB的安装及其配置"
tags:
    - Linux
    - MongoDB
---

## 安装
```
# 添加依赖
yum -y install wget

# 下载文件
cd /usr/local
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-4.0.3.tgz

# 解压文件
tar -zxvf mongodb-linux-x86_64-4.0.3.tgz
mv mongodb-linux-x86_64-4.0.3 mongodb

# 删除文件
rm -rf mongodb-linux-x86_64-4.0.3.tgz
```

## 配置
```
# 创建配置文件
cd mongodb
mkdir data
mkdir logs
mkdir etc
touch etc/mongo.conf
vi etc/mongo.conf
    dbpath=/usr/local/mongodb/data
    logpath=/usr/local/mongodb/logs/mongo.log
    fork=true
    bind_ip=0.0.0.0
touch logs/mongo.log

# 启动
/usr/local/mongodb/bin/mongod -f /usr/local/mongodb/etc/mongo.conf --fork

# 关闭
/usr/local/mongodb/bin/mongod -shutdown -dbpath=/usr/local/mongodb/data/

# 创建用户
/usr/local/mongodb/bin/mongod -f /usr/local/mongodb/etc/mongo.conf --fork
./bin/mongo
use xxx
db.createUser({user:"root",pwd:"123456",roles:[{role:"readWrite",db:"xxx"}]})
db.auth("root","123456")
/usr/local/mongodb/bin/mongod -shutdown -dbpath=/usr/local/mongodb/data/
```
