---
title: "docker"
date: 2020-01-17T09:47:11+01:00
draft: false
---



## Dockerfile 创建镜像
`docker build -t carlos/centos .`

## 创建容器 
`docker run --name xgp -p 15426:5000 -p 15427:22 -d carlos/centos /bin/bash`

## 容器commit镜像
`docker commit container_name image_name:version`

## apline

### apline修改时区

```
RUN echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.8/main/" > /etc/apk/repositories \
        && echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.8/community/" >> /etc/apk/repositories \
        && apk update && apk add tzdata \
        && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \ 
        && echo "Asia/Shanghai" > /etc/timezone

```

## docker-copyedit.py

用法介绍: `python docker-copyedit.py FROM SXKJ:32775/t99app:1.5  INTO SXKJ:32775/t99app:1.6  REMOVE ALL volumes and remove all ports`

详细用法:

```shell
./docker-copyedit.py  FROM image1 INTO image2 remove all volumes
 
./docker-copyedit.py into image2 from image1 set user null
 
./docker-copyedit.py set user null and set cmd null from image1 into image2
 
./docker-copyedit.py FROM image1 INTO image2 set user null + set cmd null + rm all volumes
     
./docker-copyedit.py FROM image1 INTO image2 -vv add volume /var/tmp

./docker-copyedit.py FROM image1 INTO image2 -vv REMOVE VOLUME /var/atlassian/jira-data

./docker-copyedit.py FROM image1 INTO image2 -vv REMOVE VOLUMES '/var/*' AND RM PORTS 80%0

./docker-copyedit.py FROM image1 INTO image2 -vv set entrypoint null and set cmd /entrypoint.sh

./docker-copyedit.py FROM image1 INTO image2 -vv set shell cmd "/entrypoint.sh foo"

./docker-copyedit.py FROM image1 INTO image2 -vv set label author "real me" and rm labels old%

./docker-copyedit.py FROM image1 INTO image2 -vv set env MAINDIR "/path" and rm env backupdir

./docker-copyedit.py FROM image1 INTO image2 -vv REMOVE PORT 4444

./docker-copyedit.py FROM image1 INTO image2 -vv remove port ldap and rm port ldaps

./docker-copyedit.py FROM image1 INTO image2 -vv remove all ports

./docker-copyedit.py FROM image1 INTO image2 -vv add port ldap and add port ldaps
```

源文件:

​	`~/Desktop/code/usefultools/docker-copyedit.py`

## 直接load失败的时候

`cat xxx.tar|docker import - {imagename:version}`

## 删除所有<none>镜像

` docker rmi -f $(docker images -f "dangling=true" -q)`

## ubuntu升级docker版本至最新

`curl -fsSL https://get.docker.com/ | sh`

## /run/docker目录满

当系统默认分配的`/run`目录大小无法满足不断扩增的`/run/docker`时, 可以选择手动扩充`/run`目录的大小, `/run`目录实质是一个临时文件系统, 存在在内存中.

**临时修改(不需要重启)**
`mount -o remount,size=4g tmpfs /run`

**永久修改**

```shell
vim  /etc/fstab 

把tmpfs这一行改为：

tmpfs                   /run               tmpfs   defaults,size=4g     0 0
```

## 迁移/var/lib/docker

```shell
# 软连接
mv /var/lib/docker /root/data/docker
ln -s /root/data/docker /var/lib/docker
```



## ubuntu 安装docker

`wget -qO- https://get.docker.com/ | sh`

## docker exec 

https://docs.docker.com/engine/reference/commandline/exec/

| Name, shorthand      | Default | Description                                                  |
| -------------------- | ------- | ------------------------------------------------------------ |
| `--detach , -d`      |         | Detached mode: run command in the background                 |
| `--detach-keys`      |         | Override the key sequence for detaching a container          |
| `--env , -e`         |         | [**API 1.25+**](https://docs.docker.com/engine/api/v1.25/) Set environment variables |
| `--interactive , -i` |         | Keep STDIN open even if not attached                         |
| `--privileged`       |         | Give extended privileges to the command                      |
| `--tty , -t`         |         | Allocate a pseudo-TTY                                        |
| `--user , -u`        |         | Username or UID (format: <name\|uid>[:<group\|gid>])         |
| `--workdir , -w`     |         | [**API 1.35+**](https://docs.docker.com/engine/api/v1.35/) Working directory inside the container |

## docker attach

*https://docs.docker.com/engine/reference/commandline/attach/*

*Attach local standard input, output, and error streams to a running container*

> **Note:** A process running as PID 1 inside a container is treated specially by Linux: it ignores any signal with the default action. So, the process will not terminate on `SIGINT` or `SIGTERM` unless it is coded to do so.

It is forbidden to redirect the standard input of a `docker attach` command while attaching to a tty-enabled container (i.e.: launched with `-t`).

## 删除docker全家桶

**centos:**

```shell
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

