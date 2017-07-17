---
title: apt-mirror
date: 2017-07-10 15:54:58
categories: Ubuntu
tags: [Ubuntu, apt mirror]
---

## 安装 apt-mirror

安装很简单，PPA自带：
``` bash
$ sudo apt-get install apt-mirror
```

## 配置 apt-mirror

apt-mirror 的配置文件位于 /etc/apt/mirror.list，我们在使用前做一下配置：
``` bash
$ sudo vim /etc/apt/mirror.list
```

``` bash
############# config ##################
#
set base_path    /var/wwwroot/apt-mirror
set run_postmirror 0
set nthreads     20
set _tilde 0
#
############# end config ##############

deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
deb https://download.docker.com/linux/ubuntu xenial stable

clean http://mirrors.aliyun.com/ubuntu
clean https://download.docker.com/linux/ubuntu
```

关于这个配置文件：
  * base_path 告诉 apt-mirror 下载的数据存在哪里。
  * run_postmirror 置为 0 告诉 apt-mirror 不要运行 post script，虽然这个不设置也不影响正常使用，但在 apt-mirror 运行结束时会报一个讨厌的错误信息。
  * deb 这些行告诉 apt-mirror 从哪里获取 package，这里我们选择了 aliyun 的镜像服务。另外，我们也把 docker 官方源给加了进去。
  * deb-src 也是被支持的，我们这里没有使用，如果你需要的话请自行加上去。
  * deb [arch=amd64]，可以这样使用来告诉 apt-mirror 取 64 位版本，否则 apt-mirror 以默认方式取与本机架构一致的版本。
  * 这个是 Ubuntu 16.04（即 xenial）版本，其他版本请自行调整。
  * clean 结束时做一些清理工作。

## 运行 apt-mirror

``` bash
$ sudo apt-mirror
```

运行结束后，可以查看一下结果：
``` bash
$ tree -L 4 /var/wwwroot/apt-mirror
/var/wwwroot/apt-mirror/
├── mirror
│   ├── download.docker.com
│   │   └── linux
│   │       └── ubuntu
│   ├── mirrors.aliyun.com
│   │   └── ubuntu
│   │       ├── dists
│   │       └── pool
├── skel
│   ├── download.docker.com
│   │   └── linux
│   │       └── ubuntu
│   ├── mirrors.aliyun.com
│   │   ├── robots.txt
│   │   └── ubuntu
│   │       └── dists
└── var
    ├── ALL
    ├── archive-log.0
    ├── archive-log.1
```

## 配置 HTTP Server

配置 HTTP Server，使得可以通过 HTTP 访问刚获取到的 mirror 仓库。以 Apache 为例：
``` xml
<VirtualHost *>
    ...

    <Location /apt-mirror>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Location>
</VirtualHost>
```

## 应用 mirror

``` bash
$ sudo vim /etc/apt/sources.list
```

``` bash
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial main restricted
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial universe
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial multiverse
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-security universe
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/mirrors.aliyun.com/ubuntu/ xenial-security multiverse
deb [arch=amd64] http://192.168.0.11/apt-mirror/mirror/download.docker.com/linux/ubuntu/ xenial stable
```

配置文件里面的 192.168.0.11 是内部 mirror 服务器的 IP 地址。

[arch=amd64] 是必须的，否则服务器会报 404 错误。32 位版本请做调整。

## 其他

请定时运行 apt-mirror 程序以更新本地仓库。