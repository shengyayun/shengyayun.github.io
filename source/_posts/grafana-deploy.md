---
title: grafana的部署
date: 2017-09-30 21:34:48
tags: [grafana,centos]
---

## 导言
grafana 是基于JS开发的，功能齐全的度量仪表盘和图形编辑器。我们可以通过collectd和业务代码采集数据，通过influxdb存储数据，最后通过grafana的web端进行展现。

## 一. 安装
```sh
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-4.5.2-1.x86_64.rpm
sudo yum localinstall grafana-4.5.2-1.x86_64.rpm
```

## 二. http配置修改
用vim打开文件/etc/grafana/grafana.ini，下面我们查看`[server]`的配置：
+ `http_port`默认是`3000`，代表它自带的http服务默认监听3000端口。
+ `domain`默认是`localhost`，结合上面的默认端口，我们可以通过`http://localhost:3000`来访问grafana的图形化界面。
+ `root_url`这个配置主要为了在使用反向代理后能正常跳转，同时邮件可以提供正确的url。

这里有个案例，我希望可以通过`http://watch.langdaren.com`直接访问grafana，但80端口已经被nginx占用：
1. `http_port`不修改，保留默认的`3000`。
2. `domain`改为`watch.langdaren.com`。
3. `root_url`改为`http://watch.langdaren.com`。
4. 修改nginx配置，添加一条这样的server配置：
```nginx
server{
　　listen 80;
　　server_name  watch.langdaren.com;
　　location / {
　　　　proxy_pass   http://127.0.0.1:3000;
　　}	
}
```
nginx配置修改完记得先执行`nginx -t`测试一下配置，然后再执行`nginx -s reload`进行重启。

## 三. smtp配置修改
配置正确的smtp，可以让grafana可以自动发送邮件，下面我使用我的腾讯邮箱进行配置。
用vim打开配置文件，下面我们查看`[smtp]`的配置：
1. `enabled`修改为`true`。
2. `host`修改为邮件系统的发送服务器地址，这里是`smtp.exmail.qq.com:465`。
3. `user`修改为我的邮箱全名`719048774@qq.com`。
4. `password`修改为我的邮箱密码。
5. `from_address`修改为邮箱全名`719048774@qq.com`。
6. `from_name`改为`shengyayun`。

## 四. 管理员账号配置修改
用vim打开配置文件，下面我们查看`[security]`的配置：
1. `admin_user`默认为`admin`，这个是管理员账号名，按需修改。
2. `admin_password`默认为`admin`，这个是管理员密码，按需修改。

## 五. 服务运行与开机自启动
```sh
systemctl start grafana-server
systemctl enable grafana-server
```

## 六. 测试
通过第二步的http配置，就知道如何访问grafana的web端了。默认的地址是`http://localhost:3000`，在我的案例里的地址是`http://watch.langdaren.com`。

## 七. 其他
grafana访问influxdb并创建dashboard之类操作，都是通过grafana的web端的管理员功能完成的，这里不做详细介绍，具体可以查看官网`http://docs.grafana.org`。

