---
layout:     post
title:      "【Linux】Nginx的安装及其配置"
tags:
    - Linux
    - Nginx
---

安装
----
```
# 添加依赖
yum -y install wget gcc gcc-c++ pcre-devel zlib zlib-devel openssl openssl-devel

# 下载文件
cd /home
wget http://nginx.org/download/nginx-1.15.8.tar.gz

# 解压文件
tar -zxvf nginx-1.15.8.tar.gz

# 删除文件
rm -rf nginx-1.15.8.tar.gz

# 安装
cd nginx-1.15.8
./configure --with-http_ssl_module
make
make install
```

配置
----
```
# 配置https
server {
    listen                     80;
    listen                     443 ssl;
    server_name                localhost;
    
    # 配置证书位置
    ssl_certificate            /home/keys/server.crt;
    # 配置秘钥位置
    ssl_certificate_key        /home/keys/server.key;
    # 双向认证
    #ssl_client_certificate    ca.crt;
    # 双向认证
    #ssl_verify_client         on;
    ssl_session_timeout        5m;
    ssl_protocols              SSLv2 SSLv3 TLSv1 TLSv1.2;
    ssl_ciphers		           ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
    ssl_prefer_server_ciphers  on;
    ...
}

# 修改nginx名称及版本，伪装任意web server
vi scr/core/nginx.h
    #define NGINX_VERSION      "1.0.0"
    #define NGINX_VER          "test/" NGINX_VERSION

# 启动
/usr/local/nginx/sbin/nginx -c /home/nginx-1.15.8/conf/nginx.conf

# 关闭
pkill -9 nginx
```