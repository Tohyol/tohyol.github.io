---
layout:     post
title:      "Docker-Compose的安装与配置"
tags:
    - Docker
    - Linux
---

## 安装
```
# 下载二进制文件
> curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

# 国内下载地址
> curl -L https://get.daocloud.io/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

# 校验完整性
> sha256sum /usr/local/bin/docker-compose

# 赋予可执行权限
> chmod +x /usr/local/bin/docker-compose

# 查看版本号
> docker-compose version
```

## 常用命令
```
# 拉取镜像
> docker-compose pull

# 创建并前台启动容器
> docker-compose up

# 创建并后台启动容器
> docker-compose up -d

# 指定YAML配置文件，进行处理
> docker-compose -f docker-compose.yml up -d

# 查看容器的运行状态
> docker-compose ps

# 启动容器
> docker-compose start

# 停止容器
> docker-compose stop

# 停止并删除容器，包括网络、数据卷（特别注意，此操作会删除所有容器的数据，且数据不可恢复）
> docker-compose down
```

## docker-compose.yml模板
```
version: "3.8"
services:
  portainer:
    container_name: portainer
    image: portainer/portainer
    ports:
      - 9000:9000
    volumes:
      - /var/run/docker.sock
  mysql:
    container_name: mysql
    image: mysql:latest
    restart: always
    ports:
      - 3306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=123456
  redis:
    container_name: redis
    image: redis:latest
    restart: always
    expose:
      - 80
    volumes:
      - /home/docker/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - /home/docker/redis/logs:/logs
  tomcat:
    container_name: tomcat
    image: tomcat:latest
    restart: always
    expose:
      - 8080
    depends_on:
      - redis
    volumes:
      - /home/docker/tomcat/webapps:/usr/local/tomcat/webapps/
      - /home/docker/tomcat/logs:/usr/local/tomcat/logs/
  nginx:
    container_name: nginx
    image: nginx:latest
    restart: always
    ports:
      - 80:80
    depends_on:
      - tomcat
    links:
      - tomcat
    volumes:
      - /home/docker/nginx/conf:/etc/nginx/conf.d
      - /home/docker/nginx/log:/var/log/nginx
    environment:
      - TZ=Asia/Shanghai

参数说明：
container_name：自定义容器命名
image：指定服务的镜像名称或镜像ID，如果镜像在本地不存在，Compose将会尝试拉取镜像
commond：覆盖容器启动后默认执行的命令
ports：用于映射端口
expose：暴露端口，但不映射到宿主机，只允许能被连接的服务访问
depends_on：用于解决容器的依赖、启动先后的问题
links：链接到其它服务中的容器
volumes：挂载一个目录或者一个已存在的数据卷容器
environment：添加环境系统配置
```

## 卸载
```
rm -rf /usr/local/bin/docker-compose
```
