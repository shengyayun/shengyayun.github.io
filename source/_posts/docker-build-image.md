---
title: 通过Dockerfile生成镜像
date: 2019-04-29 20:54:48
tags: [docker,centos]
---

> [Get Started, Part 2: Containers](https://docs.docker.com/get-started/part2/)
> [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

## 导言
这里将通过一个简单的场景来学习Dockerfile的使用：制作一个安装了openresty的centos的镜像（image）。
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

## 一. 创建一个工作目录
```sh
# 后续这个目录将存放Dockerfile和entrypoint.sh
mkdir hello
```

## 二. 在工作目录中创建一个Dockerfile文件
```sh
# 指定基础镜像，并且必须是第一条指令（如果不需要基础镜像，那么替换为 FROM scratch）
# 这里的centos是镜像名，7是标签。这里对应系统centos 7
FROM centos:7

# 容器的工作路径，对后续的RUN, CMD, ENTRYPOINT, COPY, ADD生效（如果对应的目录不存在则会创建，可以重复设置）
# 这里会在容器中创建一个/app文件夹，下面的对应操作会在/app中进行
WORKDIR /app

# 从本地拷贝文件到容器
# 这里会把本地的demo目录内的全部文件拷贝到容器的/app目录下（./指当前目录，而当前目录通过WORKDIR定义）
COPY * ./

# 容器在build时执行指令
RUN yum install -y lsof git
RUN yum install -y yum-utils
RUN yum-config-manager -y --add-repo https://openresty.org/package/centos/openresty.repo
RUN yum install -y openresty

# 容器启动时执行指令，CMD命令的功能类似，但主要区别在于CMD可被dock run的COMMAND参数覆盖，而ENTRYPOINT不会
# 如果CMD或ENTRYPOINT配置了多条，且CMD指令不是完整的可执行命令，那么CMD的内容将成为ENTRYPOINT的参数。反之仅最后一条生效。
# 当容器启动时会执行/app/entrypoint.sh文件（build的时候不会执行），该文件在上面的COPY操作中被docker从本地的工作目录拷贝到了容器的/app目录
ENTRYPOINT ./entrypoint.sh

# 对外开放端口
# 若果不将80端口对外暴露，容器外将无法访问容器的80端口
EXPOSE 80
```

## 三. 在工作目录中创建一个entrypoint.sh文件
```sh
#!/bin/sh

# 启动openresty服务
openresty

# docker会将没有前台守护进程的容器直接关闭，所以这里通过本指令阻塞脚本的执行
tail -f /dev/null
```
文件生成完成后，记得还要通过`chmod +x entrypoint.sh`给脚本加上执行权限，否则容器启动时将无法执行脚本。

## 四. 在工作目录下build镜像
> Usage:	docker build [OPTIONS] PATH | URL | -

```sh
# 待生成镜像的name为hello，tag为缺省值latest，路径为.（当前工作目录）
docker build -t hello .
# 指令执行完成后，控制台输出镜像完整的ID
```
执行``docker image ls``，docker会打印出本地存储的全部镜像，其中REPOSITORY为hello，TAG为latest的记录对应刚创建的镜像。

## 五. 创建容器
> Usage:	docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

```sh
# -d 该容器在后台执行
# -p 4000:80 将容器的80端口映射到本地的4000端口（通过localhost:4000访问容器的openresty的80端口）。
# hello 使用镜像hello
docker run -d -p 4000:80 hello
# 指令执行完成后，控制台输出容器完整的ID
```
执行``docker container ls``，docker会打印出本地执行中的全部容器，其中IMAGE为hello的记录对应刚启动的容器。
打开浏览器访问``http://localhost:4000``，可以看到openrestry的默认主页，至此镜像及其对应的容器已创建完成。


