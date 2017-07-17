---
title: Docker
date: 2017-07-10 16:13:19
categories: Docker
tags: Docker
---

## 退出容器
退出容器有两种方式：
  * ctrl+d 退出容器且关闭, docker ps 查看无
  * ctrl+p+q 退出容器但不关闭, docker ps 查看有

## 关于 GUI
参考 stackoverflow 上的 [Can you run GUI apps in a docker container?](https://stackoverflow.com/questions/16296753/can-you-run-gui-apps-in-a-docker-container)。
``` bash
# Dockerfile partial

# Access xserver on the host
RUN apt-get install -qqy x11-apps
ENV DISPLAY :0
```
``` bash
$ XSOCK=/tmp/.X11-unix
$ XAUTH=/tmp/.docker.xauth
$ xauth nlist :0 | sed -e 's/^..../ffff/' | xauth -f $XAUTH nmerge -
$ docker run --rm -v $XSOCK:$XSOCK -v $XAUTH:$XAUTH -e XAUTHORITY=$XAUTH -t -i mytest xeyes
```

## Docker 加速
在阿里云申请加速服务，记下加速器 URL。新建 /etc/docker/daemon.json 并加入如下内容（里面的 URL 就是你加速器的 URL）：
``` json
{
  "registry-mirrors": ["https://1inmm0s4.mirror.aliyuncs.com"]
}
```
上面的加速器如果不能用可以自己在阿里云申请一个，保存后重启 docker 服务以加载配置：
``` bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

## Container自启动
参考 [How do I auto start docker containers at system boot](https://serverfault.com/questions/633067/how-do-i-auto-start-docker-containers-at-system-boot)。
``` bash
$ docker run --restart=always -d redis
```

## Docker Compose

### nvidia-docker
参考 [Use nvidia-docker-compose](https://stackoverflow.com/questions/41346401/use-nvidia-docker-compose-launch-a-container-but-exited-soon)。
如果你运行过 nvidia-docker，该参考中的第二步就不需要了，nvidia-docker 会自动创建 volume nvidia_driver_<version>。
``` yaml
version: '3'

volumes:
  nvidia_driver_375.66:  # 使用 docker volume ls 查看，与之匹配
    external: true  # this will use the volume we created above

services:
  cuda:
    command: nvidia-smi
    devices:  # this is required
      - /dev/nvidiactl
      - /dev/nvidia-uvm
      - /dev/nvidia0  # in general: /dev/nvidia# where # depends on which gpu card is wanted to be used
    image: nvidia/cuda:8.0-runtime-ubuntu16.04
    volumes:
      - nvidia_driver_375.66:/usr/local/nvidia/:ro
```

### Variables & Template

有两种简单的方式实现变量功能。

** Environment file **

Docker compose 是支持 environment file 的，详细可参考官方文档 [Declare default environment variables in file](https://docs.docker.com/compose/env-file/)。

如果没有特别指定，默认的 environment file 就是与 compose file 同一目录的 .env 文件。Environment file 的内容是一些键值对，compose 将他们应用到 compose file 所声明的变量。这种方式有不足：无法作为 compose file 内容的 key，如果有这种需求，请看接下来的另一种方式。

** envsubst **

参考 [How to use environment variables in docker compose](https://stackoverflow.com/questions/29377853/how-to-use-environment-variables-in-docker-compose)。