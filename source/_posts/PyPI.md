---
title: PyPI
date: 2017-07-10 16:22:12
categories: Python
tags: [Python, PyPI, pip2pi]
---

## 使用 pip2pi 构建本地镜像

pip2pi 一款构建 PyPi 仓库的工具，项目网址：https://github.com/wolever/pip2pi 。

#### 安装
``` bash
$ pip install pip2pi
```

#### 下载所需要的包
``` bash
$ pip2tgz /var/www/pypi/ -r requirements.txt
$ pip2tgz /var/www/pypi/ pyramid==1.2
```

#### 构建索引
``` bash
$ dir2pi /var/www/pypi/
```

#### 使用
``` bash
$ pip3 install -i http://myserver/pypi/simple --trusted-host myserver --upgrade pip
```