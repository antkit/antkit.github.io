---
title: Hello Hexo
date: 2015-03-09 15:54:58
categories: Hexo
tags: [Hexo]
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Install node.js and npm

``` bash
$ cd ~/tools
$ wget https://nodejs.org/dist/v6.11.1/node-v6.11.1-linux-x64.tar.xz
$ tar xzvf node-v6.11.1-linux-x64.tar.xz
$ echo "export PATH=$HOME/tools/node-v6.11.1-linux-x64/bin:$PATH" >> ~/.bashrc
$ source ~/.bashrc
```

### Initialize web site

``` bash
$ git clone https://github.com/antkit/antkit.github.io.git
$ git checkout -b develop origin/develop
$ cd antkit.github.io.git
$ npm install --registry=https://registry.npm.taobao.org
```

Now hexo is under directory ./node_modules/hexo/bin. You can run server:

``` bash
./node_modules/hexo/bin/hexo server
```


### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/deployment.html)
