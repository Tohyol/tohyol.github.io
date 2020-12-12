---
layout:     post
title:      "【Linux】MySQL的安装及其配置"
tags:
    - Linux
    - MySQL
---

安装
----
```
# 添加依赖
wget -i -c https://repo.mysql.com//mysql80-community-release-el7-1.noarch.rpm
yum -y install mysql80-community-release-el7-1.noarch.rpm mysql-community-server

# 删除文件
yum -y remove mysql57-community-release-el7-10.noarch;
rm -rf mysql57-community-release-el7-10.noarch.rpm
```

配置
----
```
# 启动mysql
systemctl start mysqld.service;

# 获取随机密码
grep "password" /var/log/mysqld.log;

# 登录mysql数据库，输入密码
mysql -u root -p

# 设置密码策略
mysql> set global validate_password_policy=0;
mysql> set global validate_password_length=1;

# 修改密码
mysql> alter user 'root'@'localhost' IDENTIFIED BY 'password';

# 授权
# 第一个*表示数据库名；第二个*表示该数据库的表名，*.*表示所有到数据库下到所有表都允许访问；
# %表示允许访问到mysql的ip地址，%表示所有ip均可以访问；
mysql> grant all privileges on *.* to root@'%' identified by 'password';

# 刷新权限
mysql> flush privileges;

# 忘记密码
vi /etc/my.cnf
    # 添加(密码重置后，删除该行)
    skip-grant-tables
systemctl restart mysqld.service;
mysql
mysql> use mysql;
mysql> update user set authentication_string = password("password") where user="root";
mysql> flush privileges;

# 启动、重启、关闭命令

# 启动mysql
systemctl start mysqld.service;

# 重启mysql
systemctl restart mysqld.service;

# 关闭mysql
systemctl stop mysqld.service;

# 针对5.7以上版本，sql查询高效配置(my.cnf/my.ini)
sql-mode="NO_ENGINE_SUBSTITUTION"

optimizer_switch="index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,subquery_materialization_cost_based=on,use_index_extensions=on,duplicateweedout=off,condition_fanout_filter=off,derived_merge=off"

max_connections=800

key_buffer_size=32M

read_buffer_size=32M

read_rnd_buffer_size=126M

innodb_buffer_pool_size=256M

join_buffer_size=256M
```