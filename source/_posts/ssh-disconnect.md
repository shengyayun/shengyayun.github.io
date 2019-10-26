---
title: SSH自动断开的解决方法
date: 2018-09-29 16:01:25
tags: [ssh,centos]
---

在服务端(centos)执行以下指令：

```sh
echo "ClientAliveInterval 60" >> /etc/ssh/sshd_config  
echo "ClientAliveCountMax 1" >> /etc/ssh/sshd_config  
service sshd restart
```