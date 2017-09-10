---
title: Gerrit 集锦
date: 2017-08-11 13:50:04
categories: Gerrit
tags: [Gerrit, Git]
---

## Tag

### Tag 的创建和删除

``` bash
# Create a tag 'v2.1' and push it to remote server
$ git tag -m'release v2.1' v2.1
$ git push origin v2.1

# Remove the created tag 'v2.1'
$ git tag -d v2.1
$ git push origin :v2.1
```