---
title: SSH的常用指令
date: 2021-03-27 11:35:09
tags:
---

```sh
# 生成密钥对
ssh-keygen -t rsa -P '' -m PEM

# 权限设置
chmod 700 .ssh 
chmod 600 .ssh/authorized_keys

# 转换OpenSSH格式
puttygen id_rsa -o id_rsa.ppk
puttygen id_rsa.ppk -O private-openssh -o id_rsa2

# 排查异常登录的IP使用的公钥
cat /var/log/secure | grep 115.192.217.112
# Accepted publickey for root from 115.192.217.112 port 57305 ssh2: RSA SHA256:4egcXPHAIXpN7XxEiBQVV5dWEtQvi4JzbicrBJRZp1I
ssh-keygen -lf ~/.ssh/authorized_keys | grep 4egcXPHAIXpN7XxEiBQVV5dWEtQvi4JzbicrBJRZp1I
```