---
title: collectd的部署
date: 2017-09-30 21:29:21
tags: [collectd,centos]
---

## 导言
collectd是一款性能监控程序，这里我将采用它对服务器进行监控，然后将数据写入influxdb。阅读该文档前先阅读influxdb的相关文档。

## 一. 安装
```sh
yum install collectd
yum install collectd-rrdtool
```

## 二. 修改配置
1. 用vim打开/etc/collectd.conf文件。
2. 将`QDNLookup`修改为`false`。如果要保留为默认的`true`的话，host必须与ip保持一致。
3. 取消`LoadPlugin logfile`的注释，同时取消`<Plugin logfile>...</Plugin>`的注释，将其中`File Stdout`一行改为`File "/var/log/collectd.log"`，这样collectd的日志会被单独写进这个文件里。
4. 取消rrdtool组件的注释：
`LoadPlugin rrdtool`
5. 为了后续测试collectd功能，取消cpu组件的注释：
`LoadPlugin cpu`
6. 通过配置network组件将监控数据传给influxdb：
```xml
<Plugin network>
　　Server "127.0.0.1" "25826"
</Plugin>
```
这里的`127.0.0.1`是influxdb的ip地址，`25826`是influxdb监听的端口号。

## 三. 配置influxdb的collectd插件
1. 用vim打开/etc/influxdb/influxdb.conf文件
2. 将`[[collectd]]`下面的`enabled`取消注释，并改为`true`。
3. 将`[[collectd]]`下面的`typesdb`取消注释，改为/etc/collectd.conf文件中`TypesDB`指向的`types.db`文件所在的**目录**。
4. 重启influxdb服务。
5. 通过influx shell创建一个新的数据库，取名collectd。

## 四. 前台执行
在测试过程中发现，如果不先在前台手动执行一次collectd，直接用`systemctl start collectd`启动服务会导致其一直报错。执行以下指令可以让collectd在前台执行：
`/usr/sbin/collectd -f`
如果没有问题的话就`ctrl+c`结束程序。

## 五.  服务运行与开机自启动
```sh
systemctl start influxdb
systemctl enable influxdb
```

## 六. 测试
通过influx shell查看数据库collectd数据库，一切正常的话，里面会出现名为cpu_value的measurement。这说明collectd的cpu组件已经在不停地往influxdb写入数据了。
