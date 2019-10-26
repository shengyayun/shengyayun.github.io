---
title: 创建Docker私有仓库
date: 2019-05-04 10:31:37
tags: [docker,centos]
---
> [Deploy a registry server](https://docs.docker.com/registry/deploying/)
> [Use volumes](https://docs.docker.com/storage/volumes/)
> [Use bind mounts](https://docs.docker.com/storage/bind-mounts/)
> [Garbage collection](https://docs.docker.com/registry/garbage-collection/)
> [HTTP API V2](https://docs.docker.com/registry/spec/api/)
> [Definition of: repository](https://docs.docker.com/glossary/?term=repository)

## 导言
本文介绍如何在本地部署docker私有仓库，涉及到registry镜像（registry:2）、volume与bind（bind mount）、仓库的http api、仓库的垃圾回收等知识点。
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

## 一. 创建一个volume
```sh
#创建一个名为registry-vol的卷
docker volume create registry-vol

#执行以下指令，可以看到有一条VOLUME NAME为registry-vol的记录
docker volume ls
```
与bind直接使用宿主的文件系统不同，volume由docker直接生成与管理，它跨系统、跨平台、易于备份与迁移。

## 二. 创建私有仓库
```sh
#从docker hub拉取私有仓库的镜像
docker pull registry:2

#创建容器
# -d 在后台运行容器
# -p 5000:5000 将本地的5000(冒号左边)端口映射到容器的5000(冒号右边)端口
# --mount src=registry-vol,dst=/var/lib/registry 将名为registry-vol的volume映射到容器的/var/lib/registry目录（通过volume的方式进行数据持久化）
# --restart=always 容器自动重启
# -e REGISTRY_STORAGE_DELETE_ENABLED=true 设置环境变量，使得私有仓库中的镜像可以被删除。
# registry:2 image:tag
docker run -d -p 5000:5000 --mount type=volume,src=registry-vol,dst=/var/lib/registry --restart=always -e REGISTRY_STORAGE_DELETE_ENABLED=true registry:2
```
这里额外提供一个案例：
```sh
#创建容器
# --mount type=bind,src=/mnt/registry,dst=/var/lib/registry 将本地的/mnt/registry映射到容器的/var/lib/registry目录（通过bind mount的方式进行数据持久化）
# -v `pwd`/config.yml:/etc/docker/registry/config.yml 将本地当前目录的config.yml文件映射到容器的/etc/docker/registry/config.yml。
docker run -d -p 5000:5000 --mount type=bind,src=/mnt/registry,dst=/var/lib/registry --restart=always -v `pwd`/config.yml:/etc/docker/registry/config.yml registry:2

```
需要注意的是：
1. 第一个案例中的`-e REGISTRY_STORAGE_DELETE_ENABLED=true`，在第二个案例中可以通过在配置文件中添加`registry:storage:delete:enabled:true`来替代。
2. 虽然-v的功能和--mount相似，但推荐使用较为冗长的--mount，因为它简单易懂。

至此，私有仓库已经创建成功，我们可以通过docker提供的api确认一下：
```sh
#访问私有仓库的接口
curl http://localhost:5000/v2
#输出结果为{}，说明私有仓库已经在正常运行
```

## 三. 推送镜像到私有仓库
```sh
#将本地镜像hello:latest（该镜像来自上个笔记）添加一个标签，其中localhost:5000对应私有仓库地址
docker tag hello localhost:5000/hello
#docker根据tag，将镜像提交到私有仓库localhost:5000
docker push localhost:5000/hello

#通过docker的http api查看存储的repository列表
curl http://localhost:5000/v2/_catalog
#输出结果为：{"repositories":["hello"]}，说明已经提交成功。现在即使删除本地的镜像，也可以通过`docker pull localhost:5000/hello`从私有仓库重新获取。
```

> repository指的是一组Docker镜像。 repository可以通过推送到仓库服务来分享。同一个repository中的不同镜像可以通过标签来归类。

hello镜像推送成功后，私有仓库会生成一个名为hello的repository。这个repository会存储各个tag的hello镜像，例如：hello:v1、hello:v2。

## 四. 搭建私有仓库的管理后台
对于私有仓库，docker只提供了http api的接口文档，它并未提供官方的管理后台。为了方便学习，采用第三方提供的[Joxit/docker-registry-ui](https://github.com/Joxit/docker-registry-ui)。
```sh
#拉取镜像
docker pull joxit/docker-registry-ui:static
#创建容器
# -p 5050:80 将本地的5050端口映射到容器的80端口
# -e REGISTRY_URL=http://172.17.0.1:5000 通过环境变量设置容器访问的仓库地址为 http://172.17.0.1:5000
# -e DELETE_IMAGES=true 通过环境变量设置镜像可删除
# -e REGISTRY_TITLE="My registry" 通过环境变量设置后台的title
docker run -d -p 5050:80 -e REGISTRY_URL=http://172.17.0.1:5000 -e DELETE_IMAGES=true -e REGISTRY_TITLE="My registry" joxit/docker-registry-ui:static
```
这里的REGISTRY_UR并不是 http://localhost:5000 ，因为容器和本机的localhost并不等价。通过以下方式取得在对应的docker网络中本机的局域网地址：
```sh
#后台容器采用默认的bridge网络， 查询该网络的详细属性
docker network inspect bridge
#输出结果中发现IPAM.Config.Gateway为172.17.0.1，这就是本机在bridge网络中的局域网地址了。
```
通过浏览器访问 http://localhost:5050 ，可以通过管理后台对私有仓库进行管理了。

## 五. 删除私有仓库中的hello

通过后台页面，找到删除功能并不复杂。但是即使删除成功，后台的repository列表中依然存在hello（虽然再也无法拉取镜像）。这并不是管理后台的问题，下面通过接口确认：
```sh
#通过docker的http api查看存储的repository列表
curl http://localhost:5000/v2/_catalog
#输出结果为：{"repositories":["hello"]}，hello的repository依然存在
```
查阅资料，发现官方指出：
1. 目前的api只能删除repository中的镜像，而不能删除repository本身。
2. 私有仓库提供的api只能删除manifests（清单）和layers（层）。所以repository一旦被创建，将无法通过api将其彻底删除。
3. api中的删除操作会移除对目标的引用，使得其可以被垃圾回收。同时让它无法被api访问。
4. 垃圾回收会清理不被任何manifests（清单）引用的数据块，数据块包含layers（层）或manifests（清单）。当manifests（清单）被删除时，它指向的layers（层）中没被其他清单引用的也会被删除。

也就是说之前通过api删除的，只是repository下的hello:latest镜像，而repository本身依然存在。
想要删除repository，需要通过一种stop-the-world（清理期间上传中的镜像可能会被误删）的方式：

```sh
#查找私有仓库的容器ID
docker container ls | grep registry:2
#容器执行指令
# exec 让容器执行指令
# -it i为交互，t为分配一个冒充的tty
# 74252955fbd9 私有仓库的容器ID，替换为实际ID
# /bin/sh 让容器执行的指令
docker exec -it 74252955fbd9 /bin/sh #执行后可以直接通过shell操作容器了
#进行垃圾回收
# 垃圾回收会清理不被任何manifests（清单）引用的数据块，数据块包含layers（层）或manifests（清单）
# 当manifests（清单）被删除时，它指向的layers（层）中没被其他清单引用的也会被删除
/bin/registry garbage-collect /etc/docker/registry/config.yml
#删除hello的repository
rm -rf /var/lib/registry/docker/registry/v2/repositories/hello

#通过docker的http api查看存储的repository列表
curl http://localhost:5000/v2/_catalog
#输出结果为：{"repositories":[]}，说明已经hello已经被彻底删除
```
