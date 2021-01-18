---
layout:     post
title:      "【Linux】Docker的安装及其配置"
tags:
    - Linux
    - Docker
---

## 安装
```
# 添加依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 设置镜像源(可选)
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 

# 安装
yum install -y docker-ce

# 加速器(腾讯加速器)
vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}

# 启动服务  
systemctl start docker
```

## 可视化图形工具
```
docker search portainer
docker pull portainer/portainer
docker run -d -p 9000:9000 --name prtainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer
```

## 常用命令
```
# 开机自启动
systemctl enable docker

# 启动服务  
systemctl start docker

# 查看服务
systemctl status docker

# 重启服务
systemctl daemon-reload
systemctl restart docker

# 关闭服务
systemctl stop docker

# 拉取及安装容器
docker pull <image>
docker run -d -p 9000:9000 --name <name> --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /home/data:/opt/data <image>/<image>

参数说明：
-d：容器在后台运行
-p 9000:9000：宿主机9000端口映射容器中的9000端口
--name：名称
--restart=always：自启动
-v /var/run/docker.sock:/var/run/docker.sock：添加到Docker的守护进程中
-v /home/data:/opt/data： 宿主机/home/data映射容器/opt/data

# 查看已下载镜像
docker images

# 查看正在运行的容器
docker ps

# 查看所有的容器
docker ps -a 

# 查看容器日志
docker logs -f <容器名 or ID>

# 删除容器
docker rm <容器名 or ID>

# 删除所有容器
docker rm $(docker ps -a -q)

# 停止、启动、杀死指定容器
docker start <容器名 or ID>
docker stop <容器名 or ID>
docker kill <容器名 or ID>

# 进入容器
docker exec -it <容器名 or ID> /bin/bash
```
