---
title: 部署一个单机模式的Nacos
date: 2020-05-02 10:50:14
tags: [nacos,centos]
---

> 部署于Centos7系统，需要额外安装MySQL、JDK8+。

## 一. MySQL
```sh
# 下载Centos7使用的MySQL5.7
# 截止2020-05-03，Nacos最新版本不支持MySQL8+
wget http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
yum localinstall mysql57-community-release-el7-10.noarch.rpm
yum install mysql-community-server

# 启用MySQL服务
systemctl start mysqld

# 从mysql日志中查找root的一次性密码
grep password /var/log/mysqld.log
# 2020-05-02T02:37:53.274469Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: )jm8BYphsfa>
# 发现系统默认生成的root密码为 )jm8BYphsfa>

# 更新root密码，否则无法执行其他sql
mysql -uroot -p')jm8BYphsfa>'
alter user 'root'@'localhost' identified by '一个很复杂很安全的密码';

# 创建Nacos需要的数据库
create database nacos_db;
```

## 二. JDK8+

> 不要忘记设置PATH、JAVA_HOME

## 三. Nacos
> 查找Nacos的最新的稳定包： https://github.com/alibaba/nacos/releases
```sh
# 下载最新的文档包，这里是1.2.1
wget https://github.com/alibaba/nacos/releases/download/1.2.1/nacos-server-1.2.1.tar.gz
# 解压
tar xzvf nacos-server-1.2.1.tar.gz
# 将解压出的目录移动到/opt目录下（目录可自选）
mv nacos /opt/

# 编辑Nacos的配置文件，
vi /opt/nacos/conf/application.properties
##  涉及db.url.0、db.user、db.password。注意db.url.0中的数据库使用第一步中创建的nacos。

# 启用单机模式的Nacos
sh /opt/nacos/bin/startup.sh -m standalone
```

编辑Nacos的配置文件`vi /opt/nacos/conf/application.properties`：
```sh
# 取消MySQL相关的注释
spring.datasource.platform=mysql
db.num=1
# 数据库使用步骤一中创建的数据库nacos_db
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_db?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
# 不推荐使用root，可自行创建一个仅有nacos库读写权限的账户
db.user=nacos_user
db.password=nacos_passwd
```

启动单机模式的Nacos：
```sh
sh /opt/nacos/bin/startup.sh -m standalone
```
