---
title: nginx站点访问日志
date: 2018-04-26 19:42:30
tags: nginx
---

## 一. 在nginx.conf的`http`块中添加日志格式`main2018`：

```nginx
log_format main2018 '$remote_addr|$remote_user|$time_local|$request|$status|$body_bytes_sent|$http_referer|$http_user_agent|$http_x_forwarded_for|$request_time|$upstream_response_time|$upstream_addr|$upstream_status';
```


## 二. 在对应项目的`server`中配置该站点的访问日志：

```nginx
access_log /var/log/www.access main2018; #使用第一步中添加的格式main2018
```


## 三. 重启nginx

```sh
nginx -t #判断配置正确
nginx -s reload
```

## 四. 查看访问日志

在服务器上执行如下指令查看最近一次访问的记录：

```sh
tail -n 1 /var/log/www.access | awk -F "|" '{printf "客户端地址: %s\n访问时间和时区: %s\n客户端用户名称: %s\n请求的URI和HTTP协议: %s\nHTTP请求状态: %s\n发送给客户端文件内容大小: %s\nurl跳转来源: %s\n客户端信息: %s\nHTTP的请求端真实的IP: %s\n请求的总时间: %s\nupstream响应时间: %s\nupstream地址: %s\nupstream状态: %s\n\n",$1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13}
```

结果是：

```
客户端地址: 115.238.29.221
访问时间和时区: -
客户端用户名称: 26/Apr/2018:17:29:20 +0800
请求的URI和HTTP协议: POST /api HTTP/1.1
HTTP请求状态: 200
发送给客户端文件内容大小: 260
url跳转来源: -
客户端信息: okhttp/3.8.1
HTTP的请求端真实的IP: 115.238.29.223
请求的总时间: 0.017
upstream响应时间: 0.017
upstream地址: 127.0.0.1:9000
upstream状态: 200
```