---
title: Gerrit 集锦
date: 2017-08-11 13:50:04
categories: Gerrit
tags: [Gerrit, Git]
---

## Tag
创建和删除 tag：
``` bash
# create and push a tag
$ git tag -m'release v2.1' v2.1
$ git push origin v2.1
# remove the created tag
$ git pull
$ git tag -d v2.1
$ git push origin :v2.1
```