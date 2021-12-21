---
title: "在银河麒麟高级服务器操作系统V10上安装docker"
categories: [ "docker" ]
tags: [ "docker", "kylinos" ]
draft: false
slug: "install-docker-on-kylinos"
date: "2021-12-21T23:39:00+08:00"
---

> 银河麒麟高级服务器操作系统 V10 是针对企业级关键业务，适应虚拟化、 云计算、大数据、工业互联网时代对主机系统可靠性、安全性、性能、扩展性和 实时性的需求，依据 CMMI 5 级标准研制的提供内生安全、云原生支持、国产 平台深入优化、高性能、易管理的新一代自主服务器操作系统；同源支持飞腾、 龙芯、申威、兆芯、海光、鲲鹏等自主平台；可支撑构建大型数据中心服务器高 可用集群、负载均衡集群、分布式集群文件系统、虚拟化应用和容器云平台等， 可部署在物理服务器和虚拟化环境、私有云、公有云和混合云环境；应用于政府、 国防、金融、教育、财税、公安、审计、交通、医疗、制造等领域。

公司有个项目需要将系统部署在 **kylinos**上，刚开始还有点头疼，害怕各种程序无法安装和使用，等安装好服务器进行使用的时候发现这不就是基于centos的嘛，虽然基于哪个版本不知道，但是可以测试的，于是我一顿操作，最后发现它是基于Centos8的，系统内核版本是 4.19，问题不大，既然是基于Centos8的，那Centos8上能跑的程序，在这肯定也能跑，然后我就开始了愉快（痛苦）的安装docker之旅了。

### 配置阿里云Centos8镜像源

之所以要配置 Centos8 的镜像源是因为在安装docker的时候需要额外的一些依赖，而这些依赖在麒麟官方的源里面是没有的。

```bash
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-8.repo
```

### 配置阿里云 docker 镜像源

```bash
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
```

### 定义 yum 变量&修改 repo

修改 centos 和 docker `repo`文件中的 `$releasever` 为 `centos_version` ，原因是在麒麟服务器操作系统V10中 `$releasever`被修改为了 10，而我们需要使用 centos 8的镜像源，如果你不替换，基本上仓库的每一个地址都是404。

```bash
echo "8" > /etc/yum/vars/centos_version
sed -i 's/$releasever/$centos_version/g' /etc/yum.repos.d/docker-ce.repo
sed -i 's/$releasever/$centos_version/g' /etc/yum.repos.d/CentOS-Base.repo
```

### 建立yum缓存

没啥可说的

```bash
yum makecache
```

### 查看docker-ce 版本

```bash
yum list docker-ce --showduplicates | sort -r

docker-ce.x86_64               3:20.10.9-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.8-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.7-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.6-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.5-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.4-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.3-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.2-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.1-3.el8                 docker-ce-stable
docker-ce.x86_64               3:20.10.12-3.el8                docker-ce-stable
docker-ce.x86_64               3:20.10.11-3.el8                docker-ce-stable
docker-ce.x86_64               3:20.10.10-3.el8                docker-ce-stable
docker-ce.x86_64               3:20.10.0-3.el8                 docker-ce-stable
docker-ce.x86_64               3:19.03.15-3.el8                docker-ce-stable
docker-ce.x86_64               3:19.03.15-3.el8                @docker-ce-stable
docker-ce.x86_64               3:19.03.14-3.el8                docker-ce-stable
docker-ce.x86_64               3:19.03.13-3.el8                docker-ce-stable
```

### 安装docker

这里要安装 docker-ce 19.03 版本，因为我在使用最新版 20.10 启动容器时出现了未知的权限问题，而麒麟服务器操作系统资料相对较少，我未能找到相应的解决方案，只好退而求其次，换到上一个稳定版本。

20.10 版本错误信息如下：
```bash
docker: Error response from daemon: OCI runtime create failed: container_linux.go:318: starting container process caused "permission denied": unknown.
ERRO[0000] error waiting for container: context canceled
```

还是安装 19.03 版本吧。

```bash
yum install docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io -y
```

### 启动docker

```
systemctl start docker
systemctl enable docker
```

### 启动 hello-world 进行测试

```bash
root@localhost ~]# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

完美使用 -:)