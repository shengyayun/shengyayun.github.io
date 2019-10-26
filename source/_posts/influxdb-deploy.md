---
title: influxdb的部署
date: 2017-09-30 21:16:53
tags: [influxdb,centos]
---

## 导言
influxdb是目前比较流行的时间序列数据库，本文只介绍如何部署influxdb，具体知识点请查阅相关资料。

## 一. 下载安装
```sh
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.3.5.x86_64.rpm
yum localinstall influxdb-1.3.5.x86_64.rpm
```


## 二. 服务的运行和开机自启动
```sh
systemctl start influxdb
systemctl enable influxdb
```


## 三. 使用influx shell与influxdb进行交互，创建一个管理员账号
```sql
influx
create user admin with password 'admin' with all privileges to admin;
```


## 四. 修改配置文件启用权限认证
1. 编辑/etc/influxdb/influxdb.conf文件，将`[http]`下的`auth-enabled`修改为`true`，然后重启服务。
2. 以后每次登陆influxdb shell，执行指令前都需要先执行`auth`指令通过身份认证。

## 五. 修改默认端口
1. 编辑/etc/influxdb/influxdb.conf文件，将`[http]`下的`bind-address = ":8086"`中的`8086`修改为`8087`，改完后重启服务。
2. 换了端口后，进入influx shell的指令就变为了`influx -port 8087`

## 六. 测试
1. 先通过influx shell执行一条创建数据库的语句：
```sql
create database testDb;
```
2. 在bash中通过curl进行一次post请求：
```sh
curl -i -X POST "http://localhost:8087/write?db=testDb&u=admin&p=admin" --data-binary "testMetric,host=mbp value=0.64"
#这里的`db=testDb`就是数据库，`u=admin`和`p=admin`分别是第三步中设置的用户名和密码，`testMetric`是度量，`host=mbp`是标签，`value=0.64`是数据
```
3. 在influx shell中执行查询语句：
```sql
use testDb; //使用testDb数据库
select * from testMetric; //查询度量testMetric
```
