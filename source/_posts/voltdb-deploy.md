---
title: voltdb的部署
date: 2018-09-29 16:04:52
tags: [voltdb,centos]
---

## 导言
Voltdb作为分布式的内存数据库，企业版比社区版主要多提供了命令日志(事务级别的持久化日志)、不停机的弹性扩容、用于灾难恢复的数据库备份、跨数据中心的备份。

## 一. 下载安装
```sh
wget https://downloads.voltdb.com/technologies/server/voltdb-latest.tar.gz
tar -zxvf voltdb-latest.tar.gz
mv voltdb-community-7.8.2 /usr/local/voltdb
mkdir /data/voltdb #这个目录存放voltdb的数据
```

## 二. 系统配置
1. voltdb不支持大页内存，终端执行以下指令，同时将它们写入`/etc/rc.local`文件：
```sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
2. 将voltdb的可执行文件加入PATH：
```sh
#更新Path
echo 'PATH=/usr/local/voltdb/bin:$PATH' >> /etc/profile
#终端执行以下指令立刻执行变更：
source /etc/profile
```

## 三. 初始化和运行
```sh
voltdb init --dir /data/voltdb/ #如果以前启动过，需要强制初始化，加上 --force
voltdb start --dir /data/voltdb/
```

## 四. 测试数据插入
1 . 终端执行`sqlcmd`进入sql终端：
```sh
create table test(id int); #创建测试表
create procedure ping as insert into test(id) values(?); #创建存储过程
```
2 . 在终端通过http执行存储过程：
```sh
curl -d "Procedure=ping&Parameters=%5B0%5D" "http://localhost:8080/api/1.0/"
#其中%5B0%5D未转码前是`[0]`，所以这里等同于 `call ping (0)`
```


## 五. 常用指令
```sh
voltadmin save #立刻生成快照
voltadmin shutdown #停止集群(不要直接停止voltdb进程，应该使用voltadmin)
voltadmin shutdown —save #停止集群并生成快照
voltadmin pause #停止客户端服务，进入维护模式
voltadmin resume #退出维护模式
voltdb start --pause
voltadmin save --blocking {path} {file-prefix} #在path目录下生成file-prefix为前缀的备份文件
voltadmin restore {path} {file-prefix}
```

## 六. 使用须知
1. 所有节点的内存大小与核数应保持一致。
1. 每个节点的默认分区数为8，推荐分区数为：cpu核数 × 0.75 (非超线程处理器)。
1. 每个分区有且只有一个线程读写，不存在锁冲突。分区越多并发越高，自然性能也越高。
1. 一个表只有一个分区列，每个表通过hash分区列，把每行数据分布到不同的分区。
1. 合理的分区列直接决定该表的增删改查性能。
1. 如果分区表存在主键，那么主键中必须包含分区列。
1. 分区列不能为空，且必须为`TINYINT`、`SMALLINT`、 `INTEGER`、`BIGINT`、`VARCHAR`类型。
1. 没有分区列的表是复制表，全部分区都会保存一份。所以虽然插入与存储成本高，但是查询成本低，适合更新较少但查询较多的小表。
1. 存在数据的列无法添加唯一、假定唯一、主键约束。
1. 被存储过程引用的表无法直接删除，需先删除对应存储过程。被索引和视图引用的表也无法直接删除，但是可以用`DROP TABLE 表名 IF EXISTS CASCADE`来级联删除。
1. K-Safe的值代表可以保证数据不丢失的最小可以同时发生故障的服务器数。假设为K-Safe值为`n`，节点数为`a`，每个节点分区数为`b`，那么一条数据有`n+1`个拷贝，每`a×b/(n+1)`个分区存有一份完整的数据。
1. 集群间每台服务器需开启NTP保证节点间时间差异不超过200ms。
1. 存储过程返回结果不能超过50MB。 