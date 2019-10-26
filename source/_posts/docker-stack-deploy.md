---
title: 部署Docker容器到Kubernetes
date: 2019-05-13 20:52:25
tags: [docker,kubernetes,centos]
---

> [Get Started, Part 5: Stacks](https://docs.docker.com/get-started/part5/)
> [Compose file version 3 reference](https://docs.docker.com/compose/compose-file/)

## 导言
虽然docker默认的swarm与k8s（kubernetes）处于竞争关系，docker依然可以部署容器到k8s。

```
Server: Docker Engine - Community
 Engine:
  Version:          18.09.2
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.6
  Git commit:       6247962
  Built:            Sun Feb 10 04:13:06 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 Kubernetes:
  Version:          v1.10.11
  StackAPI:         v1beta2
```

## 一. vi compose.yml
文件内容如下（保存前删除注释）：
```
version: "3.7"                                        #compose file的版本，Docker Engine版本18.06.0+支持3.7
services:                                             #service中运行的单个容器被称为task，系统中的单个模块被称为service
  web:
    image: localhost:5000/hello                       #使用私有镜像localhost:5000/hello（这里必须使用仓库中的，不可以用本地的）
    deploy:                                           #部署操作
      replicas: 5                                     #部署5个副本
      resources:
        limits:
          cpus: "0.1"                                 #每个副本最多使用10%的CPU时间片
          memory: 50M                                 #每个副本最多使用50MB的RAM
      restart_policy:
        condition: on-failure                         #容器运行失败后自动重启
    ports:
      - "4000:80"                                     #本地的4000端口映射到容器的80端口
    redis:
      image: redis                                    #docker hub上的redis镜像
      ports:
        - "6379:6379"                                 #本地的6379端口映射到容器的6379端口
      volumes:
        - "/home/docker/data:/data"                   #容器的/data目录挂载到本地的/home/docker/data目录（redis会将日志文件存储到/data目录）
      deploy:
        placement:
          constraints: [node.role == manager]         #限制本service只在manager节点执行
      command: redis-server --appendonly yes          #启动redis-server，同时指定数据持久化策略：AOF模式
```

## 二. docker stack deploy

```sh
# service组成的完整的功能模块（甚至是系统）被称为stack。指定orchestrator为k8s，通过上一步保存的yml文件，生成名为hello的stack。
docker stack deploy --orchestrator=kubernetes -c compose.yml hello

# 查看k8s的全部resources
kubectl get all
# 查看docker的全部stack
docker stack ls
```
