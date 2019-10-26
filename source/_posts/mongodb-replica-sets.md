---
title: mongodb的Replica Sets
date: 2017-12-10 15:18:47
tags: [mongodb,centos]
---

## 导言
1. 我使用了3台服务器来部署Replica Sets，分别为主节点ma、从节点mb、仲裁节点mc(这里的ma、mb、mc已经被我配置进了hosts，可以直接被解析为ip)。
2. 其中主节点负责写与实时读，从节点负责非实时读（主从数据同步存在延迟）。
3. 仲裁节点不提供数据读写支持，但会在主节点停止工作后将从节点升级为主节点。

## 一. 安装
新建文件**/etc/yum.repos.d/mongodb-enterprise.repo**，内容如下：
```ini
[mongodb-enterprise]
name=MongoDB Enterprise Repository
baseurl=https://repo.mongodb.com/yum/redhat/$releasever/mongodb-enterprise/3.4/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc
```
然后直接通过yum安装：
```sh
yum install mongodb-enterprise
```

## 二. 编辑配置文件
执行`vi /etc/mongod.conf`，配置内容如下：
```
#monogodb

systemLog:
  destination: file #日志输出目的地
  logAppend: true #当mongod/mongos重启后，是否在现有日志的尾部继续添加日志
  path: /var/log/mongodb/mongod.log #日志路径

storage:
  dbPath: /var/lib/mongo #mongod进程存储数据目录
  indexBuildRetry: true #当构建索引时mongod意外关闭，那么再次启动重新构建索引
  directoryPerDB: true #不同DB的数据存储在不同的目录中
  journal:
    enabled: true #开启journal日志持久存储，journal日志用来数据恢复

processManagement:
  fork: true #主进程fork出一个子进程在后台执行
  pidFilePath: /var/run/mongodb/mongod.pid #pid文件地址

net:
  port: 27017  #默认端口

security:
  authorization: enabled #是否开启用户访问控制
  clusterAuthMode: keyFile #集群中members之间的认证模式
  keyFile: /etc/mongodb-keyfile #keyfile的位置
  javascriptEnabled: false #是否关闭server端的javascript功能


replication:
  replSetName: rs0 #复制集的名称
  oplogSizeMB: 8192 #replication操作日志的最大尺寸，单位：MB
```

## 三. 在全部节点上都需要进行步骤一和步骤二，全部完成后再进行以下操作


## 四. 配置集群间认证文件mongodb-keyfile
1 . 在主节点ma上执行`openssl rand -base64 745 > /etc/mongodb-keyfile`。

2 . 将生成的**/etc/mongodb-keyfile**文件复制到其他节点的相同路径。

3 . 所有的节点上修改该文件的拥有者与权限：
```sh
chmod 600 /etc/mongodb-keyfile
chown mongod:mongod /etc/mongodb-keyfile
```

## 五. 在主节点进行集群初始化和管理员账号设置
1 . 编辑主节点的配置文件**/etc/mongod.conf**，将`security`下的`authorization`编辑为`disabled`。
> authorization为enable时无法创建管理员账号。

2 . 主节点执行以下指令：
```sh
systemctl start mongod.service
systemctl enable mongod.service
```
3 . 执行`mongo`进入mongo shell，然后执行以下指令：
```js
rs.initiate()
db.createUser({user: "root",pwd: "非常复杂的超级管理员密码",roles: [ { role: "root", db: "admin" } ]});
```
4 . 退出mongo shell，编辑**/etc/mongod.conf**，将`security`下的`authorization`改回为`enabled`。
5 . 执行 `systemctl restart mongod.service`来重启mongodb服务。


## 六. 启动其他节点的mongodb服务
在其他节点上启动mongodb服务：
```sh
systemctl start mongod.service
systemctl enable mongod.service
```


## 七. 把其他节点添加到集群
> 第五步中已经在主节点里对集群进行了初始化，然后第六步中启动了从节点和仲裁节点的mongodb服务，现在就差将它们加入集群了。

1 . 在主节点执行`mongo`进入mongo shell，执行以下指令完成用户认证：
```js
use admin
db.auth('root', '非常复杂的超级管理员密码')
```

2 . 添加从节点
```js
rs.add('mb:27017')
```
3 . 添加仲裁节点
```js
rs.addArb('mc:27017')
```

## 八. 创建业务用的数据库与账号
```js
use test_db
db.createUser({user:'test_user',pwd:'复杂的test_user的密码',roles:[{role:"readWrite",db:"test_db"}]})
```
现在，业务里可以通过连接字符串`mongodb://test_user:复杂的test_user的密码@ma:27017,mb:27017/test_db`来对该集群进行读写了。