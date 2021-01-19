---
layout:     post
title:      "【Linux】Docker-Compose的安装及其配置"
tags:
    - Linux
    - Docker
---

## 安装
```
# 下载二进制文件
curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
curl -L https://get.daocloud.io/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

# 校验完整性
sha256sum /usr/local/bin/docker-compose

# 赋予可执行权限
chmod +x /usr/local/bin/docker-compose

# 查看版本号
docker-compose version
```

## 常用命令
```
# 拉取镜像
docker-compose pull

# 创建并前台启动容器
docker-compose up

# 创建并后台启动容器
docker-compose up -d

# 指定YAML配置文件，进行处理
docker-compose -f docker-compose.yml up -d

# 查看容器的运行状态
docker-compose ps

# 启动容器
docker-compose start

# 停止容器
docker-compose stop

# 停止并删除容器，包括网络、数据卷（特别注意，此操作会删除所有容器的数据，且数据不可恢复）
docker-compose down
```

## docker-compose.yml模板
```
version: "3.8"
services:
  portainer:
    image: portainer/portainer
    ports:
      - 9000:9000
    volumes:
      - /var/run/docker.sock
  redis:
    image: redis:latest
    restart: always
    expose:
      - 80
    volumes:
      - /home/docker/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - /home/docker/redis/logs:/logs
  tomcat:
    image: tomcat:latest
    restart: always
    expose:
      - 8080
    volumes:
      - /home/docker/tomcat/webapps:/usr/local/tomcat/webapps/
      - /home/docker/tomcat/logs:/usr/local/tomcat/logs/
  nginx:
    image: nginx:latest
    restart: always
    ports:
      - 80:80
    links:
      - tomcat
    volumes:
      - /home/docker/nginx/conf:/etc/nginx/conf.d
      - /home/docker/nginx/log:/var/log/nginx
    environment:
      - TZ=Asia/Shanghai
```
